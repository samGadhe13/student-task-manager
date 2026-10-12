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
    }

    post {
        success {
            echo 'CI pipeline including SonarQube completed successfully'
        }

        failure {
            echo 'Pipeline failed. Review the failed stage and Console Output.'
        }

        always {
            echo 'Pipeline execution finished.'
        }
    }
}