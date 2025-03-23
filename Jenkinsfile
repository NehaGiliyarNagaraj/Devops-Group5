pipeline {
    agent any

    environment {
        // Docker Hub username (not a secret, just for image tagging)
        DOCKERHUB_ACCOUNT = 'ak2267'  // Your Docker Hub username
        PATH = "/usr/local/bin:/opt/homebrew/bin:${env.PATH}"  // Add both possible Docker paths
        DOCKER_HOST = "unix:///Users/adarshkumar/.docker/run/docker.sock"  // Correct Docker socket path
        DOCKER_CONTEXT = "desktop-linux"  // Docker context
        HOME = "/Users/adarshkumar"  // Set home directory
    }

    stages {
        stage('Debug Environment') {
            steps {
                sh '''
                    echo "Current PATH: $PATH"
                    echo "Docker location: $(which docker)"
                    echo "Docker version: $(docker --version)"
                    echo "Docker socket exists: $(test -S /Users/adarshkumar/.docker/run/docker.sock && echo "Yes" || echo "No")"
                    echo "Docker context: $(docker context ls)"
                '''
            }
        }

        stage('Checkout Repository') {
            steps {
                // Clone the GitHub repo (uses the Jenkins built-in 'git' step)
                git url: 'https://github.com/F21AO-Group5/Devops-Group5.git', branch: 'main'
            }
        }

        stage('Run Tests') {
            environment {
                // Test environment variables
                NODE_ENV = 'test'
                MONGO_URI = 'mongodb://localhost:27018/user-service-test'  // Changed port to 27018
                JWT_SECRET = 'test-secret'
            }
            steps {
                script {
                    // Clean up any existing test containers first
                    sh '''
                        echo "Cleaning up any existing test containers..."
                        docker rm -f mongodb-test || true
                        
                        echo "Starting MongoDB container for testing..."
                        docker run -d --name mongodb-test -p 27018:27017 mongo:latest
                        
                        # Wait for MongoDB to be ready
                        echo "Waiting for MongoDB to start..."
                        for i in $(seq 1 30); do
                            if docker exec mongodb-test mongosh --eval "db.stats()" > /dev/null 2>&1; then
                                echo "MongoDB is ready!"
                                break
                            fi
                            if [ $i -eq 30 ]; then
                                echo "Error: MongoDB failed to start"
                                exit 1
                            fi
                            echo "Waiting... ($i/30)"
                            sleep 1
                        done
                    '''
                    
                    dir('user-service') {
                        // Install dependencies and run tests
                        sh '''
                            echo "Installing dependencies for user-service..."
                            npm install
                            
                            # Install test reporter locally
                            npm install mocha-junit-reporter --save-dev
                            
                            echo "Running user-service tests..."
                            # Run tests with JUnit reporter for Jenkins integration
                            export NODE_ENV=test
                            export MOCHA_FILE="./test-results.xml"
                            export MONGO_URI="mongodb://localhost:27018/user-service-test"
                            export JWT_SECRET="test-secret"
                            
                            # Run tests using local reporter
                            npx mocha \
                                --recursive \
                                --timeout 5000 \
                                --reporter mocha-junit-reporter \
                                --exit \
                                "./test/**/*.test.js" || exit 1
                        '''
                    }
                }
            }
            post {
                always {
                    // Archive the test results
                    junit allowEmptyResults: true, testResults: '**/test-results.xml'
                    
                    // Clean up test container
                    sh '''
                        echo "Cleaning up test containers..."
                        docker rm -f mongodb-test || true
                    '''
                }
            }
        }

        stage('Build Docker Images') {
            steps {
                // Build each service's Docker image and tag with Jenkins build number
                sh 'docker build -t $DOCKERHUB_ACCOUNT/user-service:$BUILD_NUMBER ./user-service'
                sh 'docker build -t $DOCKERHUB_ACCOUNT/patient-service:$BUILD_NUMBER ./patient-service'
                sh 'docker build -t $DOCKERHUB_ACCOUNT/referral-service:$BUILD_NUMBER ./referral-service'
                sh 'docker build -t $DOCKERHUB_ACCOUNT/lab-service:$BUILD_NUMBER ./lab-service'
            }
        }

        stage('Docker Test') {
            steps {
                sh 'docker --version'
            }
        }

        stage('Docker Hub Login and Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub-id', passwordVariable: 'DOCKER_PASSWORD', usernameVariable: 'DOCKER_USERNAME')]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker info
                        
                        # Push images with error handling
                        services=("user-service" "patient-service" "referral-service" "lab-service")
                        for service in "${services[@]}"; do
                            echo "Pushing $DOCKERHUB_ACCOUNT/$service:$BUILD_NUMBER to Docker Hub..."
                            if docker push $DOCKERHUB_ACCOUNT/$service:$BUILD_NUMBER; then
                                echo "Successfully pushed $service image"
                            else
                                echo "Failed to push $service image. Check Docker Hub permissions and connection."
                                exit 1
                            fi
                        done
                    '''
                }
            }
        }

        stage('Deploy with Docker Compose') {
            steps {
                // Clean up existing containers and deploy new ones
                sh '''
                    # Stop and remove existing containers
                    docker-compose down --remove-orphans
                    
                    # Remove the mongodb container specifically if it exists
                    docker rm -f mongodb || true
                    
                    # Deploy all containers using docker-compose
                    docker-compose up -d
                '''
            }
        }
    }

    post {
        failure {
            echo "Build failed! Please check the logs."
        }
        success {
            echo "Build and deployment successful."
        }
    }
}
