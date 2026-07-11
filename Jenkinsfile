pipeline {
    agent any

    environment {
        // Docker Hub
        IMAGE_NAME = "rohit261/rudrabannataxiservices"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        
        // Kubernetes
        NAMESPACE  = "taxi-prod"
        KUBECONFIG = credentials('kubeconfig-file')
        
        // Slack/Email Notifications (Optional)
        SLACK_CHANNEL = '#deployments'
    }

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '10'))
    }

    triggers {
        // Auto-trigger on GitHub push
        pollSCM('* * * * *') // Every minute (GitHub webhook better)
    }

    stages {

        stage('📥 Checkout Code') {
            steps {
                checkout scm
                echo "Branch: ${env.BRANCH_NAME}"
                echo "Commit: ${env.GIT_COMMIT}"
            }
        }

        stage('🔍 Code Quality Check') {
            steps {
                sh '''
                    echo "Running code quality checks..."
                    mvn checkstyle:check || true
                    echo "Code quality check completed"
                '''
            }
        }

        stage('🏗️ Build Application') {
            steps {
                sh '''
                    echo "Building with Maven..."
                    mvn clean package -DskipTests
                    echo "Build completed successfully!"
                    ls -la target/*.jar
                '''
            }
        }

        stage('🧪 Run Tests') {
            steps {
                sh '''
                    echo "Running unit tests..."
                    mvn test
                '''
            }
            post {
                always {
                    junit 'target/surefire-reports/*.xml'
                }
            }
        }

        stage('🐳 Build Docker Image') {
            steps {
                sh """
                    echo "Building Docker image: ${IMAGE_NAME}:${IMAGE_TAG}"
                    docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                    docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                    echo "Docker image built successfully!"
                """
            }
        }

        stage('📤 Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo "Logging into Docker Hub..."
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                        
                        echo "Pushing images..."
                        docker push ${IMAGE_NAME}:${IMAGE_TAG}
                        docker push ${IMAGE_NAME}:latest
                        
                        echo "Cleaning up..."
                        docker logout
                    """
                }
            }
        }

        stage('☸️ Deploy to Kubernetes') {
            steps {
                script {
                    // Configure kubectl
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        
                        echo "Deploying to Kubernetes namespace: ${NAMESPACE}"
                        
                        // Apply namespace if not exists
                        sh """
                            kubectl get namespace ${NAMESPACE} || kubectl create namespace ${NAMESPACE}
                        """
                        
                        // Deploy MySQL first
                        sh """
                            echo "Deploying MySQL..."
                            kubectl apply -f k8s/namespace.yaml
                            kubectl apply -f k8s/secret.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/configmap.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/pvc.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/mysql-service.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/mysql-statefulset.yaml -n ${NAMESPACE}
                        """
                        
                        // Wait for MySQL
                        sh """
                            echo "Waiting for MySQL to be ready..."
                            kubectl wait --for=condition=ready pod -l app=mysql -n ${NAMESPACE} --timeout=300s
                            echo "MySQL is ready!"
                        """
                        
                        // Deploy Application
                        sh """
                            echo "Deploying Application..."
                            kubectl set image deployment/taxi-backend \
                                taxi-backend=${IMAGE_NAME}:${IMAGE_TAG} \
                                -n ${NAMESPACE} || true
                                
                            kubectl apply -f k8s/backend-deployment.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/backend-service.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/hpa.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/ingress.yaml -n ${NAMESPACE}
                        """
                        
                        // Deploy Monitoring (Optional)
                        sh """
                            echo "Deploying Monitoring Stack..."
                            kubectl apply -f k8s/prometheus-deployment.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/prometheus-service.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/grafana-deployment.yaml -n ${NAMESPACE}
                            kubectl apply -f k8s/grafana-service.yaml -n ${NAMESPACE}
                        """
                    }
                }
            }
        }

        stage('✅ Verify Deployment') {
            steps {
                script {
                    withKubeConfig([credentialsId: 'kubeconfig']) {
                        sh """
                            echo "========================================"
                            echo "Deployment Status"
                            echo "========================================"
                            
                            echo "Checking rollout status..."
                            kubectl rollout status deployment/taxi-backend -n ${NAMESPACE} --timeout=300s
                            
                            echo ""
                            echo "Pods:"
                            kubectl get pods -n ${NAMESPACE}
                            
                            echo ""
                            echo "Services:"
                            kubectl get svc -n ${NAMESPACE}
                            
                            echo ""
                            echo "HPA Status:"
                            kubectl get hpa -n ${NAMESPACE}
                            
                            echo ""
                            echo "Ingress:"
                            kubectl get ingress -n ${NAMESPACE}
                            
                            echo ""
                            echo "========================================"
                            echo "Application Deployed Successfully! 🚀"
                            echo "========================================"
                        """
                        
                        // Get application URL
                        def serviceIP = sh(
                            script: "kubectl get svc taxi-backend-service -n ${NAMESPACE} -o jsonpath='{.status.loadBalancer.ingress[0].ip}'",
                            returnStdout: true
                        ).trim()
                        
                        if (serviceIP) {
                            echo "Application URL: http://${serviceIP}:8080"
                            echo "Health Check: http://${serviceIP}:8080/actuator/health"
                        }
                    }
                }
            }
        }

        stage('🧹 Cleanup') {
            steps {
                sh '''
                    echo "Cleaning up old Docker images..."
                    docker image prune -af --filter "until=24h"
                    
                    echo "Cleaning workspace..."
                    cleanWs()
                '''
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline Succeeded!"
            // Send notification
            emailext(
                to: 'devops-team@example.com',
                subject: "✅ SUCCESS: Pipeline ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: """
                    Pipeline succeeded!
                    Build: ${env.BUILD_NUMBER}
                    Job: ${env.JOB_NAME}
                    Image: ${IMAGE_NAME}:${IMAGE_TAG}
                    Namespace: ${NAMESPACE}
                    URL: ${env.BUILD_URL}
                """
            )
        }
        
        failure {
            echo "❌ Pipeline Failed!"
            
            script {
                withKubeConfig([credentialsId: 'kubeconfig']) {
                    // Rollback
                    sh """
                        echo "Rolling back deployment..."
                        kubectl rollout undo deployment/taxi-backend -n ${NAMESPACE} || true
                    """
                }
            }
            
            emailext(
                to: 'devops-team@example.com',
                subject: "❌ FAILED: Pipeline ${env.JOB_NAME} - Build #${env.BUILD_NUMBER}",
                body: "Pipeline failed! Check: ${env.BUILD_URL}"
            )
        }
        
        always {
            echo "Pipeline completed. Build: ${env.BUILD_NUMBER}"
            cleanWs()
        }
    }
}
