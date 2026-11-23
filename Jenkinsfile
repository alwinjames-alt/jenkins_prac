pipeline {
  agent any
  tools { nodejs 'node22' }

  environment {
    IMAGE = 'alwinjames313/jenkins_prac'
    TAG   = "${BUILD_NUMBER}"
  }

  stages {
    stage('Install') {
      steps { sh 'npm ci' }
    }

    stage('Test') {
      steps { sh 'npm test' }
    }

    stage('Docker Build') {
      steps {
        sh 'docker build --platform linux/arm64 -t $IMAGE:$TAG -t $IMAGE:latest .'
      }
    }

    stage('Docker Push') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                                          usernameVariable: 'DH_USER',
                                          passwordVariable: 'DH_PASS')]) {
          sh '''
            echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
            docker push $IMAGE:$TAG
            docker push $IMAGE:latest
          '''
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