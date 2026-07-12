pipeline {
    agent any

    environment {
        IMAGE_NAME = "rohit261/rudrabannataxiservices"
        IMAGE_TAG = "${BUILD_NUMBER}"
        NAMESPACE = "taxi-prod"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
                echo "✅ Code checked out successfully"
            }
        }

        stage('Build Maven') {
            steps {
                sh '''
                    echo "Building with Maven..."
                    mvn clean package -DskipTests
                    ls -la target/*.jar
                '''
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                    echo "Building Docker Image..."
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                    docker images | grep rudrabannataxiservices
                """
            }
        }

        stage('Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                        echo "Logging into Docker Hub..."
                        echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                        echo "Pushing images..."
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to K8s') {
            steps {
                sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@localhost "
                        kubectl apply -f /home/ubuntu/taxi_DevOpsflow/k8s/namespace.yaml
                        kubectl apply -f /home/ubuntu/taxi_DevOpsflow/k8s/secret.yaml -n ${NAMESPACE}
                        kubectl apply -f /home/ubuntu/taxi_DevOpsflow/k8s/configmap.yaml -n ${NAMESPACE}
                        kubectl apply -f /home/ubuntu/taxi_DevOpsflow/k8s/mysql-service.yaml -n ${NAMESPACE}
                        kubectl apply -f /home/ubuntu/taxi_DevOpsflow/k8s/mysql-statefulset.yaml -n ${NAMESPACE}
                        echo 'Waiting for MySQL...'
                        kubectl wait --for=condition=ready pod -l app=mysql -n ${NAMESPACE} --timeout=120s || true
                        kubectl apply -f /home/ubuntu/taxi_DevOpsflow/k8s/backend-deployment.yaml -n ${NAMESPACE}
                        kubectl apply -f /home/ubuntu/taxi_DevOpsflow/k8s/backend-service.yaml -n ${NAMESPACE}
                        kubectl set image deployment/taxi-backend taxi-backend=${IMAGE_NAME}:${IMAGE_TAG} -n ${NAMESPACE}
                        echo '✅ Deployment applied!'
                    "
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    ssh -o StrictHostKeyChecking=no ubuntu@localhost "
                        echo '=== VERIFYING DEPLOYMENT ==='
                        kubectl rollout status deployment/taxi-backend -n ${NAMESPACE} --timeout=120s || true
                        echo ''
                        echo 'Pods:'
                        kubectl get pods -n ${NAMESPACE} -o wide
                        echo ''
                        echo 'Services:'
                        kubectl get svc -n ${NAMESPACE}
                        echo '=== DONE ==='
                    "
                """
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline Completed Successfully!"
        }
        failure {
            echo "❌ Pipeline Failed! Rolling back..."
            sh """
                ssh -o StrictHostKeyChecking=no ubuntu@localhost "kubectl rollout undo deployment/taxi-backend -n ${NAMESPACE} || true"
            """
        }
        always {
            echo "Pipeline completed. Build: ${env.BUILD_NUMBER}"
        }
    }
}
