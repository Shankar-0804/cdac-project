pipeline {
    agent any

    environment {
        IMAGE_NAME        = "shankar0804/flask-blog"
        IMAGE_TAG         = "latest"
        SONAR_PROJECT_KEY = "flask-blog"
    }

    stages {

        stage('Checkout') {
            steps {
                git branch: 'main',
                    url: 'https://github.com/Shankar-0804/cdac-project.git'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('Sonar') {
                    sh """
                        $SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectKey=$SONAR_PROJECT_KEY \
                        -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                      docker push $IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                  docker compose pull
                  docker compose up -d --force-recreate
                '''
            }
        }
    }
}
