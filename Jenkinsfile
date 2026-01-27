pipeline {
    agent none

    environment {
        IMAGE_NAME        = "shankar0804/flask-blog-app"
        IMAGE_TAG         = "${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = "flask-blog"
        SCANNER_HOME      = tool 'sonar-scanner'
    }

    stages {

        stage('Checkout') {
            agent { label 'security-agent' }
            steps {
                git branch: 'main',
                    url: 'https://github.com/Shankar-0804/cdac-project.git'
                stash name: 'source-code', includes: '**'
            }
        }

        stage('SonarQube Analysis') {
            agent { label 'security-agent' }
            steps {
                unstash 'source-code'
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
            agent { label 'security-agent' }
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('OWASP Dependency Check') {
            agent { label 'master' }  // <-- now runs on master
            steps {
                unstash 'source-code'
                dependencyCheck additionalArguments: '''
                    --scan .
                    --format HTML
                    --failOnCVSS 11
                ''', odcInstallation: 'dc'

                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
            }
        }

        stage('Trivy FS Scan') {
            agent { label 'security-agent' }
            steps {
                unstash 'source-code'
                sh '''
                  trivy fs --severity HIGH,CRITICAL --exit-code 1 .
                '''
            }
        }

        stage('Build Image') {
            agent { label 'docker-agent' }
            steps {
                unstash 'source-code'
                sh '''
                  docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Trivy Image Scan') {
            agent { label 'security-agent' }
            steps {
                sh '''
                  trivy image --severity HIGH,CRITICAL --exit-code 0 \
                  $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Push Image') {
            agent { label 'docker-agent' }
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
            agent { label 'docker-agent' }
            steps {
                unstash 'source-code'
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
