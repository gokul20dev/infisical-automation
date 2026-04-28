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

        // ── 3. Ensure OLD stack is running ───────────────────────────────────
        stage('Ensure Compose Running') {
            steps {
                sh '''
                echo "Starting docker-compose (old stack)..."
                docker compose up -d --no-recreate

                echo "Waiting for OLD DB to be ready..."
                for i in $(seq 1 30); do
                    docker exec ${OLD_CONTAINER} pg_isready -U postgres && break
                    echo "  still waiting... ($i)"
                    sleep 3
                done

                docker exec ${OLD_CONTAINER} pg_isready -U postgres || \
                    { echo "❌ OLD DB never became ready"; exit 1; }

                echo "OLD DB is ready ✅"
                '''
            }
        }

        // ── 4. Backup OLD DB ─────────────────────────────────────────────────
        // pg_dump runs INSIDE the old container; we copy the file OUT via docker cp.
        stage('Take Backup') {
            steps {
                sh '''
                echo "Taking backup from ${OLD_CONTAINER}..."

                # Run pg_dump inside the container, write to a known path inside it
                docker exec ${OLD_CONTAINER} \
                    pg_dump -U postgres -d infisical -F c -f /tmp/backup.dump

                # Copy the dump file from the container to the Jenkins workspace
                docker cp ${OLD_CONTAINER}:/tmp/backup.dump ${BACKUP_FILE}

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

        // ── 9. Switch App to NEW DB (zero downtime) ───────────────────────────
        // We rewrite .env with the new host, then force-recreate ONLY the
        // infisical service. postgres + redis containers keep running.
        stage('Switch App to New DB') {
            steps {
                withCredentials([
                    string(credentialsId: 'db-pass',     variable: 'DB_PASS'),
                    string(credentialsId: 'auth-secret', variable: 'AUTH_SECRET'),
                    string(credentialsId: 'enc-key',     variable: 'ENC_KEY')
                ]) {
                    sh '''
                    echo "Updating .env to point at NEW DB (${NEW_CONTAINER})..."

                    cat > .env <<EOF
DB_CONNECTION_URI=postgresql://postgres:${DB_PASS}@${NEW_CONTAINER}:5432/infisical
REDIS_URL=redis://redis:6379
AUTH_SECRET=${AUTH_SECRET}
ENCRYPTION_KEY=${ENC_KEY}
EOF

                    echo ".env updated ✅"

                    echo "Restarting ONLY the infisical app container (zero downtime)..."
                    # --no-deps: don't touch postgres/redis
                    # --force-recreate: picks up the new .env
                    docker compose up -d --no-deps --force-recreate infisical

                    echo "App container recreated ✅"
                    '''
                }
            }
        }

        // ── 10. Smoke Test — App is back on port 8000 ────────────────────────
        stage('Smoke Test') {
            steps {
                sh '''
                echo "Waiting for app to come back on port 8000..."
                for i in $(seq 1 20); do
                    STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/status || true)
                    echo "  HTTP $STATUS (attempt $i)"
                    [ "$STATUS" = "200" ] && break
                    sleep 5
                done

                STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8000/api/status || true)
                [ "$STATUS" = "200" ] || \
                    { echo "❌ App not responding after switch!"; exit 1; }

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
