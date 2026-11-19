pipeline {
  agent any
  tools { nodejs 'node22' }

  environment {
    IMAGE    = 'YOUR_DOCKERHUB_USER/jenkins-demo'
    TAG      = "${BUILD_NUMBER}"
    APP_HOST = 'ubuntu@<EC2_PUBLIC_IP>'
    APP_URL  = 'http://<EC2_PUBLIC_IP>:3000'
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
        sh 'docker build --platform linux/amd64 -t $IMAGE:$TAG -t $IMAGE:latest .'
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

  post {
    always {
      sh 'docker logout || true'
      cleanWs()
    }
  }
}