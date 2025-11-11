pipeline {
    agent any
    
    environment {
        COMPOSE_FILE = 'docker-compose-jenkins.yml'
        PROJECT_DIR = "${WORKSPACE}"
    }
    
    stages {
        stage('Cleanup - Stop All Jenkins Containers') {
            steps {
                script {
                    echo '🧹 Stopping and removing all Jenkins containers...'
                    sh '''
                        cd ${PROJECT_DIR}
                        docker-compose -f ${COMPOSE_FILE} down --volumes 2>/dev/null || true
                        docker rm -f mongo-jenkins backend-jenkins frontend-jenkins 2>/dev/null || true
                        docker volume rm mern_stack_project_ecommerce_hayroo_mongo_data_jenkins 2>/dev/null || true
                        echo "✅ All Jenkins containers cleaned up"
                    '''
                }
            }
        }
        
        stage('Checkout Code from GitHub') {
            steps {
                echo '📥 Fetching latest code from GitHub repository...'
                checkout scm
                sh '''
                    echo "Current directory: $(pwd)"
                    echo "Branch: ${GIT_BRANCH}"
                    echo "Commit: ${GIT_COMMIT}"
                '''
            }
        }
        
        stage('Verify Project Structure') {
            steps {
                script {
                    echo '🔍 Verifying project files...'
                    sh '''
                        echo "=== Root Directory ==="
                        ls -la
                        
                        echo "\n=== Client Directory ==="
                        ls -la client/ 2>/dev/null || echo "❌ Client directory not found"
                        
                        echo "\n=== Server Directory ==="
                        ls -la server/ 2>/dev/null || echo "❌ Server directory not found"
                        
                        echo "\n=== Docker Compose File ==="
                        cat ${COMPOSE_FILE} 2>/dev/null || echo "❌ docker-compose-jenkins.yml not found"
                        
                        # Check if server.js exists
                        if [ -f "server/server.js" ]; then
                            echo "✅ server.js found"
                        elif [ -f "server/index.js" ]; then
                            echo "✅ index.js found"
                        else
                            echo "❌ No server entry point found"
                        fi
                    '''
                }
            }
        }
        
        stage('Configure Environment Variables') {
            steps {
                script {
                    echo '⚙️  Creating environment configuration files...'
                    sh '''
                        # Backend .env
                        cat > server/.env << 'EOF'
DATABASE=mongodb://mongo-jenkins:27017/ecommerce
PORT=5000
BRAINTREE_MERCHANT_ID=n74dc2kw9g3ws389
BRAINTREE_PUBLIC_KEY=bgytmgzhz5f6t2tg
BRAINTREE_PRIVATE_KEY=e6f226166da99d874f00008f0bba14fe
EOF
                        
                        # Frontend .env
                        cat > client/.env << 'EOF'
REACT_APP_API_URL=http://51.20.104.42:5001
EOF
                        
                        echo "✅ Environment files created"
                        echo "Backend .env:"
                        cat server/.env
                        echo "\nFrontend .env:"
                        cat client/.env
                    '''
                }
            }
        }
        
        stage('Fix Backend Entry Point') {
            steps {
                script {
                    echo '🔧 Ensuring correct backend entry point...'
                    sh '''
                        cd server
                        
                        # Check which file exists
                        if [ -f "app.js" ]; then
                            echo "✅ Using app.js as entry point"
                            ENTRY_FILE="app.js"
                        elif [ -f "server.js" ]; then
                            echo "✅ Using server.js as entry point"
                            ENTRY_FILE="server.js"
                        elif [ -f "index.js" ]; then
                            echo "✅ Using index.js as entry point"
                            ENTRY_FILE="index.js"
                        else
                            echo "❌ ERROR: No entry point file found!"
                            exit 1
                        fi
                        
                        # Update docker-compose command with correct entry point
                        cd ..
                        sed -i "s/node app.js/node ${ENTRY_FILE}/g" ${COMPOSE_FILE}
                        sed -i "s/node server.js/node ${ENTRY_FILE}/g" ${COMPOSE_FILE}
                        sed -i "s/node index.js/node ${ENTRY_FILE}/g" ${COMPOSE_FILE}
                        
                        echo "Updated docker-compose backend command to use ${ENTRY_FILE}"
                    '''
                }
            }
        }
        
        stage('Build and Start Containers') {
            steps {
                script {
                    echo '🚀 Starting containerized application...'
                    sh '''
                        cd ${PROJECT_DIR}
                        docker-compose -f ${COMPOSE_FILE} up -d
                        
                        echo "⏳ Waiting 30 seconds for services to initialize..."
                        sleep 30
                    '''
                }
            }
        }
        
        stage('Health Check') {
            steps {
                script {
                    echo '🏥 Checking container health...'
                    sh '''
                        echo "\n=== Container Status ==="
                        docker ps -a | head -1
                        docker ps -a | grep jenkins || echo "No jenkins containers found"
                        
                        echo "\n=== MongoDB Status ==="
                        docker logs mongo-jenkins --tail 10 2>/dev/null || echo "MongoDB logs unavailable"
                        
                        echo "\n=== Backend Status ==="
                        docker logs backend-jenkins --tail 20 2>/dev/null || echo "Backend logs unavailable"
                        
                        echo "\n=== Frontend Status ==="
                        docker logs frontend-jenkins --tail 20 2>/dev/null || echo "Frontend logs unavailable"
                    '''
                }
            }
        }
        
        stage('Verify Deployment') {
            steps {
                script {
                    echo '✅ Verifying application endpoints...'
                    sh '''
                        echo "\n🔗 Testing Backend (Port 5001)..."
                        curl -v http://localhost:5001 2>&1 | head -20 || echo "⚠️  Backend not responding yet"
                        
                        echo "\n🔗 Testing Frontend (Port 3001)..."
                        curl -v http://localhost:3001 2>&1 | head -20 || echo "⚠️  Frontend not responding yet"
                        
                        echo "\n📊 Final Container Status:"
                        docker ps | grep jenkins
                    '''
                }
            }
        }
    }
    
    post {
        success {
            echo '''
            ✅ ======================================
            ✅  PIPELINE EXECUTED SUCCESSFULLY!
            ✅ ======================================
            
            📍 Application URLs:
               Frontend: http://51.20.104.42:3001
               Backend:  http://51.20.104.42:5001
               MongoDB:  localhost:27018
            
            💡 Note: Applications may take 1-2 minutes to fully start
            '''
        }
        failure {
            echo '❌ Pipeline failed! Checking logs...'
            sh '''
                echo "\n=== Detailed Error Logs ==="
                docker-compose -f ${COMPOSE_FILE} logs --tail 100
                
                echo "\n=== Container Status ==="
                docker ps -a
            '''
        }
        always {
            echo '📝 Pipeline execution completed.'
            sh 'docker ps -a | grep jenkins || echo "No jenkins containers running"'
        }
    }
}
