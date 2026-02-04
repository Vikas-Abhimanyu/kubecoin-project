pipeline {
    agent any

    environment {
        DOCKER_USER = "vikasabhimanyu"
        DOCKER_CRED = "dockerhub-creds"
        KUBECONFIG  = credentials('kubeconfig-creds')

        GIT_SHA = ''
        IMAGE_TAG = ''

        BACKEND_IMAGE = ''
        FRONTEND_IMAGE = ''
        ENV = ''
        NAMESPACE = ''
    }

    stages {

        stage('Determine Environment') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'DEV') {
                        ENV = 'dev'
                        NAMESPACE = 'dev'
                    } else if (env.BRANCH_NAME == 'TESTING') {
                        ENV = 'testing'
                        NAMESPACE = 'testing'
                    } else if (env.BRANCH_NAME == 'PRODUCTION') {
                        ENV = 'production'
                        NAMESPACE = 'production'
                    } else {
                        error "Unsupported branch: ${env.BRANCH_NAME}"
                    }

                    GIT_SHA = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    IMAGE_TAG = "${ENV}-${GIT_SHA}"

                    BACKEND_IMAGE  = "${DOCKER_USER}/backend:${IMAGE_TAG}"
                    FRONTEND_IMAGE = "${DOCKER_USER}/frontend:${IMAGE_TAG}"

                    echo "Deploying to ${ENV} with tag ${IMAGE_TAG}"
                }
            }
        }

        stage('Build & Push Images') {
            parallel {

                stage('Backend') {
                    steps {
                        withCredentials([usernamePassword(
                            credentialsId: DOCKER_CRED,
                            usernameVariable: 'USER',
                            passwordVariable: 'PASS'
                        )]) {
                            sh '''
                              echo "$PASS" | docker login -u "$USER" --password-stdin
                              docker build -t $BACKEND_IMAGE backend
                              docker push $BACKEND_IMAGE
                            '''
                        }
                    }
                }

                stage('Frontend') {
                    steps {
                        withCredentials([usernamePassword(
                            credentialsId: DOCKER_CRED,
                            usernameVariable: 'USER',
                            passwordVariable: 'PASS'
                        )]) {
                            sh '''
                              echo "$PASS" | docker login -u "$USER" --password-stdin
                              docker build -t $FRONTEND_IMAGE frontend
                              docker push $FRONTEND_IMAGE
                            '''
                        }
                    }
                }

            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh """
                  sed -i 's|IMAGE_BACKEND|$BACKEND_IMAGE|g' k8s/${ENV}/backend-deployment.yaml
                  sed -i 's|IMAGE_FRONTEND|$FRONTEND_IMAGE|g' k8s/${ENV}/frontend-deployment.yaml

                  kubectl apply -n ${NAMESPACE} -f k8s/${ENV}/

                  kubectl rollout status deployment/backend -n ${NAMESPACE}
                  kubectl rollout status deployment/frontend -n ${NAMESPACE}
                """
            }
        }
    }

    post {
        failure {
            echo "Deployment failed — check image build or cluster state."
        }
    }
}

