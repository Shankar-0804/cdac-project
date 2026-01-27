pipeline {
    agent none

    environment {
        IMAGE_NAME        = "shankar0804/flask-blog-app"
        IMAGE_TAG         = "${BUILD_NUMBER}"
        SONAR_PROJECT_KEY = "flask-blog"
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
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('sonar-server') {
                        sh """
                        ${scannerHome}/bin/sonar-scanner \
                          -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                          -Dsonar.sources=.
                        """
                    }
                }
            }
        }

        stage('OWASP Dependency Check') {
            agent { label 'security-agent' }
            steps {
                unstash 'source-code'
                withCredentials([string(
                    credentialsId: 'nvd-api-key',
                    variable: 'NVD_API_KEY'
                )]) {
                    sh '''
                    dependency-check.sh \
                      --project "flask-blog" \
                      --scan . \
                      --format HTML \
                      --out dependency-check-report \
                      --nvdApiKey $NVD_API_KEY \
                      --failOnCVSS 7
                    '''
                }
                archiveArtifacts artifacts: 'dependency-check-report/**', fingerprint: true
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
                trivy image --severity HIGH,CRITICAL --exit-code 0 $IMAGE_NAME:$IMAGE_TAG
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
