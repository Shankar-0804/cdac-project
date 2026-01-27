pipeline {
    agent none  // All stages specify their own agent

    environment {
        IMAGE_NAME        = "shankar0804/flask-blog-app"
        IMAGE_TAG         = "${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = "flask-blog"
    }

    stages {

        stage('Checkout') {
            agent any
            steps {
                git branch: 'main',
                    url: 'https://github.com/Shankar-0804/cdac-project.git'
            }
        }

        stage('SonarQube Analysis') {
            agent { label 'security-agent' }
            steps {
                withSonarQubeEnv('sonar-server') {
                    script {
                        def scannerHome = tool 'sonar-scanner'
                        sh """
                            ${scannerHome}/bin/sonar-scanner \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('Quality Gate') {
            agent { label 'security-agent' }
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('OWASP Dependency Check') {
            agent { label '' }  
            steps {
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: """
                        --scan .
                        --format HTML
                        --failOnCVSS 11
                        --nvdApiKey $NVD_API_KEY
                    """,
                    odcInstallation: 'dc'

                    dependencyCheckPublisher pattern: '**/dependency-check-report.xml'
                }
            }
        }

        stage('Trivy FS Scan') {
            agent { label 'security-agent' }
            steps {
                sh '''
                  trivy fs --severity HIGH,CRITICAL --exit-code 1 .
                '''
            }
        }

        stage('Build Image') {
            agent { label 'docker-agent' }
            steps {
                sh '''
                  docker build -t $IMAGE_NAME:$IMAGE_TAG .
                '''
            }
        }

        stage('Trivy Image Scan') {
            agent { label 'docker-agent' }
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
