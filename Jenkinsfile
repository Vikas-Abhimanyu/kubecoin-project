pipeline {
  agent any

  environment {
    DOCKERHUB_REPO      = "vikasabhimanyu"
    DOCKER_CREDENTIALS = "dockerhub-creds"
    KUBECONFIG_CRED    = "kubeconfig-creds"
  }

  stages {

    stage('Determine Environment') {
      steps {
        script {
          if (env.BRANCH_NAME == 'DEV') {
            env.ENV = 'dev'
          } else if (env.BRANCH_NAME == 'TESTING') {
            env.ENV = 'testing'
          } else if (env.BRANCH_NAME == 'PRODUCTION') {
            env.ENV = 'production'
          }

          if (!env.ENV) {
            error("Invalid branch for deployment: ${env.BRANCH_NAME}")
          }
        }
        echo "Deploying to environment: ${env.ENV}"
      }
    }

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build Docker Images (Parallel)') {
      agent { label 'ubuntu-1' }
      steps {
        parallel(
          Backend: {
            sh "docker build -t $DOCKERHUB_REPO/backend:$ENV-$BUILD_NUMBER backend"
          },
          Frontend: {
            sh "docker build -t $DOCKERHUB_REPO/frontend:$ENV-$BUILD_NUMBER frontend"
          }
        )
      }
    }

    stage('Push Images to DockerHub') {
      agent { label 'ubuntu-1' }
      steps {
        withCredentials([usernamePassword(
          credentialsId: DOCKER_CREDENTIALS,
          usernameVariable: 'USER',
          passwordVariable: 'PASS'
        )]) {
          sh """
          echo $PASS | docker login -u $USER --password-stdin
          docker push $DOCKERHUB_REPO/backend:$ENV-$BUILD_NUMBER
          docker push $DOCKERHUB_REPO/frontend:$ENV-$BUILD_NUMBER
          """
        }
      }
    }
     stage('Deploy to Kubernetes') {
 	 agent { label 'ubuntu-2' }
  	steps {
    	withCredentials([file(credentialsId: KUBECONFIG_CRED, variable: 'KUBECONFIG')]) {
      		sh """
      		kubectl set image deployment/backend backend=$DOCKERHUB_REPO/backend:$ENV-$BUILD_NUMBER -n $ENV
      		kubectl set image deployment/frontend frontend=$DOCKERHUB_REPO/frontend:$ENV-$BUILD_NUMBER -n $ENV
      		"""
        }
      }
    }
  }
}

