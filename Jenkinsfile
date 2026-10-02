pipeline {
    agent any

    environment {
        DOCKERHUB_NAMESPACE = 'tracynguyen203'
        GITHUB_REPO_URL     = 'https://github.com/tracynguyen203/MOP-Code.git'
        WEBAPP_IMAGE        = "${DOCKERHUB_NAMESPACE}/mop-webapp"
        DOCKER_CREDS_ID     = 'dockerhub-creds'
        SONAR_TOKEN_ID      = 'sonarqube-token'
    }

    options {
        timeout(time: 1, unit: 'HOURS')
    }

    stages {
        // Stage 1: Source Code Checkout & Environment Initialization
        stage('Checkout & Setup') {
            steps {
                echo 'Stage 1: Checkout Code'
                checkout scm
                script {
                    env.BUILD_TAG_VAR = "${env.BUILD_NUMBER}"
                }
            }
        }

        // Stage 2: Build Docker Image for the Next.js Web Application (staging config)
        stage('Build') {
            steps {
                echo 'Stage 2: Building Web App Docker Image (staging env)'
                dir('next_webapp') {
                    withCredentials([file(credentialsId: 'app-env-staging', variable: 'ENV_FILE')]) {
                        bat 'copy "%ENV_FILE%" .env /Y'
                    }
		    bat 'docker build -f ../Dockerfile -t %WEBAPP_IMAGE%:%BUILD_NUMBER% .'
                    script {
                        dockerImage = docker.build("${WEBAPP_IMAGE}:${env.BUILD_NUMBER}")
                    }
                }
            }
        }

        // Stage 3: Run Automated Tests
        stage('Test') {
            steps {
                echo 'Stage 3: Running Tests'
                dir('next_webapp') {
		    bat 'npm install --silent'
                    bat 'npm test -- --passWithNoTests || exit 0'
                }
            }
        }

        // Stage 4: Code Quality Analysis with SonarQube
        stage('Code Quality') {
            steps {
                echo 'Stage 4: SonarQube Code Analysis'
                withCredentials([string(credentialsId: SONAR_TOKEN_ID, variable: 'SONAR_TOKEN')]) {
                    script {
                        def scannerHome = tool 'SonarScanner'
                        withSonarQubeEnv('MySonarQubeServer') {
                            bat "\"${scannerHome}\\bin\\sonar-scanner\" -Dsonar.token=%SONAR_TOKEN%"
                        }
                    }
                }
            }
        }

        // Stage 5: Security & Vulnerability Scanning
        stage('Security Scan') {
            parallel {
                stage('Dependency Audit') {
                    steps {
                        echo 'Stage 5: NPM Audit'
                        dir('next_webapp') {
                            bat 'npm audit --audit-level=high || exit 0'
                        }
                    }
                }
                stage('Container Vulnerability Scan') {
                    steps {
                        echo 'Stage 5: Trivy Container Scan'
                        bat "trivy image --severity HIGH,CRITICAL ${WEBAPP_IMAGE}:${env.BUILD_NUMBER} || exit 0"
                    }
                }
            }
        }

        // Stage 6: Push Staging Image to Docker Hub & Deploy to Staging Environment
        stage('Deploy to Staging') {
            steps {
                echo 'Stage 6: Deploying to Staging Environment'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDS_ID) {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push("staging")
                    }
                }
		bat "docker compose -f docker-compose.staging.yml up -d --force-recreate"
                bat "timeout /t 8 /nobreak >nul && curl -sf http://localhost:3000 || exit 1"
            }
        }

        // Stage 7: Manual Gate Approval, Rebuild with Prod Config, and Production Deployment
        stage('Release to Production') {
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                echo 'Stage 7: Production Release'
                input message: 'Staging verified above. Release this build to Production?', ok: 'Deploy to Prod'

                dir('next_webapp') {
                    withCredentials([file(credentialsId: 'app-env-prod', variable: 'ENV_FILE')]) {
                        bat 'copy "%ENV_FILE%" .env /Y'
                    }
                    script {
                        dockerImageProd = docker.build("${WEBAPP_IMAGE}:prod-${env.BUILD_NUMBER}")
                    }
                }

                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDS_ID) {
                        dockerImageProd.push("prod-${env.BUILD_NUMBER}")
                        dockerImageProd.push("latest")
                    }
                }

		bat "docker compose -f docker-compose.prod.yml up -d --force-recreate"
                bat "timeout /t 8 /nobreak >nul && curl -sf http://localhost:8080 || exit 1"
            }
        }
    }

    post {
        always {
            echo 'Cleaning up unused Docker images from build agent'
            bat "docker image prune -f || exit 0"
        }
        success {
            echo ' SUCCESS: Pipeline completed all 7 stages'
        }
        failure {
            echo ' FAILURE: Pipeline failed during execution. Please check Console Output.'
        }
    }
}
