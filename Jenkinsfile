pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        DOCKER_IMAGE = "sony9014/app"
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Auto Version Increment') {
            steps {
                script {

                    sh 'git fetch --tags'

                    def latestTag = sh(
                        script: "git describe --tags \$(git rev-list --tags --max-count=1) 2>/dev/null || echo v1.0.0",
                        returnStdout: true
                    ).trim()

                    echo "Latest Tag: ${latestTag}"

                    def version = latestTag.replace("v","").tokenize('.')
                    def major = version[0]
                    def minor = version[1]
                    def patch = version[2].toInteger() + 1

                    env.APP_VERSION = "v${major}.${minor}.${patch}"

                    echo "New Version: ${APP_VERSION}"

                    withCredentials([usernamePassword(
                        credentialsId: 'github-api-creds',
                        usernameVariable: 'GIT_USERNAME',
                        passwordVariable: 'GIT_PASSWORD'
                    )]) {

                        sh """
                        git config user.name "jenkins"
                        git config user.email "jenkins@local"

                        git tag ${APP_VERSION}

                        git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/jeevana1409/app.git ${APP_VERSION}
                        """
                    }
                }
            }
        }

        stage('Update Maven Version') {
            steps {
                sh "mvn versions:set -DnewVersion=${APP_VERSION}"
            }
        }

        stage('Build WAR') {
            steps {
                sh "mvn clean package -DskipTests"
            }
        }

        stage('Run Tests') {
            steps {
                sh "mvn test"
            }
        }

        stage('Upload Artifact to Nexus') {
            steps {
                sh "mvn deploy -DskipTests"
            }
        }

        stage('Security Scan') {
            steps {                                          
                sh "trivy fs --severity HIGH,CRITICAL --exit-code 1 ."
            }
        }

        stage('Docker Build & Push') {
            steps {
                script {

                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {

                        sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin

                        docker build -t ${DOCKER_IMAGE}:${APP_VERSION} .

                        docker push ${DOCKER_IMAGE}:${APP_VERSION}

                        docker logout
                        """
                    }
                }
            }
        }

        stage('Deploy to Dev') {
            steps {
                script {

                    sshagent(credentials: ['dev-ssh']) {

                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@16.176.135.187 "
                        docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                        docker stop app || true &&
                        docker rm app || true &&
                        docker run -d -p 8080:8080 --name app ${DOCKER_IMAGE}:${APP_VERSION}
                        "
                        """
                    }
                }
            }
        }

        stage('Deploy to QA') {

            steps {

                input message: "Deploy to QA?"

                script {

                    sshagent(credentials: ['docker-server-ssh']) {

                        sh """
                        ssh -o StrictHostKeyChecking=no ec2-user@<QA_PUBLIC_IP> "
                        docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                        docker stop qa-app || true &&
                        docker rm qa-app || true &&
                        docker run -d -p 8080:8080 --name qa-app ${DOCKER_IMAGE}:${APP_VERSION}
                        "
                        """
                    }
                }
            }
        }
    }

    post {
        success {
            echo "✅ Pipeline Completed Successfully"
        }

        failure {
            echo "❌ Pipeline Failed"
        }
    }
}
