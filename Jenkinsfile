pipeline {
    agent any

    environment {
        DOCKER_IMAGE = "aceest-fitness-app:${env.BUILD_ID}"
    }

    stages {
        stage('Initialize Environment') {
            steps {
                script {
                    echo 'Checking for Docker and Python CLI...'
                    sh '''
                        if ! command -v docker &> /dev/null; then
                            echo "Docker not found, installing..."
                            sudo apt-get update && sudo apt-get install -y docker.io
                        fi
                        
                        if ! command -v python3 &> /dev/null; then
                            echo "Python not found, installing..."
                            sudo apt-get update && sudo apt-get install -y python3 python3-pip python3-venv
                        fi
                    '''
                }
            }
        }

        stage('Build & Test Local') {
            steps {
                echo 'Installing requirements and running Pytest...'
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt
                    pytest
                '''
            }
        }

        stage('Docker Build & Verify') {
            steps {
                echo 'Building and verifying Docker image...'
                sh '''
                    docker build -t ${DOCKER_IMAGE} .
                    # Phase 6: Run tests INSIDE the container
                    docker run --rm ${DOCKER_IMAGE} pytest
                '''
            }
        }
    }

    post {
        failure {
            echo "Build failed. If 'permission denied' appears, the jenkins user needs docker group access."
        }
    }
}