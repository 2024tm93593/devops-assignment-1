pipeline {
    agent any
    parameters {
        string(
            name: 'IMAGE_VERSION',
            defaultValue: '',
            description: 'Docker image version tag. Leave blank to use the Jenkins build number.'
        )
        choice(
            name: 'DEPLOY_STRATEGY',
            choices: ['blue-green', 'rolling', 'canary', 'shadow', 'ab-testing'],
            description: 'Deployment strategy to use.'
        )
    }
    environment {
        IMAGE_TAG = "${params.IMAGE_VERSION ?: env.BUILD_ID}"
        DOCKER_IMAGE = "aceest-fitness-app:${IMAGE_TAG}"
        DOCKERHUB_IMAGE = "chetan56881/aceest-fitness-app"
    }
    stages {
        stage('Env Setup & Test') {
            steps {
                sh '''
                    apt-get update && apt-get install -y docker.io python3-venv python3-pip

                    python3 -m venv venv
                    . venv/bin/activate
                    pip install -r requirements.txt

                    export PYTHONPATH=$PYTHONPATH:.
                    pytest tests/ -v --cov=. --cov-report=xml
                '''
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarCloud') {
                    withEnv(["PATH+SONAR=${tool 'SonarScanner'}/bin"]) {
                        sh '''
                            sonar-scanner \
                                -Dsonar.projectKey=2024tm93593_devops-assignment-1 \
                                -Dsonar.projectName="2024tm93593_devops-assignment-1" \
                                -Dsonar.sources=. \
                                -Dsonar.exclusions=**/venv/**,**/__pycache__/**,**/*.pyc,**/tests/** \
                                -Dsonar.tests=tests \
                                -Dsonar.python.version=3.11 \
                                -Dsonar.python.coverage.reportPaths=coverage.xml \
                                -Dsonar.host.url=https://sonarcloud.io
                        '''
                    }
                }
            }
        }
        stage('Build & Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-credentials',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker buildx build --progress=plain --push \
                            -t $DOCKERHUB_IMAGE:${IMAGE_TAG} \
                            -t $DOCKERHUB_IMAGE:latest \
                            .
                        docker logout
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                script {
                    withCredentials([
                        sshUserPrivateKey(
                            credentialsId: 'gcp-vm-ssh-key',
                            keyFileVariable: 'SSH_KEY'
                        ),
                        string(credentialsId: 'GCP_VM_IP', variable: 'VM_IP'),
                        string(credentialsId: 'GCP_VM_USER', variable: 'VM_USER')
                    ]) {
                        if (params.DEPLOY_STRATEGY == 'blue-green') {
                            sh '''
                                chmod 600 $SSH_KEY
                                ssh -i $SSH_KEY -o StrictHostKeyChecking=no $VM_USER@$VM_IP "mkdir -p ~/blue-green"
                                scp -i $SSH_KEY -o StrictHostKeyChecking=no \
                                    k8s/blue-green/namespaces.yaml \
                                    k8s/blue-green/blue-deployment.yaml \
                                    k8s/blue-green/green-deployment.yaml \
                                    k8s/blue-green/blue-service.yaml \
                                    k8s/blue-green/green-service.yaml \
                                    k8s/blue-green/router-service.yaml \
                                    k8s/blue-green/state-configmap.yaml \
                                    k8s/blue-green/switch-traffic.sh \
                                    k8s/blue-green/rollback.sh \
                                    $VM_USER@$VM_IP:~/blue-green/
                                ssh -i $SSH_KEY -o StrictHostKeyChecking=no $VM_USER@$VM_IP "
                                    set -e
                                    chmod +x ~/blue-green/switch-traffic.sh ~/blue-green/rollback.sh

                                    kubectl get namespace blue &>/dev/null || kubectl create namespace blue
                                    kubectl get namespace green &>/dev/null || kubectl create namespace green
                                    kubectl get configmap blue-green-state -n default &>/dev/null || kubectl apply -f ~/blue-green/state-configmap.yaml
                                    kubectl get svc aceest-fitness-app-router -n default &>/dev/null || kubectl apply -f ~/blue-green/router-service.yaml
                                    kubectl get svc aceest-fitness-app -n blue &>/dev/null || kubectl apply -f ~/blue-green/blue-service.yaml
                                    kubectl get svc aceest-fitness-app -n green &>/dev/null || kubectl apply -f ~/blue-green/green-service.yaml

                                    ACTIVE=\\$(kubectl get configmap blue-green-state -n default -o jsonpath='{.data.active-color}')
                                    if [ \\\"\\$ACTIVE\\\" = \\\"blue\\\" ]; then TARGET=\\\"green\\\"; else TARGET=\\\"blue\\\"; fi

                                    echo '================================================'
                                    echo \\\"  BLUE-GREEN DEPLOY: \\$ACTIVE --> \\$TARGET\\\"
                                    echo '================================================'

                                    kubectl set image deployment/aceest-fitness-app -n \\$TARGET aceest-fitness-app=${DOCKERHUB_IMAGE}:${IMAGE_TAG}
                                    kubectl apply -f ~/blue-green/\\${TARGET}-deployment.yaml
                                    kubectl rollout status deployment/aceest-fitness-app -n \\$TARGET --timeout=120s

                                    echo '--- Pre-switch: pods in blue namespace ---'
                                    kubectl get pods -n blue -l app=aceest-fitness-app
                                    echo '--- Pre-switch: pods in green namespace ---'
                                    kubectl get pods -n green -l app=aceest-fitness-app
                                    echo '--- Router endpoints (current) ---'
                                    kubectl get endpoints aceest-fitness-app-router -n default

                                    ~/blue-green/switch-traffic.sh \\$TARGET

                                    echo '--- Final router endpoints ---'
                                    kubectl get endpoints aceest-fitness-app-router -n default
                                    kubectl get configmap blue-green-state -n default -o yaml
                                "
                            '''
                        } else {
                            echo "Strategy '${params.DEPLOY_STRATEGY}' selected — coming soon. Falling back to standard deploy."
                            sh '''
                                chmod 600 $SSH_KEY
                                scp -i $SSH_KEY -o StrictHostKeyChecking=no k8s/deployment.yaml $VM_USER@$VM_IP:~/deployment.yaml
                                ssh -i $SSH_KEY -o StrictHostKeyChecking=no $VM_USER@$VM_IP "kubectl apply -f ~/deployment.yaml"
                            '''
                        }
                    }
                }
            }
        }
    }
    post {
        failure {
            script {
                if (params.DEPLOY_STRATEGY == 'blue-green') {
                    withCredentials([
                        sshUserPrivateKey(credentialsId: 'gcp-vm-ssh-key', keyFileVariable: 'SSH_KEY'),
                        string(credentialsId: 'GCP_VM_IP', variable: 'VM_IP'),
                        string(credentialsId: 'GCP_VM_USER', variable: 'VM_USER')
                    ]) {
                        sh '''
                            chmod 600 $SSH_KEY
                            ssh -i $SSH_KEY -o StrictHostKeyChecking=no $VM_USER@$VM_IP "
                                chmod +x ~/blue-green/rollback.sh
                                ~/blue-green/rollback.sh
                                echo '--- Rollback: pods in blue ---'
                                kubectl get pods -n blue -l app=aceest-fitness-app
                                echo '--- Rollback: pods in green ---'
                                kubectl get pods -n green -l app=aceest-fitness-app
                                echo '--- Rollback: router endpoints ---'
                                kubectl get endpoints aceest-fitness-app-router -n default
                            "
                        '''
                    }
                }
            }
        }
        always {
            cleanWs()
        }
    }
}
