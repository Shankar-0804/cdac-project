pipeline {
    agent any

    environment {
        IMAGE_NAME        = "shankar0804/flask-blog-app"
        IMAGE_TAG         = "${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = "flask-blog"
        SCANNER_HOME      = tool 'sonar-scanner'
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
                withSonarQubeEnv('sonar-server') {
                    sh """
                        ${SCANNER_HOME}/bin/sonar-scanner \
                        -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                        -Dsonar.sources=.
                    """
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('OWASP Dependency Check') {
            steps {
                dependencyCheck additionalArguments: '''
                    --scan .
                    --format HTML
                    --failOnCVSS 11
                ''',
                odcInstallation: 'dc'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                  trivy fs --severity HIGH,CRITICAL --exit-code 1 .
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh '''
                  docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Trivy Image Scan') {
            steps {
                sh '''
                  trivy image --severity HIGH,CRITICAL --exit-code 0 \
                  $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'docker-hub',
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh '''
                      echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                      docker push $IMAGE_NAME:$IMAGE_TAG
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                sh '''
                  echo "IMAGE_NAME=$IMAGE_NAME" > .env
                  echo "IMAGE_TAG=$IMAGE_TAG" >> .env

                  docker compose pull
                  docker compose up -d --force-recreate
                '''
            }
        }
    }
}
