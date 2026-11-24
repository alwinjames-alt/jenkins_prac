pipeline {
  agent any
  tools { nodejs 'node22' }

  environment {
    IMAGE    = 'alwinjames313/jenkins_prac'
    TAG      = "${BUILD_NUMBER}"
    APP_HOST = 'ec2-user@54.173.11.115'
    APP_URL  = 'http://54.173.11.115:3000'
  }

  stages {
    stage('Install') {
      steps { sh 'npm ci' }
    }

    stage('Test') {
      steps { sh 'npm test' }
    }

    stage('Release') {
      when { branch 'main' }
      stages {
        stage('Docker Build & Push') {
          steps {
            withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                              usernameVariable: 'DH_USER',
                                              passwordVariable: 'DH_PASS')]) {
              sh '''
                echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                docker buildx build --platform linux/arm64 \
                  -t $IMAGE:$TAG -t $IMAGE:latest --push .
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