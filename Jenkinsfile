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

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                    echo "Deploying to Kubernetes..."
                    
                    # Check kubectl connection
                    kubectl get nodes
                    
                    # Apply all configurations
                    kubectl apply -f k8s/namespace.yaml --validate=false
                    kubectl apply -f k8s/secret.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/configmap.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/pvc.yaml -n ${NAMESPACE}
                    
                    # Deploy MySQL first
                    kubectl apply -f k8s/mysql-service.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/mysql-statefulset.yaml -n ${NAMESPACE}
                    
                    echo "Waiting for MySQL to be ready..."
                    kubectl wait --for=condition=ready pod -l app=mysql -n ${NAMESPACE} --timeout=120s || true
                    
                    # Deploy Backend
                    kubectl apply -f k8s/backend-deployment.yaml -n ${NAMESPACE}
                    kubectl apply -f k8s/backend-service.yaml -n ${NAMESPACE}
                    
                    # Update image to latest build
                    kubectl set image deployment/taxi-backend taxi-backend=${IMAGE_NAME}:${IMAGE_TAG} -n ${NAMESPACE}
                    
                    echo "✅ Deployment applied!"
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                    echo "========================================="
                    echo "VERIFYING DEPLOYMENT"
                    echo "========================================="
                    
                    echo "Rollout Status:"
                    kubectl rollout status deployment/taxi-backend -n ${NAMESPACE} --timeout=120s || true
                    
                    echo ""
                    echo "Pods:"
                    kubectl get pods -n ${NAMESPACE} -o wide
                    
                    echo ""
                    echo "Services:"
                    kubectl get svc -n ${NAMESPACE}
                    
                    echo ""
                    echo "Deployments:"
                    kubectl get deployments -n ${NAMESPACE}
                    
                    echo ""
                    echo "========================================="
                """
            }
        }
    }

    post {
        success {
            echo "🎉 Pipeline Completed Successfully!"
        }
        failure {
            echo "❌ Pipeline Failed! Performing rollback..."
            sh """
                kubectl rollout undo deployment/taxi-backend -n ${NAMESPACE} || true
                echo "Rollback completed"
            """
        }
        always {
            echo "Pipeline completed. Build: ${env.BUILD_NUMBER}"
        }
    }
}
