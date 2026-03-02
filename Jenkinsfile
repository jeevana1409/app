pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        DOCKER_IMAGE = "sony9014/app"
        DEV_SERVER   = "ec2-user@3.27.63.129"
    }

    stages {

        stage('Checkout') {
            when {
                branch 'dev'
            }
            steps {
                checkout scm
            }
        }
        stage('Initialization - Version Check') {
    when {
        branch 'dev'
    }
    steps {
        script {
            echo "Fetching tags..."
            sh 'git fetch --tags'

            def tag = sh(
                script: "git describe --tags --abbrev=0 || echo 'NO_TAG'",
                returnStdout: true
            ).trim()

            if (tag == "NO_TAG") {
                error "❌ No Git tag found."
            }

            env.APP_VERSION = tag
            echo "Using version: ${env.APP_VERSION}"
        }
    }
}

        stage('Build') {
            when {
                branch 'dev'
            }
            steps {
                sh 'mvn clean package'
            }
        }

        stage('Test') {
            when {
                branch 'dev'
            }
            steps {
                sh 'mvn test'
            }
        }

        stage('Docker Build & Push') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    withCredentials([usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )]) {

                        sh '''
                            echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                            docker build -t $DOCKER_IMAGE:${APP_VERSION} .
                            docker push $DOCKER_IMAGE:${APP_VERSION}
                        '''
                    }
                }
            }
        }

        stage('Deploy to Dev Server') {
            when {
                branch 'dev'
            }
            steps {
                script {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${DEV_SERVER} '
                        docker pull ${DOCKER_IMAGE}:${APP_VERSION} &&
                        docker stop app || true &&
                        docker rm app || true &&
                        docker run -d -p 8080:8080 --name app ${DOCKER_IMAGE}:${APP_VERSION}
                        '
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "✅ Deployment to DEV server successful!"
        }
        failure {
            echo "❌ DEV deployment failed!"
        }
    }
}
