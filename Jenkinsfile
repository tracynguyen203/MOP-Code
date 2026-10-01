// Jenkinsfile
// CI/CD pipeline for Chameleon-company/Planting-Optimisation-Tool
// Stages: Build -> Test -> Code Quality -> Security -> Deploy -> Release -> Monitoring
//
// Assumed repo layout (adjust paths if yours differs):
//   backend/   -> FastAPI (Python), requirements.txt, Dockerfile, tests/
//   frontend/  -> React + Vite, package.json, Dockerfile
//
// Required Jenkins plugins: Pipeline, Git, Docker Pipeline, JUnit, HTML Publisher,
//   SonarQube Scanner, OWASP Dependency-Check (or Trivy via sh), Credentials Binding,
//   Slack Notification (optional), Datadog Plugin (optional)
//
// Required Jenkins credentials (Manage Jenkins > Credentials):
//   github-creds        - GitHub username/PAT (repo checkout + status)
//   dockerhub-creds      - Docker Hub (or GHCR) username/password
//   sonarqube-token       - SonarQube auth token (secret text)
//   staging-server-creds  - SSH key for staging Docker host
//   prod-server-creds     - SSH key for production Docker host
//   datadog-api-key       - Datadog API key (secret text)

pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
        ansiColor('xterm')
    }

    environment {
        DOCKERHUB_NAMESPACE = 'yourdockerhubusername'          // TODO: replace
        BACKEND_IMAGE        = "${DOCKERHUB_NAMESPACE}/pot-backend"
        FRONTEND_IMAGE        = "${DOCKERHUB_NAMESPACE}/pot-frontend"
        IMAGE_TAG              = "${env.BUILD_NUMBER}"
        SONARQUBE_ENV            = 'MySonarQubeServer'           // Name configured in Jenkins > System
        STAGING_HOST               = 'staging.example.com'          // TODO: replace
        PROD_HOST                   = 'prod.example.com'               // TODO: replace
        DATADOG_SITE                  = 'datadoghq.com'
        GITHUB_REPO_URL              = 'https://github.com/your-username/Planting-Optimisation-Tool.git' // TODO: replace
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_SHORT = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
                }
                echo "Checked out commit ${env.GIT_COMMIT_SHORT}"
            }
        }

        // ---------------------------------------------------------------
        // STAGE 4: BUILD
        // ---------------------------------------------------------------
        stage('Build') {
            parallel {
                stage('Build Backend') {
                    steps {
                        dir('backend') {
                            sh '''
                                python3 -m venv .venv
                                . .venv/bin/activate
                                pip install --upgrade pip
                                pip install -r requirements.txt
                            '''
                            sh "docker build -t ${BACKEND_IMAGE}:${IMAGE_TAG} -t ${BACKEND_IMAGE}:latest ."
                        }
                    }
                }
                stage('Build Frontend') {
                    steps {
                        dir('frontend') {
                            sh '''
                                npm ci
                                npm run build
                            '''
                            sh "docker build -t ${FRONTEND_IMAGE}:${IMAGE_TAG} -t ${FRONTEND_IMAGE}:latest ."
                        }
                    }
                }
            }
            post {
                success {
                    echo "Build artefacts created: ${BACKEND_IMAGE}:${IMAGE_TAG}, ${FRONTEND_IMAGE}:${IMAGE_TAG}"
                    archiveArtifacts artifacts: 'frontend/dist/**', allowEmptyArchive: true
                }
            }
        }

        // ---------------------------------------------------------------
        // STAGE 5: TEST
        // ---------------------------------------------------------------
        stage('Test') {
            parallel {
                stage('Backend Tests') {
                    steps {
                        dir('backend') {
                            sh '''
                                . .venv/bin/activate
                                pip install pytest pytest-cov
                                pytest tests/ --junitxml=test-results/backend-results.xml \
                                    --cov=app --cov-report=xml:coverage.xml
                            '''
                        }
                    }
                    post {
                        always {
                            junit 'backend/test-results/backend-results.xml'
                        }
                    }
                }
                stage('Frontend Tests') {
                    steps {
                        dir('frontend') {
                            sh '''
                                npm run test -- --run --reporter=junit --outputFile=test-results/frontend-results.xml
                            '''
                        }
                    }
                    post {
                        always {
                            junit 'frontend/test-results/frontend-results.xml'
                        }
                    }
                }
            }
        }

        // ---------------------------------------------------------------
        // STAGE 6: CODE QUALITY
        // ---------------------------------------------------------------
        stage('Code Quality') {
            steps {
                script {
                    // 'SonarScanner' must match the name you give it under
                    // Manage Jenkins > Tools > SonarQube Scanner installations
                    env.SCANNER_HOME = tool 'SonarScanner'
                }
                withSonarQubeEnv("${SONARQUBE_ENV}") {
                    sh '''
                        ${SCANNER_HOME}/bin/sonar-scanner \
                          -Dsonar.projectKey=planting-optimisation-tool \
                          -Dsonar.sources=backend/app,frontend/src \
                          -Dsonar.python.coverage.reportPaths=backend/coverage.xml \
                          -Dsonar.exclusions=**/node_modules/**,**/dist/**,**/.venv/**
                    '''
                }
            }
            post {
                always {
                    // Fail the build if the Sonar quality gate fails (non-blocking timeout = 5 min)
                    timeout(time: 5, unit: 'MINUTES') {
                        script {
                            def qg = waitForQualityGate()
                            if (qg.status != 'OK') {
                                echo "WARNING: Quality gate status was ${qg.status}"
                                currentBuild.result = 'UNSTABLE'
                            }
                        }
                    }
                }
            }
        }

        // ---------------------------------------------------------------
        // STAGE 7: SECURITY
        // ---------------------------------------------------------------
        stage('Security') {
            parallel {
                stage('Dependency Scan (Python)') {
                    steps {
                        dir('backend') {
                            sh '''
                                . .venv/bin/activate
                                pip install safety
                                safety check -r requirements.txt --json > ../safety-report.json || true
                            '''
                        }
                    }
                }
                stage('Dependency Scan (npm)') {
                    steps {
                        dir('frontend') {
                            sh 'npm audit --json > ../npm-audit-report.json || true'
                        }
                    }
                }
                stage('Container Image Scan (Trivy)') {
                    steps {
                        sh '''
                            trivy image --exit-code 0 --severity HIGH,CRITICAL \
                                --format json -o trivy-backend-report.json ${BACKEND_IMAGE}:${IMAGE_TAG}
                            trivy image --exit-code 0 --severity HIGH,CRITICAL \
                                --format json -o trivy-frontend-report.json ${FRONTEND_IMAGE}:${IMAGE_TAG}
                        '''
                    }
                }
            }
            post {
                always {
                    archiveArtifacts artifacts: '*-report.json', allowEmptyArchive: true
                    // See report document for the manual triage: issue / severity / remediation
                }
            }
        }

        // ---------------------------------------------------------------
        // STAGE 8: DEPLOY (to test/staging environment)
        // ---------------------------------------------------------------
        stage('Deploy to Staging') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push ${BACKEND_IMAGE}:${IMAGE_TAG}
                        docker push ${FRONTEND_IMAGE}:${IMAGE_TAG}
                    '''
                }
                sshagent(credentials: ['staging-server-creds']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no docker-compose.staging.yml \
                            deploy@${STAGING_HOST}:/opt/pot/docker-compose.yml
                        ssh -o StrictHostKeyChecking=no deploy@${STAGING_HOST} \
                            "cd /opt/pot && export IMAGE_TAG=${IMAGE_TAG} && \
                             docker compose pull && docker compose up -d --remove-orphans"
                    '''
                    // DOCKERHUB_NAMESPACE, DB_PASSWORD etc. live in /opt/pot/.env on the
                    // host itself (created once, see setup guide) - only IMAGE_TAG changes
                    // per build, so only it needs to be passed in on every deploy.
                }
                // Backend container publishes :8000 directly (no reverse proxy in this
                // assignment setup) - hit that port, not :443/:80.
                sh 'sleep 15 && curl -sf http://${STAGING_HOST}:8000/api/health || exit 1'
            }
        }

        // ---------------------------------------------------------------
        // STAGE 9: RELEASE (promote to production)
        // ---------------------------------------------------------------
        stage('Release to Production') {
            when {
                branch 'master'
            }
            steps {
                timeout(time: 1, unit: 'HOURS') {
                    input message: "Promote build ${IMAGE_TAG} to PRODUCTION?", ok: 'Release'
                }
                sh '''
                    docker tag ${BACKEND_IMAGE}:${IMAGE_TAG} ${BACKEND_IMAGE}:release-${IMAGE_TAG}
                    docker tag ${FRONTEND_IMAGE}:${IMAGE_TAG} ${FRONTEND_IMAGE}:release-${IMAGE_TAG}
                    docker push ${BACKEND_IMAGE}:release-${IMAGE_TAG}
                    docker push ${FRONTEND_IMAGE}:release-${IMAGE_TAG}
                '''
                sshagent(credentials: ['prod-server-creds']) {
                    sh '''
                        scp -o StrictHostKeyChecking=no docker-compose.prod.yml \
                            deploy@${PROD_HOST}:/opt/pot/docker-compose.yml
                        ssh -o StrictHostKeyChecking=no deploy@${PROD_HOST} \
                            "cd /opt/pot && export IMAGE_TAG=release-${IMAGE_TAG} && \
                             docker compose pull && docker compose up -d --remove-orphans"
                    '''
                }
                sh 'sleep 15 && curl -sf http://${PROD_HOST}:8000/api/health || exit 1'
                withCredentials([usernamePassword(credentialsId: 'github-creds',
                        usernameVariable: 'GIT_USER', passwordVariable: 'GIT_TOKEN')]) {
                    sh '''
                        git tag -a "v${IMAGE_TAG}" -m "Release ${IMAGE_TAG} (${GIT_COMMIT_SHORT})"
                        AUTH_URL=$(echo "${GITHUB_REPO_URL}" | sed "s#https://#https://${GIT_USER}:${GIT_TOKEN}@#")
                        git push "${AUTH_URL}" "v${IMAGE_TAG}"
                    '''
                }
            }
        }

        // ---------------------------------------------------------------
        // STAGE 10: MONITORING & ALERTING
        // ---------------------------------------------------------------
        stage('Monitoring & Alerting') {
            steps {
                withCredentials([string(credentialsId: 'datadog-api-key', variable: 'DD_API_KEY')]) {
                    sh '''
                        cat > dd-event.json << EOF
{"title": "POT deploy ${IMAGE_TAG}", "text": "Deployed commit ${GIT_COMMIT_SHORT} to production", "tags": ["service:planting-optimisation-tool", "env:production"]}
EOF
                        curl -sf -X POST "https://api.${DATADOG_SITE}/api/v1/events" \
                          -H "DD-API-KEY: ${DD_API_KEY}" \
                          -H "Content-Type: application/json" \
                          -d @dd-event.json
                    '''
                }
                sh 'curl -sf http://${PROD_HOST}:8000/api/health || echo "WARN: health check failed post-deploy"'
            }
        }
    }

    post {
        always {
            echo "Pipeline finished with status: ${currentBuild.currentResult}"
        }
        failure {
            withCredentials([string(credentialsId: 'datadog-api-key', variable: 'DD_API_KEY')]) {
                sh '''
                    cat > dd-failure-event.json << EOF
{"title": "POT pipeline FAILED", "text": "Build ${BUILD_NUMBER} failed.", "alert_type": "error", "tags": ["service:planting-optimisation-tool"]}
EOF
                    curl -sf -X POST "https://api.${DATADOG_SITE}/api/v1/events" \
                      -H "DD-API-KEY: ${DD_API_KEY}" -H "Content-Type: application/json" \
                      -d @dd-failure-event.json || true
                '''
            }
        }
    }
}
