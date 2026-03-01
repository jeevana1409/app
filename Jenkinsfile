pipeline {
    agent any

    tools {
        maven 'Maven'
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Initialization - Tag Check') {
            when {
                changeRequest()
            }
            steps {
                script {
                    echo "Checking tag policy..."

                    def tag = sh(script: "git tag --points-at HEAD", returnStdout: true).trim()

                    if (!tag) {
                        error("❌ No tag found on this commit. PR blocked.")
                    }

                    echo "✅ Tag found: ${tag}"
                }
            }
        }

        stage('Build & SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh 'mvn clean verify sonar:sonar'
                }
            }
        }

        stage("Quality Gate") {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Create Pull Request to Dev') {
            when {
                allOf {
                    not { changeRequest() }
                    not { branch 'dev' }
                }
            }
            steps {
                echo "Branch is: ${env.BRANCH_NAME}"

                withCredentials([string(credentialsId: 'github-cred', variable: 'GITHUB_TOKEN')]) {
                    sh """
                        curl -X POST https://api.github.com/repos/jeevana1409/app/pulls \
                        -H "Authorization: token \$GITHUB_TOKEN" \
                        -H "Accept: application/vnd.github.v3+json" \
                        -d '{
                            "title": "Auto PR: ${env.BRANCH_NAME} → dev",
                            "head": "${env.BRANCH_NAME}",
                            "base": "dev",
                            "body": "Automatically created after successful SonarQube Quality Gate."
                        }'
                    """
                }
            }
        }
    }
}
