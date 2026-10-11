pipeline {
    agent any
    options {
        timestamps()
    }
    pipeline {
        agent any
        stages {
            stage('Build') {
                steps {
                    sh '''
                    python3 --version
                    python3 -m pip --version
                    python3 -m pip install --user -r requirements.txt
                    '''
                }
            }
            stage('Test') {
                steps {
                    sh '''
                    python3 -m pytest
                    '''
                }
            }
            stage('Deploy') {
                steps {
                    echo 'Deployment stage started'
                }
            }
        }
    }
    stage('Deploy to Staging') {
        steps {
            sh '''
            set -eu
            STAGING_DIR="$WORKSPACE/staging"
            mkdir -p "$STAGING_DIR"
            tar \
            --exclude=.git \
            --exclude=.venv \
            --exclude=staging \
            --exclude=__pycache__ \
            --exclude=.pytest_cache \
            -cf - . | \
            (cd "$STAGING_DIR" && tar -xf -)
            echo "Application copied to: $STAGING_DIR"
            '''
        }
    }
}
post {
    success {
        echo 'Pipeline completed successfully.'
        mail to: 'chandhinishrivs@gmail.com',
        subject: "SUCCESS: ${
            env.JOB_NAME
        } #${
            env.BUILD_NUMBER
        }",
        body: "Build succeeded.\nDetails: ${
            env.BUILD_URL
        }"
    }
    failure {
        echo 'Pipeline failed. Check the console output.'
        mail to: 'chandhinishrivs@gmail.com',
        subject: "FAILED: ${
            env.JOB_NAME
        } #${
            env.BUILD_NUMBER
        }",
        body: "Build failed. Check the console output.\nDetails: ${
            env.BUILD_URL
        }"
    }
}
}