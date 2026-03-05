pipeline {
    agent any

    triggers {
        githubPush()
    }

    environment {
        DOCKER_USER     = "vikasabhimanyu"
        DOCKER_CRED     = "dockerhub-creds"
        GIT_CRED        = "github-creds"
        KUBECONFIG_CRED = "kubeconfig-creds"
    }

    stages {

        stage('Determine Environment') {
            steps {
                script {

                    switch(env.BRANCH_NAME.toLowerCase()) {
                        case 'dev':
                            env.ENV = 'dev'
                            env.NAMESPACE = 'dev'
                            break
                        case 'testing':
                            env.ENV = 'testing'
                            env.NAMESPACE = 'testing'
                            break
                        case 'production':
                            env.ENV = 'production'
                            env.NAMESPACE = 'production'
                            break
                        default:
                            error "Unsupported branch ${env.BRANCH_NAME}"
                    }

                    env.GIT_SHA = sh(
                        script: "git rev-parse --short HEAD",
                        returnStdout: true
                    ).trim()

                    env.BACKEND_IMAGE  = "${DOCKER_USER}/backend:${ENV}-${GIT_SHA}"
                    env.FRONTEND_IMAGE = "${DOCKER_USER}/frontend:${ENV}-${GIT_SHA}"

                    echo "Environment: ${ENV}"
                    echo "Namespace: ${NAMESPACE}"
                    echo "Backend Image: ${BACKEND_IMAGE}"
                    echo "Frontend Image: ${FRONTEND_IMAGE}"
                }
            }
        }

        stage('Build & Push Images') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${DOCKER_CRED}",
                        usernameVariable: 'DOCKER_USER_VAR',
                        passwordVariable: 'DOCKER_PASS_VAR'
                    )
                ]) {
                    sh '''
                        echo $DOCKER_PASS_VAR | docker login -u $DOCKER_USER_VAR --password-stdin

                        docker build -t ${BACKEND_IMAGE} ./backend
                        docker push ${BACKEND_IMAGE}

                        docker build -t ${FRONTEND_IMAGE} ./frontend
                        docker push ${FRONTEND_IMAGE}
                    '''
                }
            }
        }

        stage('Update Image Tag in GitHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: "${GIT_CRED}",
                        usernameVariable: 'GIT_USER',
                        passwordVariable: 'GIT_PASS'
                    )
                ]) {
                    sh '''
                        git config user.email "ci-bot@example.com"
                        git config user.name "CI Bot"

                        echo "Cleaning git state"
                        rm -rf .git/rebase-merge || true
                        rm -rf .git/rebase-apply || true

                        echo "Fetching latest repo"
                        git fetch origin

                        echo "Checking out branch"
                        git checkout ${BRANCH_NAME}

                        echo "Resetting workspace to remote branch"
                        git reset --hard origin/${BRANCH_NAME}

                        echo "Updating image tags in manifests"

                        sed -i "s|image: vikasabhimanyu/backend:.*|image: ${BACKEND_IMAGE}|g" k8s/backend-deployment.yaml
                        sed -i "s|image: vikasabhimanyu/frontend:.*|image: ${FRONTEND_IMAGE}|g" k8s/frontend-deployment.yaml

                        echo "Verifying image updates"
                        grep image k8s/backend-deployment.yaml
                        grep image k8s/frontend-deployment.yaml

                        git add k8s/*.yaml
                        git commit -m "Update image tags to ${GIT_SHA}" || echo "No changes to commit"

                        echo "Pushing updated manifests"
                        git push https://${GIT_USER}:${GIT_PASS}@github.com/Vikas-Abhimanyu/kubecoin-project.git ${BRANCH_NAME} --force
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([
                    file(credentialsId: "${KUBECONFIG_CRED}", variable: 'KUBECONFIG_FILE')
                ]) {
                    sh '''
                        export KUBECONFIG=$KUBECONFIG_FILE

                        echo "Applying Kubernetes manifests"

                        kubectl apply -f k8s/backend-deployment.yaml -n ${NAMESPACE}
                        kubectl apply -f k8s/frontend-deployment.yaml -n ${NAMESPACE}

                        echo "Checking deployment status"

                        kubectl get pods -n ${NAMESPACE}

                        kubectl rollout status deployment/backend -n ${NAMESPACE}
                        kubectl rollout status deployment/frontend -n ${NAMESPACE}
                    '''
                }
            }
        }
    }

    post {
        failure {
            echo "Deployment failed"
        }
        success {
            echo "Deployment successful"
        }
    }
}
