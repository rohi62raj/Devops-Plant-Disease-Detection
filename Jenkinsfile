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
                    def containerName = "smoke-test-${env.BUILD_NUMBER}"

                    // Run container (no port mapping needed — we use container IP directly)
                    sh "docker run -d --name ${containerName} ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG}"

                    try {
                        // Wait for PyTorch models to load (~90 seconds)
                        sleep(time: 90, unit: 'SECONDS')

                        // Get the container IP on the Docker bridge network
                        def containerIp = sh(
                            script: "docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' ${containerName}",
                            returnStdout: true
                        ).trim()

                        echo "Smoke test container IP: ${containerIp}"

                        // Health check — test /api/models endpoint via container IP
                        sh "curl -f --max-time 15 http://${containerIp}:8000/api/models"
                        echo 'Smoke test PASSED — API is responding'
                    } finally {
                        // Always clean up the test container
                        sh "docker stop ${containerName} || true"
                        sh "docker rm ${containerName} || true"
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
                sh "docker rmi ${DOCKERHUB_USERNAME}/${IMAGE_NAME}:${IMAGE_TAG} || true"
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
