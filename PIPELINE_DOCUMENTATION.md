# Movie Picture CI/CD Pipeline Documentation

This document provides complete instructions and architectural documentation for the Continuous Integration (CI) and Continuous Deployment (CD) pipelines built with GitHub Actions for the **Movie Picture** application.

---

## 1. Pipeline Architecture

The application comprises two decoupled microservices:
1. **Frontend Application**: React SPA written in TypeScript/JavaScript, served via static file server inside Docker on port `3000`.
2. **Backend Application**: Python Flask REST API served with `uwsgi` inside Docker on port `5000`.

### Workflows Matrix

| Workflow Name | File Path | Trigger Event | Path Filter | Parallel Jobs | Dependent Jobs |
| :--- | :--- | :--- | :--- | :--- | :--- |
| **Frontend Continuous Integration** | `.github/workflows/frontend-ci.yaml` | `pull_request` (target: `main`), `workflow_dispatch` | `starter/frontend/**` | `lint`, `test` | `build` (needs `lint`, `test`) |
| **Backend Continuous Integration** | `.github/workflows/backend-ci.yaml` | `pull_request` (target: `main`), `workflow_dispatch` | `starter/backend/**` | `lint`, `test` | `build` (needs `lint`, `test`) |
| **Frontend Continuous Deployment** | `.github/workflows/frontend-cd.yaml` | `push` (target: `main`), `workflow_dispatch` | `starter/frontend/**` | `lint`, `test` | `deploy` (needs `lint`, `test`) |
| **Backend Continuous Deployment** | `.github/workflows/backend-cd.yaml` | `push` (target: `main`), `workflow_dispatch` | `starter/backend/**` | `lint`, `test` | `deploy` (needs `lint`, `test`) |

---

## 2. Infrastructure Setup (Terraform & AWS EKS)

The underlying cloud infrastructure (VPC, EKS cluster, node groups, and ECR repositories) is provisioned using Terraform.

### Step 2.1: Initialize and Apply Terraform
```bash
cd setup/terraform
terraform init
terraform apply -auto-approve
```

### Step 2.2: Note Terraform Outputs
Retrieve outputs generated after provisioning:
```bash
terraform output
```
Key outputs:
- `cluster_name`: `cluster`
- `frontend_ecr`: `<account-id>.dkr.ecr.us-east-1.amazonaws.com/frontend`
- `backend_ecr`: `<account-id>.dkr.ecr.us-east-1.amazonaws.com/backend`
- `github_action_user_arn`: ARN of the dedicated `github-action-user`

### Step 2.3: Authorize GitHub Actions IAM User in Kubernetes
Run the setup helper script to grant `system:masters` permissions to `github-action-user`:
```bash
aws eks update-kubeconfig --name cluster --region us-east-1
cd setup
chmod +x init.sh
./init.sh
```

---

## 3. GitHub Secrets Configuration

To comply with security requirements, no AWS credentials or secrets are committed to version control.
Navigate to your GitHub repository: **Settings > Secrets and variables > Actions** and add the following repository secrets:

| Secret Name | Description | Example / Source |
| :--- | :--- | :--- |
| `AWS_ACCESS_KEY_ID` | Access key for `github-action-user` | Generated under IAM > Users > github-action-user > Security Credentials |
| `AWS_SECRET_ACCESS_KEY` | Secret key for `github-action-user` | Generated alongside Access Key ID |
| `AWS_REGION` *(optional)* | AWS deployment region (defaults to `us-east-1`) | `us-east-1` |
| `EKS_CLUSTER_NAME` *(optional)* | EKS Cluster name (defaults to `cluster`) | `cluster` |
| `REACT_APP_MOVIE_API_URL` *(optional)* | Backend API URL for frontend Docker build | `http://<backend-k8s-loadbalancer-dns>` or `http://localhost:5000` |

---

## 4. Pipeline Details

### 4.1 Frontend CI (`frontend-ci.yaml`)
- **Lint Job**: Runs `npm run lint` using ESLint/Prettier. Utilizes `actions/cache@v4` to cache `~/.npm`.
- **Test Job**: Runs `npm run test` with `CI: true`. Runs concurrently with `lint`.
- **Build Job**: Executes only after `lint` and `test` pass (`needs: [lint, test]`). Builds Docker container passing `REACT_APP_MOVIE_API_URL` via `--build-arg`.

