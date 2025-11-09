pipeline {
    agent any
    
    environment {
        COMPOSE_FILE = 'docker-compose-jenkins.yml'
        PROJECT_DIR = "${WORKSPACE}"
    }
    
    stages {
        stage('Cleanup') {
            steps {
                script {
                    echo 'Cleaning up existing containers...'
                    sh '''
                        docker-compose -f ${COMPOSE_FILE} down --volumes || true
                        docker rm -f mongo-jenkins backend-jenkins frontend-jenkins 2>/dev/null || true
                    '''
                }
            }
        }
        
        stage('Checkout') {
            steps {
                echo 'Checking out code from GitHub...'
                checkout scm
            }
        }
        
        stage('Verify Files') {
            steps {
                script {
                    echo 'Verifying project structure...'
                    sh '''
                        ls -la
                        echo "=== Client Directory ==="
                        ls -la client/ || echo "Client directory not found"
                        echo "=== Server Directory ==="
                        ls -la server/ || echo "Server directory not found"
                        echo "=== Docker Compose File ==="
                        cat ${COMPOSE_FILE}
                    '''
                }
            }
        }
        
        stage('Update Environment Files') {
            steps {
                script {
                    echo 'Updating .env files for Jenkins environment...'
                    sh '''
                        # Update backend .env
                        cat > server/.env << EOF
DATABASE=mongodb://mongo-jenkins:27017/ecommerce
PORT=5000
BRAINTREE_MERCHANT_ID=n74dc2kw9g3ws389
BRAINTREE_PUBLIC_KEY=bgytmgzhz5f6t2tg
BRAINTREE_PRIVATE_KEY=e6f226166da99d874f00008f0bba14fe
EOF
                        
                        # Update frontend .env
                        cat > client/.env << EOF
REACT_APP_API_URL=http://51.20.104.42:5001
EOF
                        
                        echo "Environment files updated successfully"
                    '''
                }
            }
        }
        
        stage('Build and Deploy') {
            steps {
                script {
                    echo 'Building and starting containers with Docker Compose...'
                    sh '''
                        docker-compose -f ${COMPOSE_FILE} up -d
                        echo "Waiting for containers to start..."
                        sleep 15
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo 'Checking container health...'
                    sh '''
                        echo "=== Container Status ==="
                        docker ps -a | grep jenkins
                        
                        echo "\\n=== Backend Logs ==="
                        docker logs backend-jenkins --tail 20
                        
                        echo "\\n=== Frontend Logs ==="
                        docker logs frontend-jenkins --tail 20
                        
                        echo "\\n=== MongoDB Status ==="
                        docker exec mongo-jenkins mongosh --eval "db.adminCommand('ping')" || echo "MongoDB check skipped"
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo 'Verifying application is accessible...'
                    sh '''
                        echo "Testing backend endpoint..."
                        curl -f http://localhost:5001 || echo "Backend not yet ready"
                        
                        echo "Testing frontend endpoint..."
                        curl -f http://localhost:3001 || echo "Frontend not yet ready"
                        
                        echo "\\nDeployment verification complete!"
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '✅ Pipeline executed successfully!'
            echo 'Frontend: http://51.20.104.42:3001'
            echo 'Backend: http://51.20.104.42:5001'
            echo 'MongoDB: localhost:27018'
        }
        failure {
            echo ' Pipeline failed! Check logs above.'
            sh 'docker-compose -f ${COMPOSE_FILE} logs --tail 50'
        }
        always {
            echo 'Pipeline execution completed.'
            sh 'docker ps -a'
        }
    }
}
