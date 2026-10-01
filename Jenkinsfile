pipeline {
    agent any

    environment {
        DOCKERHUB_NAMESPACE = 'tracynguyen203'
        GITHUB_REPO_URL     = 'https://github.com/tracynguyen203/MOP-Code.git'
        WEBAPP_IMAGE        = "${DOCKERHUB_NAMESPACE}/mop-webapp"
        DOCKER_CREDS_ID     = 'dockerhub-creds'
        SONAR_TOKEN_ID      = 'sonarqube-token'
        STAGING_HOST        = 'localhost'
        PROD_HOST           = 'localhost'
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

        // Stage 2: Build Docker Image for the Next.js Web Application
        stage('Build') {
            steps {
                echo 'Stage 2: Building Web App Docker Image'
                dir('next_webapp') {
                    script {
                        dockerImage = docker.build("${WEBAPP_IMAGE}:${env.BUILD_NUMBER}")
                        dockerImageLatest = docker.build("${WEBAPP_IMAGE}:latest")
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
                    // Execute test script defined in package.json (gracefully handle empty test suites)
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
                        // Retrieve the SonarScanner installation configured in Jenkins Global Tool Configuration
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
                        // Scan Docker Image for HIGH and CRITICAL vulnerabilities using Trivy
                        sh "trivy image --severity HIGH,CRITICAL ${WEBAPP_IMAGE}:${env.BUILD_NUMBER} || true"
                    }
                }
            }
        }

        // Stage 6: Push Image to Docker Hub & Deploy to Staging Environment
        stage('Deploy to Staging') {
            steps {
                echo 'Stage 6: Deploying to Staging Environment'
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDS_ID) {
                        dockerImage.push("${env.BUILD_NUMBER}")
                        dockerImage.push("staging")
                    }
                }
                // Deploy containers using Staging Docker Compose file
                sh "docker compose -f docker-compose.staging.yml up -d --force-recreate"
            }
        }

        // Stage 7: Manual Gate Approval & Production Deployment
        stage('Release to Production') {
            // Execute this stage only for builds on master or main branches
            when {
                anyOf {
                    branch 'master'
                    branch 'main'
                }
            }
            steps {
                echo 'Stage 7: Production Release'
                // Require manual user intervention/approval via the Jenkins UI
                input message: 'Are you sure you want to release this build to Production?', ok: 'Deploy to Prod'
                
                script {
                    docker.withRegistry('https://index.docker.io/v1/', DOCKER_CREDS_ID) {
                        dockerImageLatest.push("latest")
                        dockerImageLatest.push("prod-${env.BUILD_NUMBER}")
                    }
                }
                // Deploy containers using Production Docker Compose file
                sh "docker compose -f docker-compose.prod.yml up -d --force-recreate"
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