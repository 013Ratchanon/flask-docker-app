// =================================================================
// HELPER FUNCTION: ส่ง Notification ไปยัง n8n
// =================================================================
def sendNotificationToN8n(status, stageName, tag, app, port) {
    sh """
        curl -X POST http://your-n8n-webhook-url \
        -H 'Content-Type: application/json' \
        -d '{
            "status": "${status}",
            "stage": "${stageName}",
            "tag": "${tag}",
            "app": "${app}",
            "port": "${port}"
        }'
    """
}

pipeline {
    agent any

    options {
        skipDefaultCheckout(true)
        timeout(time: 30, unit: 'MINUTES')
    }

    environment {
        DOCKER_HUB_CREDENTIALS_ID = 'docker-jenkins'
        DOCKER_REPO               = "013ratchanon/flask-docker-app"

        DEV_APP_NAME              = "flask-app-dev"
        DEV_HOST_PORT             = "5001"
        PROD_APP_NAME             = "flask-app-prod"
        PROD_HOST_PORT            = "3000"
    }

    parameters {
        choice(name: 'ACTION', choices: ['Build & Deploy', 'Rollback'], description: 'เลือก Action')
        string(name: 'ROLLBACK_TAG', defaultValue: '', description: 'Image Tag สำหรับ Rollback')
        choice(name: 'ROLLBACK_TARGET', choices: ['dev', 'prod'], description: 'เลือก Environment')
    }

    stages {

        stage('Checkout') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                checkout scm
            }
        }

        stage('Install & Test') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
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

        stage('Build & Push Docker Image') {
            when { expression { params.ACTION == 'Build & Deploy' } }
            steps {
                script {
                    IMAGE_TAG = (env.BRANCH_NAME == 'main') ?
                        sh(script: 'git rev-parse --short HEAD', returnStdout: true).trim()
                        : "dev-${env.BUILD_NUMBER}"

                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_HUB_CREDENTIALS_ID) {
                        def image = docker.build("${DOCKER_REPO}:${IMAGE_TAG}")
                        image.push()

                        if (env.BRANCH_NAME == 'main') {
                            image.push('latest')
                        }
                    }
                }
            }
        }

        // ✅ DEV ใช้ develop branch
        stage('Deploy to DEV') {
            when {
                allOf {
                    expression { params.ACTION == 'Build & Deploy' }
                    branch 'develop'
                }
            }
            steps {
                script {
                    sh """
                        set -e
                        docker pull ${DOCKER_REPO}:${IMAGE_TAG}
                        docker stop ${DEV_APP_NAME} || true
                        docker rm ${DEV_APP_NAME} || true
                        docker run -d --name ${DEV_APP_NAME} -p ${DEV_HOST_PORT}:5000 ${DOCKER_REPO}:${IMAGE_TAG}
                    """
                }
            }
            post {
                success {
                    sendNotificationToN8n('success', 'DEV Deploy', IMAGE_TAG, DEV_APP_NAME, DEV_HOST_PORT)
                }
            }
        }

        stage('Approval for Production') {
            when {
                allOf {
                    expression { params.ACTION == 'Build & Deploy' }
                    branch 'main'
                }
            }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Deploy ${IMAGE_TAG} to PRODUCTION?"
                }
            }
        }

        stage('Deploy to PROD') {
            when {
                allOf {
                    expression { params.ACTION == 'Build & Deploy' }
                    branch 'main'
                }
            }
            steps {
                script {
                    sh """
                        set -e
                        docker pull ${DOCKER_REPO}:${IMAGE_TAG}
                        docker stop ${PROD_APP_NAME} || true
                        docker rm ${PROD_APP_NAME} || true
                        docker run -d --name ${PROD_APP_NAME} -p ${PROD_HOST_PORT}:5000 ${DOCKER_REPO}:${IMAGE_TAG}
                    """
                }
            }
            post {
                success {
                    sendNotificationToN8n('success', 'PROD Deploy', IMAGE_TAG, PROD_APP_NAME, PROD_HOST_PORT)
                }
            }
        }

        stage('Rollback') {
            when { expression { params.ACTION == 'Rollback' } }
            steps {
                script {
                    if (!params.ROLLBACK_TAG?.trim()) {
                        error "กรุณาระบุ ROLLBACK_TAG"
                    }

                    def targetApp  = (params.ROLLBACK_TARGET == 'dev') ? DEV_APP_NAME  : PROD_APP_NAME
                    def targetPort = (params.ROLLBACK_TARGET == 'dev') ? DEV_HOST_PORT : PROD_HOST_PORT
                    def image      = "${DOCKER_REPO}:${params.ROLLBACK_TAG}"

                    sh """
                        set -e
                        docker pull ${image}
                        docker stop ${targetApp} || true
                        docker rm ${targetApp} || true
                        docker run -d --name ${targetApp} -p ${targetPort}:5000 ${image}
                    """

                    sendNotificationToN8n('success', 'Rollback', params.ROLLBACK_TAG, targetApp, targetPort)
                }
            }
        }
    }

    post {
        always {
            script {
                if (params.ACTION == 'Build & Deploy' && IMAGE_TAG) {
                    sh """
                        docker image rm -f ${DOCKER_REPO}:${IMAGE_TAG} || true
                        docker image rm -f ${DOCKER_REPO}:latest || true
                    """
                }
                cleanWs()
            }
        }
    }
}