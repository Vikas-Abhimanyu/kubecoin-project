pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        DOCKER_USER = "vikasabhimanyu"
        DOCKER_CRED = "dockerhub-creds"
        KUBECONFIG_CRED = "kubeconfig-creds"
    }

    stages {
        stage('Determine Environment') {
            steps {
                script {
                    if (env.BRANCH_NAME == 'DEV') {
                        env.ENV = 'dev'
                        env.NAMESPACE = 'dev'
                    } else if (env.BRANCH_NAME == 'TESTING') {
                        env.ENV = 'testing'
                        env.NAMESPACE = 'testing'
                    } else if (env.BRANCH_NAME == 'PRODUCTION') {
                        env.ENV = 'production'
                        env.NAMESPACE = 'production'
                    } else {
                        error "Unsupported branch: ${env.BRANCH_NAME}"
                    }

                    env.GIT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()

                    env.BACKEND_IMAGE = "${DOCKER_USER}/backend:${env.ENV}-${env.GIT_SHA}"
                    env.FRONTEND_IMAGE = "${DOCKER_USER}/frontend:${env.ENV}-${env.GIT_SHA}"

                    echo "Deploying to ${env.ENV} namespace with images:"
                    echo "Backend: ${env.BACKEND_IMAGE}"
                    echo "Frontend: ${env.FRONTEND_IMAGE}"
                }
            }
        }

        stage('Build & Push Backend Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED}", usernameVariable: 'DOCKER_USER_VAR', passwordVariable: 'DOCKER_PASS_VAR')]) {
                    sh '''
                      echo $DOCKER_PASS_VAR | docker login -u $DOCKER_USER_VAR --password-stdin
                      docker build -t ${BACKEND_IMAGE} ./backend
                      docker push ${BACKEND_IMAGE}
                    '''
                }
            }
        }

        stage('Build & Push Frontend Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED}", usernameVariable: 'DOCKER_USER_VAR', passwordVariable: 'DOCKER_PASS_VAR')]) {
                    sh '''
                      echo $DOCKER_PASS_VAR | docker login -u $DOCKER_USER_VAR --password-stdin
                      docker build -t ${FRONTEND_IMAGE} ./frontend
                      docker push ${FRONTEND_IMAGE}
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')]) {
                    sh '''
                      export KUBECONFIG=$KUBECONFIG_FILE
                      kubectl set image deployment/backend backend=${BACKEND_IMAGE} -n ${NAMESPACE}
                      kubectl set image deployment/frontend frontend=${FRONTEND_IMAGE} -n ${NAMESPACE}
                      kubectl rollout status deployment/backend -n ${NAMESPACE}
                      kubectl rollout status deployment/frontend -n ${NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        failure {
            echo "Deployment failed. Check build logs and cluster status."
        }
    }
}

