// =============================================================================
// CloudEagle – sync-service Jenkins Declarative Pipeline
// Supports: QA (develop branch), Staging (staging branch), Prod (main branch)
// =============================================================================

pipeline {

    agent { label 'docker-builder' }

    // -------------------------------------------------------------------------
    // Pipeline-wide environment variables
    // -------------------------------------------------------------------------
    environment {
        APP_NAME            = 'sync-service'
        GCP_PROJECT         = credentials('gcp-project-id')
        GCP_REGION          = 'us-central1'
        ARTIFACT_REGISTRY   = "${GCP_REGION}-docker.pkg.dev/${GCP_PROJECT}/sync-service"
        IMAGE_TAG           = "${env.GIT_COMMIT[0..7]}"
        FULL_IMAGE          = "${ARTIFACT_REGISTRY}/${APP_NAME}:${IMAGE_TAG}"
        SONAR_TOKEN         = credentials('sonarqube-token')
        SLACK_CHANNEL       = '#deployments'

        // Resolved per branch in stages below
        DEPLOY_ENV          = resolveEnv()
        GKE_CLUSTER         = resolveCluster()
    }

    // -------------------------------------------------------------------------
    // Trigger: GitHub webhooks push events + PR events
    // -------------------------------------------------------------------------
    triggers {
        githubPush()
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '30'))
        timeout(time: 45, unit: 'MINUTES')
        disableConcurrentBuilds(abortPrevious: true)
        timestamps()
    }

    // =========================================================================
    // STAGES
    // =========================================================================
    stages {

        // ---------------------------------------------------------------------
        stage('Checkout') {
        // ---------------------------------------------------------------------
            steps {
                checkout scm
                script {
                    env.GIT_COMMIT_MSG = sh(
                        script: 'git log -1 --pretty=%B',
                        returnStdout: true
                    ).trim()
                    echo "Branch: ${env.BRANCH_NAME} | Commit: ${env.GIT_COMMIT[0..7]}"
                    echo "Message: ${env.GIT_COMMIT_MSG}"
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Build & Unit Test') {
        // ---------------------------------------------------------------------
            steps {
                sh '''
                    ./mvnw clean verify \
                        -Dspring.profiles.active=test \
                        -Dmaven.test.failure.ignore=false \
                        --no-transfer-progress
                '''
            }
            post {
                always {
                    junit 'target/surefire-reports/**/*.xml'
                    jacoco(
                        execPattern: 'target/jacoco.exec',
                        classPattern: 'target/classes',
                        sourcePattern: 'src/main/java',
                        minimumInstructionCoverage: '70'
                    )
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Static Analysis') {
        // ---------------------------------------------------------------------
            // Run on PRs and branch merges alike
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        ./mvnw sonar:sonar \
                            -Dsonar.projectKey=${APP_NAME} \
                            -Dsonar.branch.name=${BRANCH_NAME} \
                            --no-transfer-progress
                    '''
                }
                // Block pipeline if quality gate fails
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Docker Build & Push') {
        // ---------------------------------------------------------------------
            // Skip for PRs — only run on branch merges
            when {
                anyOf {
                    branch 'develop'
                    branch 'staging'
                    branch 'main'
                }
            }
            steps {
                script {
                    // Authenticate to GCP Artifact Registry
                    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
                        sh 'gcloud auth activate-service-account --key-file=$GCP_KEY'
                        sh "gcloud auth configure-docker ${GCP_REGION}-docker.pkg.dev --quiet"
                    }

                    sh """
                        docker build \
                            --build-arg BUILD_VERSION=${IMAGE_TAG} \
                            --build-arg SPRING_PROFILE=${DEPLOY_ENV} \
                            -t ${FULL_IMAGE} \
                            -f Dockerfile .
                    """

                    sh "docker push ${FULL_IMAGE}"

                    // Also tag as environment-latest for quick reference
                    sh "docker tag ${FULL_IMAGE} ${ARTIFACT_REGISTRY}/${APP_NAME}:${DEPLOY_ENV}-latest"
                    sh "docker push ${ARTIFACT_REGISTRY}/${APP_NAME}:${DEPLOY_ENV}-latest"
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Deploy to QA') {
        // ---------------------------------------------------------------------
            when { branch 'develop' }
            steps {
                deployToGKE('qa', GKE_CLUSTER, false)
            }
        }

        // ---------------------------------------------------------------------
        stage('Deploy to Staging') {
        // ---------------------------------------------------------------------
            when { branch 'staging' }
            steps {
                deployToGKE('staging', GKE_CLUSTER, false)
            }
        }

        // ---------------------------------------------------------------------
        stage('Production Approval Gate') {
        // ---------------------------------------------------------------------
            when { branch 'main' }
            steps {
                script {
                    // Save current stable image tag BEFORE deploying (for rollback)
                    def currentStable = sh(
                        script: "kubectl get deployment ${APP_NAME} -n prod -o jsonpath='{.spec.template.spec.containers[0].image}'",
                        returnStdout: true
                    ).trim()
                    env.ROLLBACK_IMAGE = currentStable
                    echo "Rollback image saved: ${env.ROLLBACK_IMAGE}"

                    slackSend(
                        channel: SLACK_CHANNEL,
                        color: 'warning',
                        message: "⏳ *${APP_NAME}* is awaiting prod deployment approval.\n" +
                                 "Commit: `${IMAGE_TAG}` | Branch: `main`\n" +
                                 "Approve here: ${env.BUILD_URL}input"
                    )

                    // Pipeline parks here — someone must click Approve
                    timeout(time: 30, unit: 'MINUTES') {
                        input(
                            message: "Deploy ${APP_NAME}:${IMAGE_TAG} to PRODUCTION?",
                            ok: 'Deploy to Production',
                            submitterParameter: 'APPROVED_BY'
                        )
                    }
                    echo "Approved by: ${env.APPROVED_BY}"
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Deploy to Production') {
        // ---------------------------------------------------------------------
            when { branch 'main' }
            steps {
                script {
                    // Blue/Green: deploy to Green deployment, then flip traffic
                    deployBlueGreen('prod', GKE_CLUSTER)
                }
            }
        }

        // ---------------------------------------------------------------------
        stage('Smoke Tests') {
        // ---------------------------------------------------------------------
            when {
                anyOf {
                    branch 'develop'
                    branch 'staging'
                    branch 'main'
                }
            }
            steps {
                script {
                    def smokeTarget = getSmokeUrl(DEPLOY_ENV)
                    sh """
                        echo "Running smoke tests against: ${smokeTarget}"
                        curl --fail --retry 5 --retry-delay 10 \
                             -H "X-Health-Check: jenkins" \
                             ${smokeTarget}/actuator/health

                        # Basic API smoke test
                        STATUS=\$(curl -s -o /dev/null -w "%{http_code}" ${smokeTarget}/api/v1/status)
                        if [ "\$STATUS" != "200" ]; then
                            echo "Smoke test FAILED — HTTP \$STATUS"
                            exit 1
                        fi
                        echo "Smoke tests PASSED"
                    """
                }
            }
        }

    } // end stages

    // =========================================================================
    // POST ACTIONS
    // =========================================================================
    post {

        success {
            script {
                if (env.BRANCH_NAME in ['develop', 'staging', 'main']) {
                    // Tag this image as stable for rollback reference
                    sh "docker tag ${FULL_IMAGE} ${ARTIFACT_REGISTRY}/${APP_NAME}:stable-${DEPLOY_ENV}"
                    sh "docker push ${ARTIFACT_REGISTRY}/${APP_NAME}:stable-${DEPLOY_ENV}"

                    slackSend(
                        channel: SLACK_CHANNEL,
                        color: 'good',
                        message: "✅ *${APP_NAME}* deployed to *${DEPLOY_ENV.toUpperCase()}* successfully.\n" +
                                 "Image: `${IMAGE_TAG}` | Approved by: ${env.APPROVED_BY ?: 'auto'}"
                    )
                }
            }
        }

        failure {
            script {
                if (env.BRANCH_NAME in ['develop', 'staging', 'main'] && env.ROLLBACK_IMAGE) {
                    echo "Deployment FAILED — initiating automatic rollback to ${env.ROLLBACK_IMAGE}"
                    sh """
                        kubectl set image deployment/${APP_NAME} \
                            ${APP_NAME}=${env.ROLLBACK_IMAGE} \
                            -n ${DEPLOY_ENV} --record
                        kubectl rollout status deployment/${APP_NAME} -n ${DEPLOY_ENV} --timeout=3m
                    """
                    slackSend(
                        channel: SLACK_CHANNEL,
                        color: 'danger',
                        message: "🚨 *${APP_NAME}* deployment to *${DEPLOY_ENV.toUpperCase()}* FAILED.\n" +
                                 "Auto-rollback to `${env.ROLLBACK_IMAGE}` executed.\n" +
                                 "Build: ${env.BUILD_URL}"
                    )
                } else {
                    slackSend(
                        channel: SLACK_CHANNEL,
                        color: 'danger',
                        message: "❌ *${APP_NAME}* pipeline FAILED on `${env.BRANCH_NAME}`.\n" +
                                 "Build: ${env.BUILD_URL}"
                    )
                }
            }
        }

        always {
            // Clean up local Docker images to save disk space
            sh "docker rmi ${FULL_IMAGE} || true"
            cleanWs()
        }
    }

} // end pipeline

// =============================================================================
// HELPER FUNCTIONS
// =============================================================================

def resolveEnv() {
    switch (env.BRANCH_NAME) {
        case 'develop':  return 'qa'
        case 'staging':  return 'staging'
        case 'main':     return 'prod'
        default:         return 'dev'
    }
}

def resolveCluster() {
    switch (env.BRANCH_NAME) {
        case 'develop':  return 'cloudeagle-qa-cluster'
        case 'staging':  return 'cloudeagle-staging-cluster'
        case 'main':     return 'cloudeagle-prod-cluster'
        default:         return 'cloudeagle-qa-cluster'
    }
}

def getSmokeUrl(String environment) {
    def urls = [
        qa:      'https://sync-service-qa.internal.cloudeagle.ai',
        staging: 'https://sync-service-staging.internal.cloudeagle.ai',
        prod:    'https://sync-service.cloudeagle.ai'
    ]
    return urls[environment] ?: 'http://localhost:8080'
}

// Standard GKE rolling deploy (used for QA and Staging)
def deployToGKE(String environment, String cluster, Boolean blueGreen) {
    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
        sh """
            gcloud auth activate-service-account --key-file=\$GCP_KEY
            gcloud container clusters get-credentials ${cluster} \
                --region ${env.GCP_REGION} \
                --project ${env.GCP_PROJECT}

            # Save current image for rollback
            PREV_IMAGE=\$(kubectl get deployment ${env.APP_NAME} \
                -n ${environment} \
                -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null || echo "")
            echo "Previous image: \$PREV_IMAGE"

            # Apply new image
            kubectl set image deployment/${env.APP_NAME} \
                ${env.APP_NAME}=${env.FULL_IMAGE} \
                -n ${environment} --record

            # Wait for rollout
            kubectl rollout status deployment/${env.APP_NAME} \
                -n ${environment} \
                --timeout=5m
        """
    }
    env.ROLLBACK_IMAGE = sh(
        script: "kubectl get deployment ${env.APP_NAME} -n ${environment} -o jsonpath='{.spec.template.spec.containers[0].image}' 2>/dev/null || echo ''",
        returnStdout: true
    ).trim()
}

// Blue/Green deploy for Production
def deployBlueGreen(String environment, String cluster) {
    withCredentials([file(credentialsId: 'gcp-sa-key', variable: 'GCP_KEY')]) {
        sh """
            gcloud auth activate-service-account --key-file=\$GCP_KEY
            gcloud container clusters get-credentials ${cluster} \
                --region ${env.GCP_REGION} \
                --project ${env.GCP_PROJECT}

            # ---------- Step 1: Determine current colour ----------
            CURRENT_COLOR=\$(kubectl get service ${env.APP_NAME}-active \
                -n ${environment} \
                -o jsonpath='{.spec.selector.color}' 2>/dev/null || echo "blue")
            if [ "\$CURRENT_COLOR" = "blue" ]; then
                NEW_COLOR="green"
            else
                NEW_COLOR="blue"
            fi
            echo "Current: \$CURRENT_COLOR  |  Deploying to: \$NEW_COLOR"

            # ---------- Step 2: Deploy to Green ----------
            kubectl set image deployment/${env.APP_NAME}-\${NEW_COLOR} \
                ${env.APP_NAME}=${env.FULL_IMAGE} \
                -n ${environment} --record

            kubectl rollout status deployment/${env.APP_NAME}-\${NEW_COLOR} \
                -n ${environment} --timeout=5m

            # ---------- Step 3: Smoke test Green internally ----------
            GREEN_POD=\$(kubectl get pod -n ${environment} \
                -l app=${env.APP_NAME},color=\${NEW_COLOR} \
                -o jsonpath='{.items[0].metadata.name}')
            kubectl exec \$GREEN_POD -n ${environment} -- \
                curl -sf http://localhost:8080/actuator/health

            # ---------- Step 4: Cut over traffic ----------
            kubectl patch service ${env.APP_NAME}-active \
                -n ${environment} \
                -p "{\\"spec\\":{\\"selector\\":{\\"color\\":\\"\${NEW_COLOR}\\"}}}"

            echo "Traffic now routing to \$NEW_COLOR"

            # ---------- Step 5: Keep old colour warm for 10 min ----------
            echo "Old (\$CURRENT_COLOR) kept warm. Will auto-scale down after monitoring window."
            # A separate CronJob / Cloud Scheduler handles scale-down after 10m
        """
    }
}
