// Jenkins declarative pipeline for the Flask student-registration app.
//
// Stages:  Build  ->  Test  ->  Deploy to Staging
// Trigger: GitHub push webhook (with a 5-minute SCM-poll fallback)
// Notify:  e-mail on success and on failure (Email Extension plugin)
//
// See README.md ("Part 1 - Jenkins pipeline") for the one-time server setup.

pipeline {
    agent any

    options {
        timestamps()
        timeout(time: 20, unit: 'MINUTES')
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '20'))
    }

    triggers {
        // Fires when GitHub calls http://<jenkins>:8080/github-webhook/ on a push.
        githubPush()
        // Fallback in case the webhook cannot reach Jenkins (e.g. no public IP).
        pollSCM('H/5 * * * *')
    }

    parameters {
        // EDIT THE DEFAULT: this is who receives the success / failure e-mails.
        string(name: 'NOTIFY_EMAIL',
               defaultValue: 'chandhinishrivs@gmail.com',
               description: 'Recipient(s) of build success/failure notifications')
    }

    environment {
        APP_NAME             = 'flask-student-app'
        VENV                 = '.venv'
        TEST_MONGO_CONTAINER = 'mongo-ci-test'
        STAGING_PORT         = '5001'
    }

    stages {

        stage('Build') {
            steps {
                echo 'Installing Python dependencies into a virtualenv...'
                sh '''
                    set -e
                    python3 --version
                    python3 -m venv "$VENV"
                    . "$VENV/bin/activate"
                    pip install --upgrade pip
                    pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Starting a throw-away MongoDB and running pytest...'
                // IMPORTANT: the test fixtures delete every document in the
                // `students` collection, so tests must NEVER run against a real
                // database. They run against a disposable local container.
                sh '''
                    set -e
                    docker rm -f "$TEST_MONGO_CONTAINER" >/dev/null 2>&1 || true
                    docker run -d --name "$TEST_MONGO_CONTAINER" -p 127.0.0.1:27017:27017 mongo:7

                    READY=0
                    for i in $(seq 1 30); do
                        if docker exec "$TEST_MONGO_CONTAINER" mongosh --quiet --eval 'db.adminCommand({ping:1})' >/dev/null 2>&1; then
                            READY=1
                            break
                        fi
                        sleep 2
                    done
                    [ "$READY" = "1" ] || { echo 'MongoDB test container did not become ready'; exit 1; }

                    . "$VENV/bin/activate"
                    mkdir -p reports
                    MONGO_URI="mongodb://localhost:27017/test_student_db" \
                    SECRET_KEY="ci-only-secret" \
                    pytest -v --junitxml=reports/junit.xml
                '''
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: 'reports/junit.xml'
                    sh 'docker rm -f "$TEST_MONGO_CONTAINER" >/dev/null 2>&1 || true'
                }
            }
        }

        stage('Deploy to Staging') {
            // Only reached when Build and Test succeeded, and only for main.
            when { expression { isMainBranch() } }
            steps {
                echo 'Building the Docker image and (re)starting the staging container...'
                withCredentials([
                    string(credentialsId: 'staging-mongo-uri',  variable: 'STAGING_MONGO_URI'),
                    string(credentialsId: 'staging-secret-key', variable: 'STAGING_SECRET_KEY')
                ]) {
                    sh '''
                        set +x
                        set -e
                        docker build -t "$APP_NAME:$BUILD_NUMBER" .

                        ENV_FILE="$(mktemp)"
                        trap 'rm -f "$ENV_FILE"' EXIT
                        printf 'MONGO_URI=%s\nSECRET_KEY=%s\n' "$STAGING_MONGO_URI" "$STAGING_SECRET_KEY" > "$ENV_FILE"

                        SKIP_PULL=1 bash deploy/deploy.sh staging "$APP_NAME:$BUILD_NUMBER" "$STAGING_PORT" "$ENV_FILE"
                    '''
                }
            }
        }
    }

    post {
        success {
            emailext(
                to: params.NOTIFY_EMAIL,
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                body: """
                    <h3 style="color:#2e7d32">Build succeeded</h3>
                    <p>
                      <b>Job:</b> ${env.JOB_NAME}<br>
                      <b>Build:</b> #${env.BUILD_NUMBER}<br>
                      <b>Branch:</b> ${env.GIT_BRANCH}<br>
                      <b>Commit:</b> ${env.GIT_COMMIT}<br>
                      <b>Details:</b> <a href="${env.BUILD_URL}">${env.BUILD_URL}</a>
                    </p>
                    <p>Tests passed and the app was deployed to staging (port ${env.STAGING_PORT}).</p>
                """
            )
        }
        failure {
            emailext(
                to: params.NOTIFY_EMAIL,
                subject: "FAILED: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                mimeType: 'text/html',
                attachLog: true,
                body: """
                    <h3 style="color:#c62828">Build failed</h3>
                    <p>
                      <b>Job:</b> ${env.JOB_NAME}<br>
                      <b>Build:</b> #${env.BUILD_NUMBER}<br>
                      <b>Branch:</b> ${env.GIT_BRANCH}<br>
                      <b>Commit:</b> ${env.GIT_COMMIT}<br>
                      <b>Console output:</b> <a href="${env.BUILD_URL}console">${env.BUILD_URL}console</a>
                    </p>
                    <p>The build log is attached.</p>
                """
            )
        }
    }
}

// True for a "Pipeline script from SCM" job on */main (GIT_BRANCH = origin/main)
// and for a multibranch job building the main branch (BRANCH_NAME = main).
def isMainBranch() {
    def branch = env.BRANCH_NAME ?: env.GIT_BRANCH ?: ''
    return branch == 'main' || branch == 'origin/main'
}
