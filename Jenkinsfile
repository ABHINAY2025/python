pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    environment {
        APP_NAME = 'python-demo'
        IMAGE_REPO = 'abhinayma/python-demo'
        SONARQUBE_SERVER = 'SonarQube'
    }

    stages {
        stage('Validate Branch') {
            steps {
                script {
                    env.SOURCE_BRANCH = env.CHANGE_BRANCH ?: (env.BRANCH_NAME ?: sh(script: 'git branch --show-current', returnStdout: true).trim())
                    def allowed = env.CHANGE_ID || env.SOURCE_BRANCH == 'develop' || env.SOURCE_BRANCH == 'main' || env.SOURCE_BRANCH.startsWith('feature/')
                    if (!allowed) {
                        error("Blocked: branch '${env.SOURCE_BRANCH}' is not an allowed source. Use feature/*, develop, or main.")
                    }
                    if (env.CHANGE_ID && !env.SOURCE_BRANCH.startsWith('feature/')) {
                        error("Blocked: pull requests must come from feature/* branches; source was '${env.SOURCE_BRANCH}'.")
                    }
                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    echo "Source branch: ${env.SOURCE_BRANCH}"
                    echo "Versioned image: ${env.IMAGE_REPO}:${env.IMAGE_TAG}"
                }
            }
        }

        stage('Python Checks') {
            steps {
                sh '''
                    set -eu
                    python3 --version
                    python3 -m py_compile app.py
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv(SONARQUBE_SERVER) {
                    sh '''
                        set -eu
                        /opt/sonar-scanner/bin/sonar-scanner \
                          -Dsonar.projectKey=python-demo \
                          -Dsonar.projectName=python-demo \
                          -Dsonar.sources=app.py \
                          -Dsonar.python.version=3.12
                    '''
                }
            }
        }

        stage('Enforce SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build and Smoke Test Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build --pull -t ${IMAGE_REPO}:${IMAGE_TAG} .
                    docker rm -f python-demo-test >/dev/null 2>&1 || true
                    docker run -d --name python-demo-test -p 127.0.0.1:15000:5000 ${IMAGE_REPO}:${IMAGE_TAG}
                    trap 'docker rm -f python-demo-test >/dev/null 2>&1 || true' EXIT
                    attempt=0
                    until [ "$(curl --silent --output /tmp/python-demo-response.txt --write-out '%{http_code}' http://127.0.0.1:15000/)" = '200' ]; do
                      attempt=$((attempt + 1))
                      if [ "$attempt" -ge 20 ]; then
                        docker logs python-demo-test
                        exit 1
                      fi
                      sleep 2
                    done
                    grep -F 'Hello from Kubernetes!' /tmp/python-demo-response.txt
                '''
            }
        }

        stage('Publish Versioned Image') {
            when {
                expression { !env.CHANGE_ID && (env.BRANCH_NAME == 'develop' || env.BRANCH_NAME == 'main') }
            }
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKER_USER',
                        passwordVariable: 'DOCKER_PASS'
                    )
                ]) {
                    sh '''
                        set -eu
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        trap 'docker logout >/dev/null 2>&1 || true' EXIT
                        docker tag ${IMAGE_REPO}:${IMAGE_TAG} "${IMAGE_REPO}:${IMAGE_TAG}"
                        docker push "${IMAGE_REPO}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Deploy DEV') {
            when { expression { !env.CHANGE_ID && env.BRANCH_NAME == 'develop' } }
            steps {
                sh '''
                    set -eu
                    kubectl apply -f k8s/namespaces.yaml
                    sed 's|NAMESPACE_PLACEHOLDER|dev|g; s|IMAGE_PLACEHOLDER|${IMAGE_REPO}:${IMAGE_TAG}|g' k8s/deployment.yaml | kubectl apply -f -
                    sed 's|NAMESPACE_PLACEHOLDER|dev|g' k8s/service.yaml | kubectl apply -f -
                    kubectl rollout status deployment/python-demo -n dev --timeout=180s
                    kubectl get pods,svc -n dev
                '''
            }
        }

        stage('Deploy QA') {
            when { expression { !env.CHANGE_ID && env.BRANCH_NAME == 'main' } }
            steps {
                sh '''
                    set -eu
                    kubectl apply -f k8s/namespaces.yaml
                    sed 's|NAMESPACE_PLACEHOLDER|qa|g; s|IMAGE_PLACEHOLDER|${IMAGE_REPO}:${IMAGE_TAG}|g' k8s/deployment.yaml | kubectl apply -f -
                    sed 's|NAMESPACE_PLACEHOLDER|qa|g' k8s/service.yaml | kubectl apply -f -
                    kubectl rollout status deployment/python-demo -n qa --timeout=180s
                    kubectl get pods,svc -n qa
                '''
            }
        }

        stage('QA Sign-off') {
            when { expression { !env.CHANGE_ID && env.BRANCH_NAME == 'main' } }
            steps {
                timeout(time: 2, unit: 'HOURS') {
                    input message: "QA has tested ${IMAGE_REPO}:${IMAGE_TAG}. Confirm QA passed before production?", ok: 'QA passed', submitter: 'qa-team'
                }
            }
        }

        stage('Approve Production') {
            when { expression { !env.CHANGE_ID && env.BRANCH_NAME == 'main' } }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Deploy the same tested image ${IMAGE_REPO}:${IMAGE_TAG} to production?", ok: 'Approve PROD', submitter: 'release-managers'
                }
            }
        }

        stage('Deploy PROD') {
            when { expression { !env.CHANGE_ID && env.BRANCH_NAME == 'main' } }
            steps {
                sh '''
                    set -eu
                    sed 's|NAMESPACE_PLACEHOLDER|prod|g; s|IMAGE_PLACEHOLDER|${IMAGE_REPO}:${IMAGE_TAG}|g' k8s/deployment.yaml | kubectl apply -f -
                    sed 's|NAMESPACE_PLACEHOLDER|prod|g' k8s/service.yaml | kubectl apply -f -
                    kubectl rollout status deployment/python-demo -n prod --timeout=180s
                    kubectl get pods,svc -n prod
                '''
            }
        }
    }

    post {
        success {
            echo "CI/CD succeeded: source=${SOURCE_BRANCH}, image=${IMAGE_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo 'CI/CD failed. Later deployment stages will not run.'
        }
        always {
            sh 'docker rm -f python-demo-test >/dev/null 2>&1 || true'
        }
    }
}