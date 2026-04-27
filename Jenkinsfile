pipeline {
    agent any

    environment {
        DB_OLD = "infisical-postgres"
        DB_NEW = "infisical-postgres-new"
        APP_OLD = "infisical"
        APP_NEW = "infisical-new"
        REDIS = "infisical-redis"
        NETWORK = "infisical_default"
        BACKUP = "backup.dump"
    }

    stages {

        stage('Take Backup') {
            steps {
                sh '''
                echo "📦 Taking backup..."
                docker exec $DB_OLD pg_dump -U postgres -d infisical -F c > $BACKUP
                '''
            }
        }

        stage('Start New DB') {
            steps {
                sh '''
                echo "🚀 Starting new DB..."
                docker rm -f $DB_NEW || true

                docker run -d \
                  --name $DB_NEW \
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
                sh 'sleep 10'
            }
        }

        stage('Restore Backup') {
            steps {
                sh '''
                echo "📥 Restoring backup..."
                cat $BACKUP | docker exec -i $DB_NEW pg_restore -U postgres -d infisical
                '''
            }
        }

        stage('Start New App (8001)') {
    steps {
        sh '''
        echo "🚀 Starting new app on 8001..."
        docker rm -f infisical-new || true

        docker run -d \
          --name infisical-new \
          --network infisical_default \
          -p 8001:8080 \
          -e AUTH_SECRET="testauth123" \
          -e DB_CONNECTION_URI="postgresql://postgres:postgres@infisical-postgres-new:5432/infisical" \
          -e REDIS_URL="redis://infisical-redis:6379" \
          infisical/infisical:latest
        '''
    }
}

        stage('Health Check') {
            steps {
                sh '''
                echo "❤️ Checking health..."
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

        stage('Switch to 8000') {
            steps {
                sh '''
                echo "🔁 Switching traffic..."

                docker stop $APP_OLD || true
                docker rm $APP_OLD || true

                docker stop $APP_NEW
                docker rm $APP_NEW

                docker run -d \
                  --name $APP_OLD \
                  --network $NETWORK \
                  -p 8000:8080 \
                  -e DB_CONNECTION_URI="postgresql://postgres:postgres@$DB_NEW:5432/infisical" \
                  -e REDIS_URL="redis://$REDIS:6379" \
                  infisical/infisical:latest
                '''
            }
        }
    }

    post {
        failure {
            echo "❌ Deployment failed. Keeping old app running..."
            sh '''
            docker stop infisical-new || true
            docker rm infisical-new || true
            '''
        }

        success {
            echo "✅ Deployment successful"
        }
    }
}
