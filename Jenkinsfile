
pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS_ID = 'roseaw-dockerhub'
        DOCKER_IMAGE = 'cithit/tchitocn'
        IMAGE_TAG = "build-${BUILD_NUMBER}"
        GITHUB_URL = 'https://github.com/tchitocn-miamioh/225-lab3-6.git'
        KUBECONFIG = credentials('tchitocn-225')
        DEV_URL = 'http://10.43.76.175'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: '*/main']],
                    userRemoteConfigs: [[url: "${GITHUB_URL}"]]
                ])
            }
        }

        stage('Lint HTML') {
            steps {
                sh 'npm install htmlhint --save-dev'
                sh 'npx htmlhint *.html'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    docker.withRegistry(
                        'https://registry-1.docker.io',
                        "${DOCKER_CREDENTIALS_ID}"
                    ) {
                        docker.build(
                            "${DOCKER_IMAGE}:${IMAGE_TAG}",
                            "-f Dockerfile.build ."
                        )
                    }
                }
            }
        }

        stage('Push Docker Image') {
            steps {
                script {
                    docker.withRegistry(
                        'https://index.docker.io/v1/',
                        "${DOCKER_CREDENTIALS_ID}"
                    ) {
                        docker.image("${DOCKER_IMAGE}:${IMAGE_TAG}").push()
                    }
                }
            }
        }

        stage('Deploy to Dev Environment') {
            steps {
                script {
                    // Update the DEV deployment image tag
                    sh """
                        sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-dev.yaml
                    """

                    // Deploy to Kubernetes DEV
                    sh 'kubectl apply -f deployment-dev.yaml'

                    // Wait for the DEV deployment to become ready
                    sh 'kubectl rollout status deployment/dev-deployment --timeout=120s'
                }
            }
        }

        stage('Check DEV Connectivity') {
            steps {
                script {
                    echo 'Checking DEV service connectivity from Jenkins...'

                    // Diagnostic only: do not fail pipeline here
                    sh '''
                        curl -v --connect-timeout 5 --max-time 10 \
                        "$DEV_URL/" || true
                    '''

                    echo 'Checking DEV service connectivity from Docker...'

                    // Test network access from a temporary container
                    sh '''
                        docker run --rm \
                        -e DEV_URL="$DEV_URL" \
                        curlimages/curl:latest \
                        -v --connect-timeout 5 --max-time 10 \
                        "$DEV_URL/" || true
                    '''
                }
            }
        }

        stage('Run Acceptance Tests') {
            steps {
                script {
                    echo 'Building Selenium test image...'

                    sh 'docker build -t qa-tests -f Dockerfile.test .'

                    echo 'Running Selenium acceptance tests...'

                    // Test must pass before deploying to PROD
                    sh 'docker run --rm qa-tests'
                }
            }
        }

        stage('Deploy to Prod Environment') {
            steps {
                script {
                    // Update the PROD deployment image tag
                    sh """
                        sed -i 's|${DOCKER_IMAGE}:latest|${DOCKER_IMAGE}:${IMAGE_TAG}|' deployment-prod.yaml
                    """

                    // Deploy to Kubernetes PROD
                    sh 'kubectl apply -f deployment-prod.yaml'

                    // Wait for the PROD deployment to become ready
                    sh 'kubectl rollout status deployment/prod-deployment --timeout=120s'
                }
            }
        }

        stage('Check Kubernetes Cluster') {
            steps {
                script {
                    sh 'kubectl get deployments'
                    sh 'kubectl get pods'
                    sh 'kubectl get services'
                    sh 'kubectl get ingress'
                }
            }
        }
    }

    post {
        success {
            slackSend color: "good",
                message: "Build Completed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }

        unstable {
            slackSend color: "warning",
                message: "Build Unstable: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }

        failure {
            slackSend color: "danger",
                message: "Build Failed: ${env.JOB_NAME} ${env.BUILD_NUMBER}"
        }
    }
}
