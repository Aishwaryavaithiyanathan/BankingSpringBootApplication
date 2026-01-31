pipeline {
    agent any // run on your Linux agent

    environment {
        TAG = '3.0'
        CONTAINER_NAME = 'bankingapp'
        HTTP_PORT = '8989'
    }

    options {
        skipDefaultCheckout(true) // We'll explicitly checkout code
        timestamps()
    }

    stages {
        stage('Prepare Environment') {
            steps {
                echo 'Initializing Environment'
            }
        }

        stage('Code Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Maven Build') {
            steps {
                bat 'mvn clean package'
            }
        }

        stage('Docker Image Build') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                    bat """
                        docker build -t \$DOCKER_USER/\$CONTAINER_NAME:\$TAG --pull --no-cache .
                    """
                }
            }
        }

        /* ===================== SBOM STAGES START ===================== */
        stage('Generate SBOM') {
            steps {
                echo 'Generating SBOM using Trivy container'
                bat """
                    docker run --rm \
                      -v /var/run/docker.sock:/var/run/docker.sock \
                      -v \$WORKSPACE:/work \
                      aquasec/trivy:latest image \
                      --format cyclonedx \
                      --output /work/sbom.json \
                      \$DOCKER_USER/\$CONTAINER_NAME:\$TAG
                """
            }
        }

        stage('Scan SBOM (Security Gate)') {
            steps {
                echo 'Scanning SBOM for HIGH and CRITICAL vulnerabilities'
                bat """
                    docker run --rm \
                      -v \$WORKSPACE:/work \
                      aquasec/trivy:latest sbom \
                      /work/sbom.json \
                      --severity HIGH,CRITICAL \
                      --exit-code 1
                """
            }
        }

        stage('Archive SBOM') {
            steps {
                archiveArtifacts artifacts: 'sbom.json', fingerprint: true
            }
        }
        /* ===================== SBOM STAGES END ===================== */

        stage('Push Docker Image') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                    bat """
                        docker login -u \$DOCKER_USER -p \$DOCKER_PASSWORD
                        docker push \$DOCKER_USER/\$CONTAINER_NAME:\$TAG
                    """
                }
            }
        }

    }

    post {
        always {
            echo 'Cleaning up Docker images'
            bat "docker image rm \$DOCKER_USER/\$CONTAINER_NAME:\$TAG || true"
        }

        success {
            echo 'Pipeline completed successfully!'
        }

        failure {
            echo 'Pipeline failed!'
        }
    }
}
