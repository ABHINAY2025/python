# Hello World CI/CD Branch Protection Demo

A small Flask Hello World application used to demonstrate safe CI/CD promotion with Jenkins, SonarQube, Docker Hub, and Kubernetes.

## Deployment policy

| Source | Pipeline behavior |
| --- | --- |
| `feature/*` | Run Python checks, SonarQube analysis, quality gate, Docker build, and HTTP smoke test. Never publish or deploy. |
| `develop` | Run all checks, push a unique versioned Docker image, and deploy to the `dev` namespace. |
| `main` | Run all checks, push a unique versioned image, deploy it to `qa`, wait for QA sign-off and production approval, then deploy the same image tag to `prod`. |
| Any other branch | Jenkins rejects the branch before build/deployment stages. |

Pull requests are intentionally limited to `feature/* -> develop` and `develop -> main`. The production rollout is not automatic: QA sign-off and an authorized production approver are required.

Image tags use `<Jenkins build number>-<short Git commit SHA>`, for example `42-a1b2c3d`. Pipeline deployments never use `latest`, so the QA and PROD stages in a single main-branch run use the same versioned image tag.

## Repository layout

```text
app.py
Dockerfile
requirements.txt
Jenkinsfile
k8s/
  namespaces.yaml
  deployment.yaml
  service.yaml
```

## Jenkins prerequisites

1. Create a **Multibranch Pipeline** job for `https://github.com/ABHINAY2025/python`. Enable branch discovery and pull-request discovery, and use `Jenkinsfile` as the pipeline script path.
2. On the build agent, install Python 3, Docker CLI with permission to use the Docker daemon, `curl`, `sed`, and `kubectl`.
3. In Jenkins credentials, create a username/password credential with ID `dockerhub-creds`. Its account must be allowed to push to Docker Hub repository `abhinayma/python-demo`.
4. In **Manage Jenkins → System**, configure the SonarQube server with the name `SonarQube`. Ensure `/opt/sonar-scanner/bin/sonar-scanner` exists on the agent.
5. In SonarQube, configure a webhook to `${JENKINS_URL}/sonarqube-webhook/`. Jenkins waits for the quality gate, so the webhook must reach Jenkins.
6. Configure Jenkins Kubernetes credentials/context so the build agent can create namespaces and deploy workloads to the target cluster. The cluster must be reachable from the agent.
7. Configure the Jenkins authorization strategy so the approver IDs/groups `qa-team` and `release-managers` are valid. Alternatively, edit the `submitter` values in `Jenkinsfile` to the actual Jenkins user IDs or groups.

## Create the branches

Start with `main` as the stable branch. Create `develop` once, then have developers work in feature branches:

```bash
git clone https://github.com/ABHINAY2025/python.git
cd python
git checkout main
git pull origin main
git checkout -b develop
git push -u origin develop
git checkout -b feature/hello-change
```

Push feature work to its own branch and open a pull request to `develop`. After DEV validation, open a pull request from `develop` to `main` to start the QA-to-production release process.

## Configure GitHub branch protection

In **Settings → Rules → Rulesets** (or **Branches → Branch protection rules**), configure rules for `main` and `develop`: require pull requests, require the Jenkins CI status check, block force pushes and branch deletion, and restrict bypass access to trusted administrators. Require at least one independent approval for `main`; do not let ordinary developers bypass the rules.

Branch protection is a GitHub repository setting and cannot be enforced by a `Jenkinsfile` alone. Jenkins branch checks are a second layer: they control what the pipeline deploys, while GitHub rules control who can update protected branches.

## Kubernetes resources and access

The pipeline applies `k8s/namespaces.yaml` to create the `dev`, `qa`, and `prod` namespaces, then renders `k8s/deployment.yaml` with the selected namespace and image tag. Each environment has its own Deployment and ClusterIP Service. Use an Ingress or `kubectl port-forward` to access the application; the service is not exposed publicly by default.

Example for testing DEV from a machine with cluster access:

```bash
kubectl port-forward -n dev svc/python-demo-service 5000:80
curl http://127.0.0.1:5000/
```

Expected response: `Hello from Kubernetes!`

## Rollback

Review the Deployment history, then roll back the affected namespace if a release fails:

```bash
kubectl rollout history deployment/python-demo -n prod
kubectl rollout undo deployment/python-demo -n prod
kubectl rollout status deployment/python-demo -n prod
```

## Important notes

- `Jenkinsfile` must run as a Multibranch Pipeline so `BRANCH_NAME`, `CHANGE_BRANCH`, and `CHANGE_TARGET` are populated reliably.
- Docker Hub tags are unique per pipeline run by convention; restrict registry write access and do not overwrite published version tags.
- The example protects the workflow but is not a replacement for a production security review, registry permissions, Kubernetes RBAC, secret management, or cluster-specific rollout/rollback testing.
