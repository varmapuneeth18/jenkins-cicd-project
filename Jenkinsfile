pipeline {
    // Run on our labeled agent
    agent { label 'maven-agent' }

    environment {
        // Artifact name — adjust if your app produces a different jar name
        APP_JAR        = "target/*.jar"
        APP_SERVER_IP  = "<APP_SERVER_PRIVATE_IP>"   // ← replace this
        APP_SERVER_USER = "ubuntu"
        DEPLOY_DIR     = "/opt/app"
    }

    tools {
        maven 'Maven'   // Must match the name configured in Manage Jenkins → Tools
    }

    stages {

        // ── Stage 1: Checkout ──────────────────────────────────────────
        stage('Checkout Code') {
            steps {
                checkout scm
                echo "✅ Code checked out from branch: ${env.BRANCH_NAME}"
            }
        }

        // ── Stage 2: Build & Test ──────────────────────────────────────
        stage('Build & Test') {
            steps {
                sh 'mvn clean test'
            }
            post {
                always {
                    junit '**/target/surefire-reports/*.xml'
                }
            }
        }

        // ── Stage 3: Security Scan (OWASP Dependency Check) ───────────
        stage('Security Scan') {
            steps {
                sh '''
                    mvn org.owasp:dependency-check-maven:check \
                        -DfailBuildOnCVSS=7 \
                        -Dformat=HTML \
                        -DoutputDirectory=target/dependency-check
                '''
            }
            post {
                always {
                    publishHTML(target: [
                        reportDir:   'target/dependency-check',
                        reportFiles: 'dependency-check-report.html',
                        reportName:  'OWASP Dependency Check Report'
                    ])
                }
            }
        }

        // ── Stage 4: Package ───────────────────────────────────────────
        stage('Package') {
            steps {
                sh 'mvn package -DskipTests'
                archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
                echo "✅ Artifact packaged and archived"
            }
        }

        // ── Stage 5: Deploy (main branch ONLY) ────────────────────────
        stage('Deploy to App Server') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ['app-server-ssh']) {
                    sh """
                        # Copy the jar to the app server
                        scp -o StrictHostKeyChecking=no \
                            target/*.jar \
                            ${APP_SERVER_USER}@${APP_SERVER_IP}:${DEPLOY_DIR}/app.jar

                        # Restart the application on the app server
                        ssh -o StrictHostKeyChecking=no \
                            ${APP_SERVER_USER}@${APP_SERVER_IP} '
                                pkill -f "app.jar" || true
                                nohup java -jar ${DEPLOY_DIR}/app.jar \
                                    > ${DEPLOY_DIR}/app.log 2>&1 &
                                echo "✅ Application restarted"
                            '
                    """
                }
            }
        }
    }

    // ── Post-pipeline notifications ────────────────────────────────────
    post {
        success {
            echo "🎉 Pipeline succeeded on branch: ${env.BRANCH_NAME}"
        }
        failure {
            echo "❌ Pipeline FAILED on branch: ${env.BRANCH_NAME}"
        }
    }
}
