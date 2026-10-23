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
        stage('Validate Branch and Version') {
            steps {
                script {
                    env.GIT_BRANCH_NAME = env.CHANGE_BRANCH ?: (env.BRANCH_NAME ?: sh(
                        script: 'git branch --show-current',
                        returnStdout: true
                    ).trim())

                    if (!(env.GIT_BRANCH_NAME == 'develop' ||
                          env.GIT_BRANCH_NAME == 'main' ||
                          env.GIT_BRANCH_NAME.startsWith('feature/') ||
                          env.GIT_BRANCH_NAME.startsWith('release/'))) {
                        error("Branch '${env.GIT_BRANCH_NAME}' is not allowed by this pipeline.")
                    }

                    env.IMAGE_TAG = "${env.BUILD_NUMBER}-${env.GIT_COMMIT.take(7)}"
                    echo "Branch: ${env.GIT_BRANCH_NAME}"
                    echo "Image tag: ${env.IMAGE_TAG}"
                }
            }
        }

        stage('Python Checks') {
            steps {
                sh '''
                    set -eu
                    python3 --version
                    python3 -m py_compile app.py
                    python3 -m pip install --disable-pip-version-check -r requirements.txt
                    python3 - <<'PY'
from app import app
client = app.test_client()
response = client.get('/')
assert response.status_code == 200, response.status_code
assert response.get_data(as_text=True) == "Hello from Kubernetes!", response.get_data(as_text=True)
print("Hello World route test passed")
PY
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

        stage('Build and Test Docker Image') {
            steps {
                sh '''
                    set -eu
                    docker build --pull -t ${IMAGE_REPO}:${IMAGE_TAG} .
                    docker rm -f python-demo-test >/dev/null 2>&1 || true
                    docker run -d --name python-demo-test -p 127.0.0.1:15000:5000 ${IMAGE_REPO}:${IMAGE_TAG}
                    trap 'docker rm -f python-demo-test >/dev/null 2>&1 || true' EXIT
                    attempt=0
                    until curl --fail --silent http://127.0.0.1:15000/; do
                      attempt=$((attempt + 1))
                      if [ "$attempt" -ge 20 ]; then
                        docker logs python-demo-test
                        exit 1
                      fi
                      sleep 2
                    done
                '''
            }
        }

        stage('Publish Immutable Image') {
            when {
                anyOf {
                    branch 'develop'
                    branch 'main'
                    branch pattern: 'release/.+', comparator: 'REGEXP'
                }
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
                        docker tag ${IMAGE_REPO}:${IMAGE_TAG} "$DOCKER_USER/${APP_NAME}:${IMAGE_TAG}"
                        docker push "$DOCKER_USER/${APP_NAME}:${IMAGE_TAG}"
                    '''
                }
            }
        }

        stage('Deploy DEV') {
            when { branch 'develop' }
            steps {
                sh '''
                    set -eu
                    kubectl apply -f k8s/dev/
                    kubectl set image deployment/python-demo python-demo=${IMAGE_REPO}:${IMAGE_TAG} -n dev
                    kubectl rollout status deployment/python-demo -n dev --timeout=180s
                '''
            }
        }

        stage('Deploy QA') {
            when {
                branch pattern: 'release/.+', comparator: 'REGEXP'
            }
            steps {
                sh '''
                    set -eu
                    kubectl apply -f k8s/qa/
                    kubectl set image deployment/python-demo python-demo=${IMAGE_REPO}:${IMAGE_TAG} -n qa
                    kubectl rollout status deployment/python-demo -n qa --timeout=180s
                '''
            }
        }

        stage('Approve Production') {
            when { branch 'main' }
            steps {
                timeout(time: 30, unit: 'MINUTES') {
                    input message: "Deploy ${IMAGE_REPO}:${IMAGE_TAG} to production?",
                          ok: 'Approve PROD',
                          submitter: 'release-managers'
                }
            }
        }

        stage('Deploy PROD') {
            when { branch 'main' }
            steps {
                sh '''
                    set -eu
                    kubectl apply -f k8s/prod/
                    kubectl set image deployment/python-demo python-demo=${IMAGE_REPO}:${IMAGE_TAG} -n prod
                    kubectl rollout status deployment/python-demo -n prod --timeout=180s
                '''
            }
        }
    }

    post {
        success {
            echo "Pipeline succeeded: branch=${GIT_BRANCH_NAME}, image=${IMAGE_REPO}:${IMAGE_TAG}"
        }
        failure {
            echo "Pipeline failed. No later stages should be treated as approved."
        }
        always {
            sh 'docker rm -f python-demo-test >/dev/null 2>&1 || true'
        }
    }
}