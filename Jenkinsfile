pipeline {
  agent any
  triggers { pollSCM('* * * * *') }   // checks GitHub every minute; no inbound port needed
  environment {
    DOCKERHUB = credentials('dockerhub-creds')
    IMAGE_TAG = "${env.BUILD_NUMBER}"
  }
  stages {
    stage('Build Images') {
      steps {
        sh '''
          docker build -t $DOCKERHUB_USR/devops-backend:$IMAGE_TAG -t $DOCKERHUB_USR/devops-backend:latest ./backend
          docker build -t $DOCKERHUB_USR/devops-frontend:$IMAGE_TAG -t $DOCKERHUB_USR/devops-frontend:latest ./frontend
        '''
      }
    }
    stage('Push to Docker Hub') {
      steps {
        sh '''
          echo $DOCKERHUB_PSW | docker login -u $DOCKERHUB_USR --password-stdin
          docker push $DOCKERHUB_USR/devops-backend:$IMAGE_TAG
          docker push $DOCKERHUB_USR/devops-backend:latest
          docker push $DOCKERHUB_USR/devops-frontend:$IMAGE_TAG
          docker push $DOCKERHUB_USR/devops-frontend:latest
        '''
      }
    }
    stage('Deploy') {
      steps {
        sh '''
          cp docker-compose.yml /opt/devops-app/
          cd /opt/devops-app
          IMAGE_TAG=$IMAGE_TAG docker compose pull
          IMAGE_TAG=$IMAGE_TAG docker compose up -d --remove-orphans
          docker image prune -f
        '''
      }
    }
  }
  post { always { sh 'docker logout || true' } }
}
