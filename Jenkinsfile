pipeline {
    agent any

    triggers {
        pollSCM('H/2 * * * *')
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Creating virtual environment and installing dependencies'

                sh '''
                    python3 --version
                    python3 -m venv .venv
                    .venv/bin/python -m pip install --upgrade pip
                    .venv/bin/python -m pip install -r requirements.txt
                '''
            }
        }

        stage('Test') {
            steps {
                echo 'Running automated tests'

                sh '''
                    .venv/bin/python -m pytest -q
                '''
            }
        }

        stage('Code Inspection') {
            steps {
                echo 'Running SonarQube analysis'

                withCredentials([
                    string(
                        credentialsId: 'sonar-token',
                        variable: 'SONAR_TOKEN'
                    )
                ]) {
                    sh '''
                        export PATH=/opt/sonar-scanner/bin:$PATH

                        sonar-scanner \
                          -Dsonar.host.url=http://localhost:9000 \
                          -Dsonar.token="$SONAR_TOKEN"
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo 'Deploying Flask application locally'

                sh '''
                    set -e

                    DEPLOY_DIR="/tmp/student-task-manager"

                    mkdir -p "$DEPLOY_DIR"

                    cp -r app.py templates static requirements.txt "$DEPLOY_DIR/"

                    cd "$DEPLOY_DIR"

                    python3 -m venv .venv
                    .venv/bin/python -m pip install -r requirements.txt

                    # Stop the previously deployed application if it is running.
                    if [ -f app.pid ]; then
                        OLD_PID=$(cat app.pid)

                        if kill -0 "$OLD_PID" 2>/dev/null; then
                            kill "$OLD_PID"
                            sleep 2
                        fi

                        rm -f app.pid
                    fi

                    # Start the Flask application in the background.
                    nohup .venv/bin/python app.py \
                        > app.log 2>&1 < /dev/null &

                    APP_PID=$!
                    echo "$APP_PID" > app.pid

                    # Wait for Flask to become ready.
                    for i in $(seq 1 15); do
                        if .venv/bin/python -c \
                            "import urllib.request; urllib.request.urlopen('http://127.0.0.1:5000/health', timeout=2)" \
                            >/dev/null 2>&1; then
                            echo "Application health check passed."
                            exit 0
                        fi

                        if ! kill -0 "$APP_PID" 2>/dev/null; then
                            echo "ERROR: Flask application stopped unexpectedly."
                            cat app.log
                            exit 1
                        fi

                        sleep 2
                    done

                    echo "ERROR: Application did not become healthy."
                    cat app.log
                    exit 1
                '''
            }
        }
    }

    post {
        success {
            echo 'SUCCESS: CI/CD pipeline completed successfully!'
            echo 'SUCCESS: Build, tests, SonarQube and deployment passed.'
            echo 'Application URL: http://localhost:5000'
        }

        failure {
            echo 'FAILURE: CI/CD pipeline failed.'
            echo 'Check Console Output to identify the failed stage.'
            echo 'Deployment is not considered successful.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}