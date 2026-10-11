pipeline {
    agent any
    environment {
        APP_NAME = 'FlaskPractice'
        STAGING_DIR = 'staging'
        PYTHON_PACKAGES = '.python-packages'
    }
    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out Flask application source code'
                checkout scm
                sh '''
                echo "Current workspace: $WORKSPACE"
                echo "Repository files:"
                ls -la
                '''
            }
        }
        stage('Build') {
            steps {
                echo 'Checking Python installation'
                sh '''
                python3 --version
                python3 -m pip --version
                if [ ! -f requirements.txt ];
                then
                echo "ERROR: requirements.txt was not found."
                exit 1
                fi
                '''
                echo 'Installing dependencies into the workspace'
                sh '''
                mkdir -p "$WORKSPACE/$PYTHON_PACKAGES"
                python3 -m pip install \
                --target "$WORKSPACE/$PYTHON_PACKAGES" \
                -r requirements.txt
                '''
            }
        }
        stage('Test') {
            steps {
                echo 'Running Flask application tests'
                withEnv(['MONGO_URI=mongodb+srv://chandhini:vasuki123@cluster0.hq71adi.mongodb.net/']) {
                    sh '''
                    export PYTHONPATH="$WORKSPACE/$PYTHON_PACKAGES${
                        PYTHONPATH:+:$PYTHONPATH
                    }"
                    python3 -m pytest -v
                    '''
                }
            }
        }
        stage('Deploy') {
            steps {
                echo 'Preparing staging deployment'
                sh '''
                mkdir -p "$WORKSPACE/$STAGING_DIR"
                tar \
                --exclude='./.git' \
                --exclude='./.venv' \
                --exclude='./.python-packages' \
                --exclude="./$STAGING_DIR" \
                --exclude='./__pycache__' \
                -cf - . | tar -xf - -C "$WORKSPACE/$STAGING_DIR"
                echo "Staging files prepared:"
                ls -la "$WORKSPACE/$STAGING_DIR"
                '''
            }
        }
    }
    post {
        success {
            echo 'CI/CD pipeline completed successfully'
            emailext(
            to: 'chandhinishrivs@gmail.com',
            subject: "SUCCESS: ${
                env.JOB_NAME
            } #${
                env.BUILD_NUMBER
            }",
            body: """Build succeeded.
            Job: ${
                env.JOB_NAME
            }
            Build Number: ${
                env.BUILD_NUMBER
            }
            Build URL: ${
                env.BUILD_URL
            }
            The Flask application completed the configured pipeline stages."""
            )
        }
        failure {
            echo 'CI/CD pipeline failed'
            emailext(
            to: 'chandhinishrivs@gmail.com',
            subject: "FAILURE: ${
                env.JOB_NAME
            } #${
                env.BUILD_NUMBER
            }",
            body: """Build failed.
            Job: ${
                env.JOB_NAME
            }
            Build Number: ${
                env.BUILD_NUMBER
            }
            Build URL: ${
                env.BUILD_URL
            }
            Please check the Jenkins Console Output for the error details."""
            )
        }
        always {
            echo 'Pipeline execution finished'
        }
    }
}