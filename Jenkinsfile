pipeline {
    agent any

    environment {
        IMAGE_NAME        = "shankar0804/flask-blog"
        IMAGE_TAG         = "latest"
        SONAR_PROJECT_KEY = "flask-blog"
        SCANNER_HOME      = tool 'Sonar'
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
                    waitForQualityGate abortPipeline: true
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
                echo "Running Trivy filesystem scan..."
                sh '''
                  trivy fs \
                  --severity HIGH,CRITICAL \
                  --exit-code 1 \
                  .
                '''
            }
        }

        stage('Build Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME:$IMAGE_TAG .'
            }
        }

        stage('Trivy Image Scan') {
            steps {
                echo "Running Trivy Docker image scan..."
                sh '''
                  trivy image \
                  --severity HIGH,CRITICAL \
                  --exit-code 0 \
                  $IMAGE_NAME:$IMAGE_TAG
                '''
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

