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
                    pip3 install -r requirements.txt
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

        stage('Docker Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin

                        docker tag python-demo:${BUILD_NUMBER} $DOCKER_USER/python-demo:${BUILD_NUMBER}
                        docker tag python-demo:${BUILD_NUMBER} $DOCKER_USER/python-demo:latest

                        docker push $DOCKER_USER/python-demo:${BUILD_NUMBER}
                        docker push $DOCKER_USER/python-demo:latest

                        docker logout
                    '''
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                sh '''
                    kubectl apply -f deployment.yaml
                    kubectl apply -f service.yaml

                    kubectl rollout restart deployment/python-demo
                    kubectl rollout status deployment/python-demo

                    kubectl get deployments
                    kubectl get pods
                    kubectl get svc
                '''
            }
        }

    }

    post {
        success {
            echo 'CI/CD/KUBERNETES PIPELINE SUCCESS'
        }

        failure {
            echo 'CI/CD/KUBERNETES PIPELINE FAILED'
        }
    }
}
