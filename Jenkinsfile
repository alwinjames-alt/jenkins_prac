def TARGETS = [
  dev:  [host: 'ec2-user@<DEV_IP>',  url: 'http://<DEV_IP>:3000',  cred: 'ec2-app-ssh', tagPrefix: 'dev-', envTag: 'dev'],
  main: [host: 'ec2-user@<PROD_IP>', url: 'http://<PROD_IP>:3000', cred: 'ec2-app-ssh', tagPrefix: '',     envTag: 'latest'],
]

pipeline {
  agent any
  tools { nodejs 'node22' }

  parameters {
    string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'Existing image tag to redeploy (leave empty for a normal build)')
  }

  environment {
    IMAGE = 'alwinjames313/jenkins_prac'
  }

  stages {
    stage('Install') {
      when { expression { !params.ROLLBACK_TAG } }
      steps { sh 'npm ci' }
    }

    stage('Test') {
      when { expression { !params.ROLLBACK_TAG } }
      steps { sh 'npm test' }
    }

    stage('Release') {
      when { expression { TARGETS.containsKey(env.BRANCH_NAME) } }
      stages {
        stage('Resolve Target') {
          steps {
            script {
              def t = TARGETS[env.BRANCH_NAME]
              env.APP_HOST = t.host
              env.APP_URL  = t.url
              env.SSH_CRED = t.cred
              env.ENV_TAG  = t.envTag
              env.TAG      = params.ROLLBACK_TAG ? params.ROLLBACK_TAG : "${t.tagPrefix}${env.BUILD_NUMBER}"
              echo "Deploying ${env.IMAGE}:${env.TAG} to ${env.APP_HOST}"
            }
          }
        }

        stage('Docker Build & Push') {
          when { expression { !params.ROLLBACK_TAG } }
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
            sshagent([env.SSH_CRED]) {
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