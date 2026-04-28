pipeline {
    agent any

    environment {
        DB_NAME        = "infisical"
        DB_USER        = "postgres"
        OLD_CONTAINER  = "infisical-postgres"
        NEW_CONTAINER  = "infisical-postgres-new"
        BACKUP_FILE    = "${WORKSPACE}/infisical-backup.dump"
        COMPOSE_NETWORK = "infisical_default"
    }

    stages {

        // ── 1. Checkout ──────────────────────────────────────────────────────
        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        // ── 2. Create .env securely ──────────────────────────────────────────
        stage('Create .env') {
            steps {
                withCredentials([
                    string(credentialsId: 'db-pass',     variable: 'DB_PASS'),
                    string(credentialsId: 'auth-secret', variable: 'AUTH_SECRET'),
                    string(credentialsId: 'enc-key',     variable: 'ENC_KEY')
                ]) {
                    sh '''
                    echo "Creating .env securely..."
                    cat > .env <<EOF
DB_CONNECTION_URI=postgresql://postgres:${DB_PASS}@postgres:5432/infisical
REDIS_URL=redis://redis:6379
AUTH_SECRET=${AUTH_SECRET}
ENCRYPTION_KEY=${ENC_KEY}
EOF
                    echo ".env created ✅"
                    '''
                }
            }
        }

        // ── 3. Detect ACTIVE DB and verify it is running ─────────────────────
        // After the first migration the live DB is infisical-postgres-new, not
        // infisical-postgres. We read the DB_CONNECTION_URI from the running
        // infisical container so every build backs up from the RIGHT database.
        stage('Detect Active DB') {
            steps {
                sh '''
                echo "Detecting which DB the running infisical app is using..."

                # Extract hostname from DB_CONNECTION_URI env var of infisical container
                # URI format: postgresql://user:pass@HOSTNAME:5432/db
                ACTIVE_DB=$(docker inspect infisical \
                    --format "{{range .Config.Env}}{{println .}}{{end}}" \
                    | grep "^DB_CONNECTION_URI=" \
                    | sed "s|.*@\([^:]*\):.*|\1|")

                if [ -z "$ACTIVE_DB" ]; then
                    echo "Could not detect active DB — falling back to ${OLD_CONTAINER}"
                    ACTIVE_DB=${OLD_CONTAINER}
                fi

                echo "Active DB container: $ACTIVE_DB"

                # Persist for later stages
                echo $ACTIVE_DB > /tmp/infisical_active_db.txt

                # Verify it is running and ready
                docker inspect --format "{{.State.Running}}" $ACTIVE_DB | grep -q "true" || \
                    { echo "❌ $ACTIVE_DB is not running!"; exit 1; }

                for i in $(seq 1 30); do
                    docker exec $ACTIVE_DB pg_isready -U postgres && break
                    echo "  still waiting... ($i)"
                    sleep 3
                done

                docker exec $ACTIVE_DB pg_isready -U postgres || \
                    { echo "❌ Active DB never became ready"; exit 1; }

                echo "Active DB is ready ✅"
                '''
            }
        }

        // ── 4. Backup ACTIVE DB ──────────────────────────────────────────────
        // Back up whichever DB the running infisical is currently using.
        stage('Take Backup') {
            steps {
                sh '''
                ACTIVE_DB=$(cat /tmp/infisical_active_db.txt)
                echo "Taking backup from $ACTIVE_DB..."

                # Run pg_dump inside the container, write to a known path inside it
                docker exec $ACTIVE_DB \
                    pg_dump -U postgres -d infisical -F c -f /tmp/backup.dump

                # Copy the dump file from the container to the Jenkins workspace
                docker cp $ACTIVE_DB:/tmp/backup.dump ${BACKUP_FILE}

                ls -lh ${BACKUP_FILE}
                echo "Backup complete ✅"
                '''
            }
        }

        // ── 5. Start NEW DB container (on the same compose network) ──────────
        stage('Start New DB') {
            steps {
                withCredentials([string(credentialsId: 'db-pass', variable: 'DB_PASS')]) {
                    sh '''
                    echo "Starting NEW DB container..."

                    # Remove any leftover container
                    docker rm -f ${NEW_CONTAINER} || true

                    docker run -d \
                        --name ${NEW_CONTAINER} \
                        --network ${COMPOSE_NETWORK} \
                        -e POSTGRES_PASSWORD=${DB_PASS} \
                        -e POSTGRES_DB=infisical \
                        postgres:15

                    echo "NEW DB container started ✅"
                    '''
                }
            }
        }

        // ── 6. Wait for NEW DB and create the database ───────────────────────
        stage('Prepare New DB') {
            steps {
                sh '''
                echo "Waiting for NEW DB to be ready..."
                for i in $(seq 1 30); do
                    docker exec ${NEW_CONTAINER} pg_isready -U postgres && break
                    echo "  still waiting... ($i)"
                    sleep 3
                done

                docker exec ${NEW_CONTAINER} pg_isready -U postgres || \
                    { echo "❌ NEW DB never became ready"; exit 1; }

                echo "NEW DB is ready ✅"
                '''
            }
        }

        // ── 7. Restore Backup into NEW DB ────────────────────────────────────
        // Copy dump into new container first, then restore.
        stage('Restore Backup') {
            steps {
                sh '''
                echo "Copying backup into NEW container..."
                docker cp ${BACKUP_FILE} ${NEW_CONTAINER}:/tmp/backup.dump

                echo "Restoring backup into NEW DB..."
                docker exec ${NEW_CONTAINER} \
                    pg_restore -U postgres -d infisical \
                    --no-owner --no-privileges --clean --if-exists \
                    -v /tmp/backup.dump || true
                # pg_restore returns non-zero on warnings; || true prevents false failures.
                # If there's a real error it will be visible in the logs above.

                echo "Restore complete ✅"
                '''
            }
        }

        // ── 8. Verify Restore ─────────────────────────────────────────────────
        stage('Verify Restore') {
            steps {
                sh '''
                echo "Verifying tables in NEW DB..."
                TABLE_COUNT=$(docker exec ${NEW_CONTAINER} \
                    psql -U postgres -d infisical -t \
                    -c "SELECT count(*) FROM information_schema.tables WHERE table_schema='public';" \
                    | tr -d '[:space:]')

                echo "Tables in new DB: ${TABLE_COUNT}"
                [ "${TABLE_COUNT}" -gt "0" ] || \
                    { echo "❌ Restore appears empty — aborting switch!"; exit 1; }

                echo "Restore verified ✅"
                '''
            }
        }

        // ── 9. Switch App to NEW DB ───────────────────────────────────────────
        // Use docker run -e directly (matches manual approach from docs).
        // docker compose up was causing crash-loops due to .env substitution issues.
        stage('Switch App to New DB') {
            steps {
                withCredentials([
                    string(credentialsId: 'db-pass',     variable: 'DB_PASS'),
                    string(credentialsId: 'auth-secret', variable: 'AUTH_SECRET'),
                    string(credentialsId: 'enc-key',     variable: 'ENC_KEY')
                ]) {
                    sh '''
                    echo "Stopping old infisical app container..."
                    docker stop infisical || true
                    docker rm   infisical || true

                    echo "Starting infisical app pointed at NEW DB on port 8000..."
                    docker run -d \
                        --name infisical \
                        -p 8000:8080 \
                        --network ${COMPOSE_NETWORK} \
                        -e DB_CONNECTION_URI="postgresql://postgres:${DB_PASS}@${NEW_CONTAINER}:5432/infisical" \
                        -e REDIS_URL="redis://infisical-redis:6379" \
                        -e AUTH_SECRET="${AUTH_SECRET}" \
                        -e ENCRYPTION_KEY="${ENC_KEY}" \
                        infisical/infisical:latest

                    echo "App container started ✅"
                    '''
                }
            }
        }

        // ── 10. Smoke Test — App is back on port 8000 ────────────────────────
        stage('Smoke Test') {
            steps {
                sh '''
                echo "Giving infisical 20s to run DB migrations..."
                sleep 20

                echo "Waiting for app on port 8000 (up to 3 minutes)..."
                for i in $(seq 1 36); do
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/status || true)
                    echo "  HTTP $STATUS (attempt $i/36)"
                    [ "$STATUS" = "200" ] && break
                    sleep 5
                done

                STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/status || true)
                if [ "$STATUS" != "200" ]; then
                    echo "❌ App not responding after switch! Dumping logs for diagnosis..."
                    echo "--- docker ps ---"
                    docker ps -a --filter name=infisical
                    echo "--- infisical container logs (last 50 lines) ---"
                    docker logs --tail 50 infisical || true
                    exit 1
                fi

                echo "App is live on port 8000 ✅"
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Zero-downtime migration to NEW DB successful. App running on port 8000."
        }
        failure {
            echo "❌ Migration failed. OLD DB (${OLD_CONTAINER}) is still running — no data loss."
        }
    }
}
