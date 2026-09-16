# Seclock

Seclock is a legally-aware digital inheritance and emergency access vault application designed to securely manage digital assets and provide controlled access during emergency or inheritance scenarios.

## Architecture / Deployment Flow

```text
                         GitHub
                           │
                           ▼
                        Jenkins
                           │
          ┌────────────────┼────────────────┐
          ▼                ▼                ▼
      Gitleaks          Semgrep          Trivy
      Secret Scan       SAST             SCA
          │                │                │
          └────────────────┼────────────────┘
                           ▼
                     Docker Build
                           │
                           ▼
                    Trivy Image Scan
                           │
                           ▼
                         ECR
                           │
                           ▼
              Update Kubernetes Manifest
                           │
                           ▼
                    Push to GitHub
                           │
                           ▼
                       Argo CD
                           │
                           ▼
                         EKS
                           │
                           ▼
                   Seclock Application
```

## Project Structure

```text
seclock/
├── audit_ledger.py
├── crypto_engine.py
├── generate_certificates.py
├── main.py
├── ocr_engine.py
├── requirements.txt
├── run.sh
├── Dockerfile
├── Jenkinsfile
├── k8s/
│   ├── deployment.yaml
│   └── service.yaml
├── sample_certificates/
└── static/
```

## Application Setup

### 1. Clone the Repository

```bash
git clone https://github.com/stelabiju/seclock.git
cd seclock
```

### 2. Install Dependencies

```bash
pip install -r requirements.txt
```

### 3. Run the Application Locally

The application runs using FastAPI and Uvicorn on port `8080`.

```bash
uvicorn main:app --host 0.0.0.0 --port 8080
```

The application can then be accessed at:

```text
http://localhost:8080
```

FastAPI documentation:

```text
http://localhost:8080/docs
```

## Docker Execution

### 4. Build the Docker Image

```bash
docker build -t seclock:latest .
```

### 5. Run the Container

```bash
docker run --rm -p 8080:8080 seclock:latest
```

The application is available at:

```text
http://localhost:8080
```

## Security Scanning

### 6. Secret Scanning with Gitleaks

Gitleaks is executed from the Jenkins pipeline against the checked-out repository.

```bash
docker run --rm \
  -v "$WORKSPACE:/path" \
  zricethezav/gitleaks:latest \
  detect --source /path --redact -v || true
```

### 7. SAST Scan with Semgrep

```bash
docker run --rm \
  -v "$WORKSPACE:/src" \
  semgrep/semgrep \
  semgrep scan --config auto /src
```

### 8. Filesystem Vulnerability Scan with Trivy

```bash
docker run --rm \
  -v "$WORKSPACE:/src" \
  aquasec/trivy:latest \
  fs --scanners vuln /src
```

### 9. Build the Versioned Docker Image

Jenkins builds the image using the Jenkins build number:

```bash
docker build -t seclock:${BUILD_NUMBER} .
```

### 10. Scan the Docker Image

The Docker socket is mounted so Trivy can scan the locally built image:

```bash
docker run --rm \
  -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy:latest \
  image seclock:${BUILD_NUMBER}
```

## Amazon ECR

### 11. Create the ECR Repository

```bash
aws ecr create-repository \
  --repository-name seclock \
  --region us-east-1
```

The repository is:

```text
457660516559.dkr.ecr.us-east-1.amazonaws.com/seclock
```

### 12. Authenticate Docker with ECR

```bash
aws ecr get-login-password --region us-east-1 | \
docker login --username AWS --password-stdin \
457660516559.dkr.ecr.us-east-1.amazonaws.com
```

### 13. Tag the Image

```bash
docker tag seclock:${BUILD_NUMBER} \
457660516559.dkr.ecr.us-east-1.amazonaws.com/seclock:${BUILD_NUMBER}
```

### 14. Push the Image

```bash
docker push \
457660516559.dkr.ecr.us-east-1.amazonaws.com/seclock:${BUILD_NUMBER}
```

## Amazon EKS

### 15. Create the EKS Cluster

```bash
eksctl create cluster \
  --name seclock-cluster \
  --region us-east-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --managed
```

### 16. Verify the Cluster

```bash
kubectl get nodes
```

The nodes should show:

```text
Ready
```

## Kubernetes Deployment

The Kubernetes manifests are stored under:

```text
k8s/
├── deployment.yaml
└── service.yaml
```

The application is deployed in:

```text
seclock-prod
```

Check the deployment:

```bash
kubectl get deployments -n seclock-prod
```

Check the pods:

```bash
kubectl get pods -n seclock-prod
```

Check the service:

```bash
kubectl get svc -n seclock-prod
```

## Argo CD

### 17. Install Argo CD

Create the namespace:

```bash
kubectl create namespace argocd
```

Install Argo CD:

```bash
kubectl apply --server-side -n argocd \
-f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml
```

Verify the installation:

```bash
kubectl get pods -n argocd
```

### 18. Create the Argo CD Application

The Argo CD Application points to the GitHub repository and the `k8s` directory.

```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: seclock
  namespace: argocd

spec:
  project: default

  source:
    repoURL: https://github.com/stelabiju/seclock.git
    targetRevision: main
    path: k8s

  destination:
    server: https://kubernetes.default.svc
    namespace: seclock-prod

  syncPolicy:
    automated:
      prune: true
      selfHeal: true

    syncOptions:
      - CreateNamespace=true
```

Verify the application:

```bash
kubectl get application seclock -n argocd
```

## Jenkins GitOps Deployment

After the Docker image is pushed to ECR, Jenkins automatically updates the image reference in:

```text
k8s/deployment.yaml
```

The image is updated using:

```bash
sed -i "s|image:.*|image: ${ECR_REPO_URI}:${BUILD_NUMBER}|" \
k8s/deployment.yaml
```

Jenkins then commits the updated manifest:

```bash
git add k8s/deployment.yaml
git commit -m "Update Seclock image to ${BUILD_NUMBER}"
```

and pushes the change back to:

```text
https://github.com/stelabiju/seclock.git
```

Argo CD detects the Git change and automatically synchronizes the Kubernetes deployment.

## Deployment Verification

After Argo CD synchronizes the new image:

```bash
kubectl get application seclock -n argocd
```

```bash
kubectl get pods -n seclock-prod
```

```bash
kubectl get deployment seclock -n seclock-prod
```

The Seclock pods should reach:

```text
Running
```

## Final Deployment Flow

```text
Developer pushes code
        ↓
      GitHub
        ↓
      Jenkins
        ↓
 Security Scans
        ↓
   Docker Build
        ↓
   Image Scan
        ↓
      ECR
        ↓
Jenkins updates
deployment.yaml
        ↓
      GitHub
        ↓
      Argo CD
        ↓
       EKS
        ↓
 Seclock Application
```

## Result

The project implements an automated CI/CD and GitOps workflow where application changes are:

1. Checked out from GitHub.
2. Scanned for secrets and security issues.
3. Built into a Docker image.
4. Scanned for container vulnerabilities.
5. Pushed to Amazon ECR.
6. Automatically referenced in the Kubernetes deployment manifest.
7. Committed back to GitHub.
8. Detected and synchronized by Argo CD.
9. Deployed to Amazon EKS.
10. Verified through the running Kubernetes pods.