### 4.2 Backend CI (`backend-ci.yaml`)
- **Lint Job**: Runs `pipenv run lint` with flake8. Utilizes `actions/cache@v4` to cache `~/.local/share/virtualenvs`.
- **Test Job**: Runs `pipenv run test` with pytest. Runs concurrently with `lint`.
- **Build Job**: Executes only after `lint` and `test` pass (`needs: [lint, test]`). Builds Docker container `mp-backend:latest`.

### 4.3 Frontend CD (`frontend-cd.yaml`)
- Runs `lint` and `test` jobs first.
- **Deploy Job** (`needs: [lint, test]`):
  1. Configures AWS authentication securely using `aws-actions/configure-aws-credentials@v4`.
  2. Logs into Amazon ECR via `aws-actions/amazon-ecr-login@v2`.
  3. Builds Docker image with `--build-arg REACT_APP_MOVIE_API_URL`.
  4. Tags image with `${{ github.sha }}` and `latest`.
  5. Pushes Docker images to ECR repository (`frontend`).
  6. Sets up `kustomize` via `imranismail/setup-kustomize@v3`.
  7. Connects to EKS cluster with `aws eks update-kubeconfig`.
  8. Modifies manifest image tag: `kustomize edit set image frontend=<ECR_REPO_URL>:<GITHUB_SHA>`.
  9. Deploys updated manifests to cluster: `kustomize build | kubectl apply -f -`.
  10. Verifies rollout: `kubectl rollout status deployment/frontend --timeout=180s`.

### 4.4 Backend CD (`backend-cd.yaml`)
- Runs `lint` and `test` jobs first.
- **Deploy Job** (`needs: [lint, test]`):
  1. Configures AWS authentication securely.
  2. Logs into Amazon ECR.
  3. Builds Docker image and tags with `${{ github.sha }}` and `latest`.
  4. Pushes Docker images to ECR repository (`backend`).
  5. Sets up `kustomize`.
  6. Connects to EKS cluster.
  7. Modifies manifest image tag: `kustomize edit set image backend=<ECR_REPO_URL>:<GITHUB_SHA>`.
  8. Deploys updated manifests to cluster: `kustomize build | kubectl apply -f -`.
  9. Verifies rollout: `kubectl rollout status deployment/backend --timeout=180s`.

---

## 5. Simulating Failures (Pipeline Gatekeeping Verification)

Per rubric requirements, verify that the pipelines reliably halt and fail when code quality checks or tests fail:

### Frontend Test Failure Simulation:
In `starter/frontend`:
```bash
FAIL_TEST=true CI=true npm test
```
In git / CI: Commit a failing expectation in `starter/frontend/src/components/__tests__/App.test.js` or set `FAIL_TEST=true` in step env. The test job will fail and prevent Docker build or deployment.

### Frontend Lint Failure Simulation:
```bash
FAIL_LINT=true npm run lint
```
Or introduce an unused variable or formatting error in `starter/frontend/src/App.js`. The lint job will fail and abort the pipeline.

### Backend Test Failure Simulation:
In `starter/backend`:
```bash
FAIL_TEST=true pipenv run test
```
Setting `FAIL_TEST=true` causes `test_movies_endpoint_returns_200` to fail assertion, blocking the build/deploy step.

### Backend Lint Failure Simulation:
```bash
pipenv run lint-fail
```
Overrides max line length to 88 characters and fails on any lines >88 characters.

---

## 6. Verifying Cluster Deployments

Once both CD pipelines run successfully on merges to `main`:

### 6.1 Check Kubernetes Resources
```bash
aws eks update-kubeconfig --name cluster --region us-east-1

# Verify pods are Running
kubectl get pods -o wide

# Verify services and LoadBalancer endpoints
kubectl get svc
```

### 6.2 Test Backend API
Get the external hostname of the backend LoadBalancer:
```bash
BACKEND_URL=$(kubectl get svc backend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
curl http://${BACKEND_URL}/movies
```
Expected output:
```json
{"movies":[{"id":"123","title":"Top Gun: Maverick"},{"id":"456","title":"Sonic the Hedgehog"},{"id":"789","title":"A Quiet Place"}]}
```

### 6.3 Test Frontend UI
Get the external hostname of the frontend LoadBalancer:
```bash
FRONTEND_URL=$(kubectl get svc frontend -o jsonpath='{.status.loadBalancer.ingress[0].hostname}')
echo "Frontend available at: http://${FRONTEND_URL}"
```
Open `http://${FRONTEND_URL}` in your web browser. The movie catalog should load and display movie titles dynamically fetched from the backend API.

---

## 7. Resource Teardown

To avoid unnecessary AWS charges after project submission or testing:
```bash
cd setup/terraform
terraform destroy -auto-approve
```