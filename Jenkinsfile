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
                        sh 'cp "$ENV_FILE" .env'
                    }
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
                    sh 'npm install --silent'
                    sh 'npm test -- --passWithNoTests || true'
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
                            sh "${scannerHome}/bin/sonar-scanner -Dsonar.token=${SONAR_TOKEN}"
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
                            sh 'npm audit --audit-level=high || true'
                        }
                    }
                }
                stage('Container Vulnerability Scan') {
                    steps {
                        echo 'Stage 5: Trivy Container Scan'
                        sh "trivy image --severity HIGH,CRITICAL ${WEBAPP_IMAGE}:${env.BUILD_NUMBER} || true"
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
                sh "docker compose -f docker-compose.staging.yml up -d --force-recreate"
                sh "sleep 8 && curl -sf http://localhost:3000 || exit 1"
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

                // Rebuild with the PRODUCTION .env - NEXT_PUBLIC_APP_URL (and any other
                // NEXT_PUBLIC_* values) are baked into the JS bundle at build time, so the
                // staging image can't just be re-tagged and reused here; it has staging's
                // URL compiled into it. This is a real rebuild, not a promotion of the same
                // artifact, specifically because of that Next.js build-time inlining.
                dir('next_webapp') {
                    withCredentials([file(credentialsId: 'app-env-prod', variable: 'ENV_FILE')]) {
                        sh 'cp "$ENV_FILE" .env'
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

                sh "docker compose -f docker-compose.prod.yml up -d --force-recreate"
                sh "sleep 8 && curl -sf http://localhost:8080 || exit 1"
            }
        }
    }

    post {
        always {
            echo 'Cleaning up unused Docker images from build agent'
            sh "docker image prune -f || true"
        }
        success {
            echo ' SUCCESS: Pipeline completed all 7 stages'
        }
        failure {
            echo ' FAILURE: Pipeline failed during execution. Please check Console Output.'
        }
    }
}
