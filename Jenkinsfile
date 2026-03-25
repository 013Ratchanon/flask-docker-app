// =================================================================
// HELPER FUNCTION: ส่ง Notification ไปยัง n8n (ตามแนวทางของ Express Pipeline)
// =================================================================

pipeline {
    // ใช้ agent any เพราะ build จะทำงานบน Jenkins controller/agent (Linux)
    agent any

    // กันเช็คเอาต์ซ้ำซ้อน (ตามแนวทาง Express)
    options {
        skipDefaultCheckout(true)
    }

    // Environment variables
    environment {
        DOCKER_HUB_CREDENTIALS_ID = 'docker-jenkins'
        DOCKER_REPO               = "013ratchanon/flask-docker-app"

        // จำลอง DEV/PROD บน Local
        DEV_APP_NAME              = "flask-app-dev"
        DEV_HOST_PORT             = "5001"
        PROD_APP_NAME             = "flask-app-prod"
        PROD_HOST_PORT            = "5000"
    }

    // Input parameters (Build & Deploy หรือ Rollback)
    parameters {
        choice(name: 'ACTION', choices: ['Build & Deploy', 'Rollback'], description: 'เลือก Action ที่ต้องการ')
        string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'สำหรับ Rollback: ใส่ Image Tag (เช่น Git Hash หรือ dev-123)')
        choice(name: 'ROLLBACK_TARGET', choices: ['dev', 'prod'], description: 'สำหรับ Rollback: เลือก Environment')
    }

    stages {
        // Stage 1: Checkout
        stage('Checkout') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                echo "Checking out code..."
                checkout scm
            }
        }

        // Stage 2: Install & Test (ใช้ Python container เหมือนแนวคิด Express/Node test)
        stage('Install & Test') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                echo "Running tests inside a consistent Docker environment..."
                script {
                    docker.image('python:3.13-slim').inside {
                        sh '''
                            pip install --no-cache-dir -r requirements.txt
                            pytest -v --tb=short --junitxml=test-results.xml
                        '''
                    }
                }
            }
            post {
                always {
                    junit 'test-results.xml'
                }
            }
        }

        // Stage 3: Build & Push Docker Image (Push latest เฉพาะ main)
        stage('Build & Push Docker Image') {
            steps {
                script {
                                sh 'whoami'
            sh 'docker version'
            sh 'docker ps'
                    def imageTag = (env.BRANCH_NAME == 'main') ? sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim() : "dev-${env.BUILD_NUMBER}"
                    env.IMAGE_TAG = imageTag

                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_HUB_CREDENTIALS_ID) {
                        echo "Building image: ${DOCKER_REPO}:${env.IMAGE_TAG}"
                        def customImage = docker.build("${DOCKER_REPO}:${env.IMAGE_TAG}")

                        echo "Pushing images to Docker Hub..."
                        customImage.push()
                        if (env.BRANCH_NAME == 'main') {
                            customImage.push('latest')
                        }
                    }
                }
            }
        }

        // Approval ก่อน Deploy ไป PROD
        stage('Approval for Production') {
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Deploy image tag '${env.IMAGE_TAG}' to PRODUCTION (Local Docker on port ${PROD_HOST_PORT})?"
                }
            }
        }

        // Deploy to PROD (Local Docker) — สำหรับ branch main
        stage('Deploy to PRODUCTION (Local Docker)') {
            steps {
                script {
                    def deployCmd = """
                            echo "Deploying container ${PROD_APP_NAME} from latest image..."
                            docker pull ${DOCKER_REPO}:${env.IMAGE_TAG}
                            docker stop ${PROD_APP_NAME} || true
                            docker rm ${PROD_APP_NAME} || true
                            docker run -d --name ${PROD_APP_NAME} -p ${PROD_HOST_PORT}:5000 ${DOCKER_REPO}:${env.IMAGE_TAG}
                            docker ps --filter name=${PROD_APP_NAME} --format "table {{.Names}}\\t{{.Image}}\\t{{.Status}}"
                        """
                    sh deployCmd
                }
            }
        }

        // Rollback เมื่อเลือก ACTION = Rollback
        stage('Execute Rollback') {
            when { expression { params.ACTION == 'Rollback' } } 
            steps {
                script {
                    if (params.ROLLBACK_TAG.trim().isEmpty()) {
                        error "เมื่อเลือก Rollback กรุณาระบุ 'ROLLBACK_TAG'"
                    }

                    // ใช้ PROD_APP_NAME และ PROD_HOST_PORT ตามที่คุณตั้งใน env
                    def targetApp = env.PROD_APP_NAME
                    def targetPort = env.PROD_HOST_PORT
                    def imageToDeploy = "${DOCKER_REPO}:${params.ROLLBACK_TAG.trim()}"

                    echo "ROLLING BACK to image: ${imageToDeploy}"

                    sh """
                        docker pull ${imageToDeploy}
                        docker stop ${targetApp} || true
                        docker rm ${targetApp} || true
                        docker run -d --name ${targetApp} -p ${targetPort}:5000 ${imageToDeploy}
                    """
                    sendNotificationToN8n('success', "Rollback Executed", params.ROLLBACK_TAG, targetApp, targetPort)
                }
            }
        }
    }

    // Post actions
    post {
        always {
            script {
                if (params.ACTION == 'Build & Deploy') {
                    echo "Cleaning up Docker images on agent..."
                    try {
                        sh """
                            docker image rm -f ${DOCKER_REPO}:${env.IMAGE_TAG} || true
                            docker image rm -f ${DOCKER_REPO}:latest || true
                        """
                    } catch (err) {
                        echo "Could not clean up images, but continuing..."
                    }
                }
                // ส่วนของการลบ Workspace
                echo "Cleaning up workspace..."
                cleanWs()
            }
        }
    }
}