pipeline {
    agent any

    environment {
        DOCKERHUB_CREDENTIALS = credentials('dockerhub-credentials')
        DOCKERHUB_USERNAME    = '12326280'
        IMAGE_NAME            = 'plant-disease-xai'
        IMAGE_TAG             = "${env.BUILD_NUMBER}"
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '5'))
        timestamps()
        timeout(time: 30, unit: 'MINUTES')
    }

    stages {

        stage('Clone Repository') {
            steps {
                echo '=== Stage 1: Cloning GitHub Repository ==='
                checkout scm
            }
        }

        stage('Docker Build') {
            steps {
                echo '=== Stage 2: Building Docker Image ==='
                script {
                    dockerImage = docker.build("${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}")
                }
                echo "Built image: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
            }
        }

        stage('Smoke Test') {
            steps {
                echo '=== Stage 3: Running Smoke Test ==='
                script {
                    // Run container in background
                    def container = dockerImage.run("-d -p 9090:8000 --name smoke-test-${env.BUILD_NUMBER}")
                    try {
                        // Wait for the app to start
                        sleep(time: 60, unit: 'SECONDS')

                        // Health check — test /api/models endpoint
                        if (isUnix()) {
                            sh 'curl -f http://localhost:9090/api/models || exit 1'
                        } else {
                            powershell 'Invoke-RestMethod http://localhost:9090/api/models'
                        }
                        echo 'Smoke test PASSED — API is responding'
                    } finally {
                        // Always clean up the test container
                        container.stop()
                    }
                }
            }
        }

        stage('Push to DockerHub') {
            steps {
                echo '=== Stage 4: Pushing Image to DockerHub ==='
                script {
                    docker.withRegistry('https://index.docker.io/v1/', 'dockerhub-credentials') {
                        dockerImage.push("${IMAGE_TAG}")
                        dockerImage.push('latest')
                    }
                }
                echo "Pushed: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"
                echo "Pushed: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:latest"
            }
        }

        stage('Cleanup') {
            steps {
                echo '=== Stage 5: Cleaning Up Local Images ==='
                script {
                    if (isUnix()) {
                        sh "docker rmi ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} || true"
                    } else {
                        powershell "docker rmi ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} -ErrorAction SilentlyContinue"
                    }
                }
            }
        }
    }

    post {
        success {
            echo """
            =========================================
            ✅ PIPELINE SUCCESS
            Image: ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}
            Build: #${env.BUILD_NUMBER}
            =========================================
            """
        }
        failure {
            echo """
            =========================================
            ❌ PIPELINE FAILED
            Build: #${env.BUILD_NUMBER}
            Check console output for details.
            =========================================
            """
        }
        always {
            echo 'Pipeline execution completed.'
            cleanWs()
        }
    }
}
