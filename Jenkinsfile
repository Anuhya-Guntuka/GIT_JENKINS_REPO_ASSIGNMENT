pipeline {
    agent any

    options {
        timestamps()
        skipDefaultCheckout(true)
    }

    environment {
        // Fail pipeline if any vulnerability has CVSS >= this number
        FAIL_ON_CVSS = '7.0'

        // Artifact (change to target/*.war if your project builds WAR)
        ARTIFACT_GLOB = 'target/*.jar'

        // Deploy (only used on main branch)
        APP_SERVER_HOST = 'YOUR_APP_SERVER_IP'
        APP_SERVER_USER = 'ubuntu'
        APP_SERVER_DEPLOY_DIR = '/opt/app'
        APP_RESTART_CMD = 'sudo systemctl restart myapp'   // change this
        SSH_CRED_ID = 'app-server-ssh-key'                 // Jenkins credential ID
    }

    stages {

        stage('1) Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Build Info') {
            steps {
                sh 'hostname'
                sh 'whoami'
            }
        }

        stage('2) Build & Test using Maven') {
            steps {
                sh 'mvn clean test'
            }
        }

        stage('3) Security Scan (OWASP Dependency-Check)') {
            steps {
                sh '''
                    set -e
                    DC_VERSION="9.0.10"
                    DC_ZIP="dependency-check-${DC_VERSION}-release.zip"
                    DC_URL="https://github.com/jeremylong/DependencyCheck/releases/download/v${DC_VERSION}/${DC_ZIP}"

                    rm -rf dependency-check dependency-check-report || true
                    mkdir -p dependency-check-report

                    if [ ! -f "$DC_ZIP" ]; then
                      curl -L -o "$DC_ZIP" "$DC_URL"
                    fi

                    unzip -oq "$DC_ZIP" -d dependency-check
                    chmod +x dependency-check/dependency-check/bin/dependency-check.sh

                    dependency-check/dependency-check/bin/dependency-check.sh \
                      --project "Jenkins-DependencyCheck" \
                      --scan . \
                      --format "HTML" \
                      --out dependency-check-report \
                      --failOnCVSS "${FAIL_ON_CVSS}"
                '''
            }
            post {
                always {
                    archiveArtifacts artifacts: 'dependency-check-report/**', allowEmptyArchive: true
                }
            }
        }

        stage('4) Package Stage') {
            steps {
                sh 'mvn -DskipTests package'
                archiveArtifacts artifacts: "${ARTIFACT_GLOB}", fingerprint: true
            }
        }

        stage('5) Deploy to Application Server (main branch only)') {
            when {
                branch 'main'
            }
            steps {
                sshagent(credentials: ["${SSH_CRED_ID}"]) {
                    sh '''
                        set -e
                        ARTIFACT_FILE=$(ls -1 ${ARTIFACT_GLOB} | head -n 1)
                        if [ -z "$ARTIFACT_FILE" ]; then
                          echo "ERROR: No artifact found at ${ARTIFACT_GLOB}"
                          exit 1
                        fi

                        echo "Copying artifact to server..."
                        scp -o StrictHostKeyChecking=no "$ARTIFACT_FILE" \
                          "${APP_SERVER_USER}@${APP_SERVER_HOST}:${APP_SERVER_DEPLOY_DIR}/app.jar"

                        echo "Restarting app..."
                        ssh -o StrictHostKeyChecking=no "${APP_SERVER_USER}@${APP_SERVER_HOST}" \
                          "${APP_RESTART_CMD}"
                    '''
                }
            }
        }
    }

    post {
        always {
            junit allowEmptyResults: true, testResults: '**/surefire-reports/*.xml'
        }
    }
}