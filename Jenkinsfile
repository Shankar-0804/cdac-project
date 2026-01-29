pipeline {
    agent any

    environment {
        IMAGE_NAME        = "shankar0804/flask-blog-app"
        IMAGE_TAG         = "${BUILD_NUMBER}"
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
            steps {
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Trivy FS Scan') {
            steps {
                sh '''
                    docker run --rm \
                      -v $(pwd):/project \
                      aquasec/trivy:latest fs \
                      --severity HIGH,CRITICAL \
                      --exit-code 1 \
                      /project
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
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      aquasec/trivy:latest image \
                      --severity HIGH,CRITICAL \
                      --exit-code 0 \
                      $IMAGE_NAME:$IMAGE_TAG
                '''
            }
        }

        stage('Push Image') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'dockerhub-cred',
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
            agent { label 'docker-ec2' }
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
