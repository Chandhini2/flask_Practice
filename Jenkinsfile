
pipeline {
    agent any

    environment {
        APP_NAME = 'FlaskPractice'
        STAGING_DIR = 'staging'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out Flask application source code'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Checking Python installation'
                sh '''
                    python3 --version
                    python3 -m pip --version
                '''

                echo 'Installing application dependencies'
                sh '''
                    python3 -m pip install --user -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running application tests'
                sh '''
                    python3 -m pytest
                '''
            }
        }

        stage('Deploy') {
            steps {
                echo 'Preparing staging deployment'

                sh '''
                    mkdir -p "$STAGING_DIR"

                    tar \
                      --exclude=.git \
                      --exclude=.venv \
                      --exclude=__pycache__ \
                      --exclude=staging \
                      -cf - . | tar -xf - -C "$STAGING_DIR"

                    echo "Application files copied to staging directory"
                    ls -la "$STAGING_DIR"
                '''
            }
        }
    }

    post {
        success {
            echo 'CI/CD pipeline completed successfully'

            emailext(
                to: 'chandhinishrivs@gmail.com',
                subject: "SUCCESS: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Build succeeded.

Job: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Build URL: ${env.BUILD_URL}

The Flask application passed the pipeline stages."""
            )
        }

        failure {
            echo 'CI/CD pipeline failed'

            emailext(
                to: 'chandhinishrivs@gmail.com',
                subject: "FAILURE: ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                body: """Build failed.

Job: ${env.JOB_NAME}
Build Number: ${env.BUILD_NUMBER}
Build URL: ${env.BUILD_URL}

Please open the Jenkins Console Output to investigate the failure."""
            )
        }

        always {
            echo 'Pipeline execution finished'
        }
    }
}