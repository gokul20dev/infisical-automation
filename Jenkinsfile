pipeline {
    agent any

    environment {
        DB_NAME = "infisical"
        DB_USER = "postgres"
        OLD_DB = "infisical-postgres"
        NEW_DB = "infisical-postgres-new"
        BACKUP_FILE = "backup.dump"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Create .env securely') {
            steps {
                withCredentials([
                    string(credentialsId: 'db-pass', variable: 'DB_PASS'),
                    string(credentialsId: 'auth-secret', variable: 'AUTH_SECRET'),
                    string(credentialsId: 'enc-key', variable: 'ENC_KEY')
                ]) {
                    sh '''
                    echo "Creating .env securely..."

                    cat > .env <<EOF
DB_CONNECTION_URI=postgresql://postgres:${DB_PASS}@postgres:5432/infisical
REDIS_URL=redis://redis:6379
AUTH_SECRET=${AUTH_SECRET}
ENCRYPTION_KEY=${ENC_KEY}
EOF
                    '''
                }
            }
        }

        stage('Clean Previous Run') {
    steps {
        sh '''
        echo "Cleaning ALL old containers..."

        # Stop compose services
        docker compose down || true

        # Remove ALL conflicting containers
        docker rm -f infisical-postgres-new || true
        docker rm -f infisical-redis || true
        docker rm -f infisical-postgres || true
        docker rm -f infisical || true
        '''
    }
}

        stage('Start Compose') {
            steps {
                sh '''
                echo "Starting docker-compose..."

                docker compose up -d

                echo "Waiting for OLD DB..."

                until docker exec infisical-postgres pg_isready -U postgres
                do
                  echo "Waiting for OLD DB..."
                  sleep 2
                done

                echo "OLD DB is ready ✅"
                '''
            }
        }

        stage('Take Backup') {
            steps {
                sh '''
                echo "Taking backup..."

                docker exec infisical-postgres \
                pg_dump -U postgres -d infisical -F c > backup.dump

                ls -lh backup.dump
                '''
            }
        }

        stage('Start New DB') {
            steps {
                withCredentials([string(credentialsId: 'db-pass', variable: 'DB_PASS')]) {
                    sh '''
                    echo "Starting NEW DB..."

                    docker run -d \
                    --name infisical-postgres-new \
                    --network $(docker network ls --format '{{.Name}}' | grep infisical | head -1) \
                    -e POSTGRES_PASSWORD=${DB_PASS} \
                    postgres:15
                    '''
                }
            }
        }

        stage('Prepare New DB') {
            steps {
                sh '''
                echo "Waiting for NEW DB..."

                until docker exec infisical-postgres-new pg_isready -U postgres
                do
                  sleep 2
                done

                docker exec infisical-postgres-new \
                psql -U postgres -c "CREATE DATABASE infisical;"
                '''
            }
        }

        stage('Restore Backup') {
    steps {
        sh '''
        echo "Restoring backup properly..."

        # Copy backup into container
        docker cp backup.dump infisical-postgres-new:/backup.dump

        # Restore inside container
        docker exec infisical-postgres-new \
        pg_restore -U postgres -d infisical /backup.dump
        '''
    }
}

        stage('Verify Restore') {
            steps {
                sh '''
                echo "Verifying tables..."

                docker exec infisical-postgres-new \
                psql -U postgres -d infisical -c "\\dt"
                '''
            }
        }

        stage('Switch App to New DB') {
            steps {
                withCredentials([
                    string(credentialsId: 'db-pass', variable: 'DB_PASS'),
                    string(credentialsId: 'auth-secret', variable: 'AUTH_SECRET'),
                    string(credentialsId: 'enc-key', variable: 'ENC_KEY')
                ]) {
                    sh '''
                    echo "Switching application to NEW DB..."

                    docker compose down

                    DB_CONNECTION_URI=postgresql://postgres:${DB_PASS}@infisical-postgres-new:5432/infisical \
                    REDIS_URL=redis://redis:6379 \
                    AUTH_SECRET=${AUTH_SECRET} \
                    ENCRYPTION_KEY=${ENC_KEY} \
                    docker compose up -d
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Secure deployment successful"
        }
        failure {
            echo "❌ Deployment failed - old system untouched"
        }
    }
}
