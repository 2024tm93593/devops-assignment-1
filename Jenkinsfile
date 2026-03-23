pipeline {
    agent {
        docker { 
            image 'python:3.9-slim' 
        }
    }

    environment {
        DOCKER_IMAGE = "aceest-fitness-app:${env.BUILD_ID}"
    }

    stages {
        stage('Install Dependencies') {
            steps {
                echo 'Installing requirements in the containerized agent...'
                sh 'pip install --user -r requirements.txt'
            }
        }

        stage('Unit Testing') {
            steps {
                echo 'Running Pytest...'
                sh 'python -m pytest'
            }
        }

        stage('Docker Build') {
            agent any 
            steps {
                echo 'Building the Fitness App Docker Image...'
                sh "docker build -t ${DOCKER_IMAGE} ."
            }
        }

        stage('Container Validation') {
            agent any
            steps {
                echo 'Final Quality Gate: Testing the built image...'
                sh "docker run --rm ${DOCKER_IMAGE} python -m pytest"
            }
        }
    }

    post {
        success {
            echo 'Deployment Pipeline Ready.'
        }
        failure {
            echo 'Build failed. Check the Quality Gate logs.'
        }
    }
}