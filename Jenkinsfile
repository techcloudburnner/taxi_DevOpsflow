pipeline {
    agent any

    environment {
        IMAGE_NAME = "rohit261/rudrabannataxiservices"
        IMAGE_TAG  = "${BUILD_NUMBER}"
        NAMESPACE  = "taxi-app"
    }

    options {
        timestamps()
        disableConcurrentBuilds()
    }

    stages {

        stage('Checkout Source') {
            steps {
                checkout scm
            }
        }

        stage('Build Application') {
            steps {
                sh 'mvn clean package -DskipTests'
            }
        }

        stage('Build Docker Image') {
            steps {
                sh """
                docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
                docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${IMAGE_NAME}:latest
                """
            }
        }

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-creds',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {

                    sh """
                    echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                    docker push ${IMAGE_NAME}:${IMAGE_TAG}
                    docker push ${IMAGE_NAME}:latest

                    docker logout
                    """
                }
            }
        }

        stage('Deploy To Kubernetes') {
            steps {
                sh """
                kubectl apply -f k8s/ -n ${NAMESPACE}

                kubectl set image deployment/taxi-backend \
                taxi-backend=${IMAGE_NAME}:${IMAGE_TAG} \
                -n ${NAMESPACE}
                """
            }
        }

        stage('Verify Deployment') {
            steps {
                sh """
                kubectl rollout status deployment/taxi-backend -n ${NAMESPACE}
                """
            }
        }

        stage('Show Resources') {
            steps {
                sh """
                kubectl get pods -n ${NAMESPACE}
                kubectl get svc -n ${NAMESPACE}
                kubectl get ingress -n ${NAMESPACE}
                """
            }
        }

        stage('Cleanup') {
            steps {
                sh '''
                docker image prune -af
                '''
            }
        }
    }

    post {

        success {
            echo "Application deployed successfully."
        }

        failure {
            echo "Deployment failed."

            sh """
            kubectl rollout undo deployment/taxi-backend -n ${NAMESPACE} || true
            """
        }

        always {
            cleanWs()
        }
    }
}