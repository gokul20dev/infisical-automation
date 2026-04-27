pipeline {
    agent any

    environment {
        DB_NAME = "infisical"
        DB_USER = "postgres"
        DB_PASS = "postgres"

        OLD_DB = "infisical-postgres"
        NEW_DB = "infisical-postgres-new"

        BACKUP_FILE = "backup.dump"

        APP_NAME = "infisical-new"
        PORT = "8001"

        NETWORK = "infisical_default"
    }

    stages {

        stage('Take Backup') {
            steps {
                sh '''
                echo "📦 Taking backup..."

                docker exec ${OLD_DB} \
                pg_dump -U ${DB_USER} -d ${DB_NAME} -F c > ${BACKUP_FILE}

                ls -lh ${BACKUP_FILE}
                '''
            }
        }

        stage('Ensure Network Exists') {
            steps {
                sh '''
                echo "🌐 Ensuring network exists..."

                docker network inspect ${NETWORK} >/dev/null 2>&1 || \
                docker network create ${NETWORK}
                '''
            }
        }

        stage('Start New DB') {
            steps {
                sh '''
                echo "🚀 Starting new Postgres..."

                docker rm -f ${NEW_DB} || true

                docker run -d \
                --name ${NEW_DB} \
                --network ${NETWORK} \
                -e POSTGRES_PASSWORD=${DB_PASS} \
                postgres:15
                '''
            }
        }

        stage('Wait for DB') {
            steps {
                sh '''
                echo "⏳ Waiting for DB..."

                until docker exec ${NEW_DB} pg_isready -U ${DB_USER}
                do
                  echo "Waiting for DB..."
                  sleep 2
                done
                '''
            }
        }

        stage('Create DB') {
            steps {
                sh '''
                echo "🛠 Creating database..."

                docker exec ${NEW_DB} \
                psql -U ${DB_USER} -c "CREATE DATABASE ${DB_NAME};"
                '''
            }
        }

        stage('Restore Backup') {
            steps {
                sh '''
                echo "♻️ Restoring backup..."

                docker cp ${BACKUP_FILE} ${NEW_DB}:/backup.dump

                docker exec ${NEW_DB} \
                pg_restore -U ${DB_USER} -d ${DB_NAME} /backup.dump
                '''
            }
        }

        stage('Verify Data') {
            steps {
                sh '''
                echo "🔍 Verifying data..."

                docker exec ${NEW_DB} \
                psql -U ${DB_USER} -d ${DB_NAME} -c "SELECT count(*) FROM users;"
                '''
            }
        }

        stage('Connect Redis to Network') {
            steps {
                sh '''
                docker network connect ${NETWORK} infisical-redis || true
                '''
            }
        }

        stage('Start New Infisical App') {
            steps {
                withCredentials([
                    string(credentialsId: 'auth-secret', variable: 'AUTH_SECRET'),
                    string(credentialsId: 'enc-key', variable: 'ENC_KEY')
                ]) {
                    sh '''
                    echo "🚀 Starting new Infisical app..."

                    docker rm -f ${APP_NAME} || true

                    docker run -d \
                    --name ${APP_NAME} \
                    -p ${PORT}:8080 \
                    --network ${NETWORK} \
                    -e DB_CONNECTION_URI=postgresql://postgres:${DB_PASS}@${NEW_DB}:5432/${DB_NAME} \
                    -e REDIS_URL=redis://infisical-redis:6379 \
                    -e AUTH_SECRET=${AUTH_SECRET} \
                    -e ENCRYPTION_KEY=${ENC_KEY} \
                    infisical/infisical:latest
                    '''
                }
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                echo "🌐 Checking app..."

                for i in {1..10}; do
                  curl -f http://localhost:${PORT} && exit 0
                  echo "Waiting for app..."
                  sleep 5
                done

                exit 1
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Backup + Restore + New App SUCCESS"
        }
        failure {
            echo "❌ Pipeline failed"
        }
    }
}
