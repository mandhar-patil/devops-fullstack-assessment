pipeline {
    agent any

    environment {
        DOCKERHUB_USER = 'mandhar12'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
    }

    triggers { pollSCM('H/2 * * * *') }   // port 8080 is closed to the internet, so we poll instead of using a webhook

    stages {
        stage('Checkout') {
            steps { checkout scm }
        }

        stage('Build Images') {
            steps {
                sh '''
                  docker build -t $DOCKERHUB_USER/devops-backend:$IMAGE_TAG -t $DOCKERHUB_USER/devops-backend:latest ./backend
                  docker build -t $DOCKERHUB_USER/devops-frontend:$IMAGE_TAG -t $DOCKERHUB_USER/devops-frontend:latest ./frontend
                '''
            }
        }

        stage('Push Images') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                      echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                      docker push $DOCKERHUB_USER/devops-backend:$IMAGE_TAG
                      docker push $DOCKERHUB_USER/devops-backend:latest
                      docker push $DOCKERHUB_USER/devops-frontend:$IMAGE_TAG
                      docker push $DOCKERHUB_USER/devops-frontend:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                withCredentials([string(credentialsId: 'django-secret-key', variable: 'DJANGO_SECRET_KEY')]) {
                    sh '''
                      export DOCKERHUB_USER=$DOCKERHUB_USER
                      export DEBUG=False
                      export ALLOWED_HOSTS='*'
                      export API_URL=/api
                      docker compose pull
                      docker compose up -d --remove-orphans
                      docker image prune -f
                    '''
                }
            }
        }
    }

    post {
        always { sh 'docker logout || true' }
    }
}
