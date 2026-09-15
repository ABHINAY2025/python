pipeline {
    agent any

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Python Test') {
            steps {
                sh '''
                    python3 --version
                    python3 hello_world.py
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        /opt/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=python-demo \
                          -Dsonar.projectName=python-demo \
                          -Dsonar.sources=. \
                          -Dsonar.python.version=3.14
                    '''
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t python-demo:${BUILD_NUMBER} .
                    docker tag python-demo:${BUILD_NUMBER} python-demo:latest
                '''
            }
        }

        stage('Docker Test') {
            steps {
                sh '''
                    docker run --rm python-demo:${BUILD_NUMBER}
                '''
            }
        }

    }

    post {
        success {
            echo 'CI PIPELINE SUCCESS'
        }

        failure {
            echo 'CI PIPELINE FAILED'
        }
    }
}
