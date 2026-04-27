pipeline {
    agent any

    environment {
        DB_CONTAINER_OLD = "infisical-postgres"
        DB_CONTAINER_NEW = "infisical-postgres-new"
        APP_CONTAINER_OLD = "infisical"
        APP_CONTAINER_NEW = "infisical-new"
        NETWORK = "infisical_default"
        BACKUP_FILE = "backup.dump"
    }

    stages {

        stage('Take Backup') {
            steps {
                sh '''
                echo "📦 Taking DB backup..."
                docker exec $DB_CONTAINER_OLD pg_dump -U postgres -d infisical -F c > $BACKUP_FILE
                '''
            }
        }

        stage('Start New DB') {
            steps {
                sh '''
                echo "🚀 Starting new DB..."
                docker rm -f $DB_CONTAINER_NEW || true

                docker run -d \
                  --name $DB_CONTAINER_NEW \
                  --network $NETWORK \
                  -e POSTGRES_USER=postgres \
                  -e POSTGRES_PASSWORD=postgres \
                  -e POSTGRES_DB=infisical \
                  postgres:15
                '''
            }
        }

        stage('Wait for DB') {
            steps {
                sh '''
                echo "⏳ Waiting for DB..."
                sleep 10
                '''
            }
        }

        stage('Restore Backup') {
            steps {
                sh '''
                echo "📥 Restoring backup..."
                cat $BACKUP_FILE | docker exec -i $DB_CONTAINER_NEW pg_restore -U postgres -d infisical
                '''
            }
        }

        stage('Start New App (8001)') {
            steps {
                sh '''
                echo "🚀 Starting new app on port 8001..."
                docker rm -f $APP_CONTAINER_NEW || true

                docker run -d \
                  --name $APP_CONTAINER_NEW \
                  --network $NETWORK \
                  -p 8001:8080 \
                  -e DB_CONNECTION_URI="postgresql://postgres:postgres@$DB_CONTAINER_NEW:5432/infisical" \
                  -e REDIS_URL="redis://redis:6379" \
                  infisical/infisical:latest
                '''
            }
        }

        stage('Health Check') {
            steps {
                sh '''
                echo "❤️ Checking app health..."
                sleep 10

                STATUS=$(curl -s -o /dev/null -w "%{http_code}" http://localhost:8001)

                if [ "$STATUS" != "200" ]; then
                  echo "❌ Health check failed"
                  exit 1
                fi

                echo "✅ App is healthy"
                '''
            }
        }

        stage('Switch Traffic to 8000') {
            steps {
                sh '''
                echo "🔁 Switching traffic..."

                docker stop $APP_CONTAINER_OLD || true
                docker rm $APP_CONTAINER_OLD || true

                docker stop $APP_CONTAINER_NEW
                docker rm $APP_CONTAINER_NEW

                docker run -d \
                  --name $APP_CONTAINER_OLD \
                  --network $NETWORK \
                  -p 8000:8080 \
                  -e DB_CONNECTION_URI="postgresql://postgres:postgres@$DB_CONTAINER_NEW:5432/infisical" \
                  -e REDIS_URL="redis://redis:6379" \
                  infisical/infisical:latest
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Deployment Successful"
        }

        failure {
            echo "❌ Deployment Failed - Rolling back..."

            sh '''
            docker stop infisical-new || true
            docker rm infisical-new || true

            echo "🔙 Rollback complete (old app still running)"
            '''
        }
    }
}
