pipeline {
  agent any
  tools { nodejs 'node22' }

  environment {
    IMAGE    = 'alwinjames313/jenkins_prac'
    IS_MAIN  = "${env.BRANCH_NAME == 'main'}"
    TAG      = "${env.BRANCH_NAME == 'main' ? '' : 'dev-'}${BUILD_NUMBER}"
    ENV_TAG  = "${env.BRANCH_NAME == 'main' ? 'latest' : 'dev'}"
    APP_HOST = "${env.BRANCH_NAME == 'main' ? 'ec2-user@54.82.75.88' : 'ec2-user@18.232.114.142'}"
    APP_URL  = "${env.BRANCH_NAME == 'main' ? 'http://54.82.75.88:3000' : 'http://18.232.114.142:3000'}"
  }

  stages {
    stage('Install') {
      steps { sh 'npm ci' }
    }

    stage('Test') {
      steps { sh 'npm test' }
    }

    stage('Release') {
      when { anyOf { branch 'main'; branch 'dev' } }
      stages {
        stage('Docker Build & Push') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                              usernameVariable: 'DH_USER',
                                              passwordVariable: 'DH_PASS')]) {
              sh '''
                echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                docker buildx build --platform linux/arm64 \
                  -t $IMAGE:$TAG -t $IMAGE:$ENV_TAG --push .
              '''
            }
          }
        }

        stage('Deploy') {
          steps {
            sshagent(['ec2-app-ssh']) {
              sh '''
                ssh -o StrictHostKeyChecking=no $APP_HOST "
                  docker pull $IMAGE:$TAG &&
                  (docker rm -f myapp || true) &&
                  docker run -d --name myapp --restart unless-stopped -p 3000:3000 $IMAGE:$TAG
                "
              '''
            }
          }
        }

        stage('Smoke Test') {
          steps { sh 'sleep 5 && curl -f $APP_URL/health' }
        }
      }
    }
  }

  post {
    always {
      sh 'docker logout || true'
      cleanWs()
    }
  }
}