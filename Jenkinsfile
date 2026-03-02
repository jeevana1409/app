pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    environment {
        SONARQUBE_SERVER = 'sonar-server'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        // ==============================
        // FEATURE BRANCH PIPELINE
        // ==============================

        stage('Build & Sonar (Feature Branch)') {
            when {
                expression { env.BRANCH_NAME.startsWith("feature") }
            }
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }

        stage('Quality Gate') {
            when {
                expression { env.BRANCH_NAME.startsWith("feature") }
            }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Create PR to Dev') {
            when {
                expression { env.BRANCH_NAME.startsWith("feature") }
            }
            steps {
                withCredentials([string(credentialsId: 'github-cred', variable: 'GITHUB_TOKEN')]) {
                    sh """
                    curl -X POST \
                    -H "Authorization: token \$GITHUB_TOKEN" \
                    -H "Accept: application/vnd.github.v3+json" \
                    https://api.github.com/repos/YOUR_USERNAME/YOUR_REPO/pulls \
                    -d '{
                        "title":"Auto PR from ${env.BRANCH_NAME}",
                        "head":"${env.BRANCH_NAME}",
                        "base":"dev",
                        "body":"Auto created PR after Quality Gate success"
                    }'
                    """
                }
            }
        }

    }   // ← closing stages block

}       // ← closing pipeline block
