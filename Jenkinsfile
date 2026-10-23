pipeline {
    agent any

    options {
        timestamps()
        disableConcurrentBuilds()
        skipDefaultCheckout()
        buildDiscarder(logRotator(numToKeepStr: '30'))
    }

    environment {
        IMAGE_REPO = 'abhinayma/python-demo'
        SONARQUBE_SERVER = 'SonarQube'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Validate Branch') {
            steps {
                script {
                    env.SOURCE_BRANCH = env.CHANGE_BRANCH ?: (env.BRANCH_NAME ?: sh(script: 'git branch --show-current', returnStdout: true).trim())
                    if (env.CHANGE_ID) {
                        def featureToDevelop = env.SOURCE_BRANCH.startsWith('feature/') && env.CHANGE_TARGET == 'develop'
                        def developToMain = env.SOURCE_BRANCH == 'develop' && env.CHANGE_TARGET == 'main'
                        if (!(featureToDevelop || developToMain)) {
                            error("Blocked pull request: only feature/* -> develop and develop -> main are allowed. Source='${env.SOURCE_BRANCH}', target='${env.CHANGE_TARGET}'.")
                        }
                    } else {
                        def allowedBranch = env.SOURCE_BRANCH == 'develop' || env.SOURCE_BRANCH == 'main' || env.SOURCE_BRANCH.startsWith('feature/')
                        if (!allowedBranch) {
                            error("Blocked: branch '${env.SOURCE_BRANCH}' is not allowed. Use feature/*, develop, or main.")
                        }
                    }
                    env.IMAGE_CHANNEL = env.BRANCH_NAME == 'develop' ? 'dev' : (env.BRANCH_NAME == 'main' ? 'main' : 'ci')
                    env.IMAGE_TAG = "${env.IMAGE_CHANNEL}-${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    env.TEST_CONTAINER = "python-demo-test-${env.JOB_NAME}-${env.BUILD_NUMBER}".toLowerCase().replaceAll(/[^a-z0-9_.-]/, '-')
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
                withSonarQubeEnv(env.SONARQUBE_SERVER) {
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
                    docker rm -f "${TEST_CONTAINER}" >/dev/null 2>&1 || true
                    docker run -d --name "${TEST_CONTAINER}" -p 127.0.0.1::5000 "${IMAGE_REPO}:${IMAGE_TAG}"
                    trap 'docker rm -f "${TEST_CONTAINER}" >/dev/null 2>&1 || true; rm -f "/tmp/${TEST_CONTAINER}.response.txt"' EXIT
                    TEST_PORT="$(docker port "${TEST_CONTAINER}" 5000/tcp | sed -n '1s/.*://p')"
                    if [ -z "$TEST_PORT" ]; then
                      docker logs "${TEST_CONTAINER}"
                      exit 1
                    fi
                    attempt=0
                    until [ "$(curl --silent --output "/tmp/${TEST_CONTAINER}.response.txt" --write-out '%{http_code}' "http://127.0.0.1:${TEST_PORT}/")" = '200' ]; do
                      attempt=$((attempt + 1))
                      if [ "$attempt" -ge 20 ]; then
                        docker logs "${TEST_CONTAINER}"
                        exit 1
                      fi
                      sleep 2
                    done
                    grep -F 'Hello from Kubernetes!' "/tmp/${TEST_CONTAINER}.response.txt"
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
            sh 'if [ -n "$TEST_CONTAINER" ]; then docker rm -f "$TEST_CONTAINER" >/dev/null 2>&1 || true; fi'
        }
    }
}