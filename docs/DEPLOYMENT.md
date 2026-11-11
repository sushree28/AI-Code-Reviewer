# Deployment Guide

Full walkthrough for standing up AI Code Reviewer on AWS: accounts, infrastructure, the GitHub App, Kubernetes, monitoring, and teardown. Commands below are shown for bash/zsh (macOS, Linux, WSL, or Git Bash on Windows).

Replace `YOUR_USERNAME`, `YOUR_GITHUB_USERNAME`, and any placeholder values (account IDs, ARNs, endpoints) with your own throughout.

## Table of Contents

1. [Accounts You Need](#1-accounts-you-need)
2. [Install Tools](#2-install-tools)
3. [AWS Account Setup](#3-aws-account-setup)
4. [GitHub Repository Setup](#4-github-repository-setup)
5. [Create the GitHub App](#5-create-the-github-app)
6. [LangFuse Setup](#6-langfuse-setup)
7. [OpenAI API Key](#7-openai-api-key)
8. [Provision AWS Infrastructure with Terraform](#8-provision-aws-infrastructure-with-terraform)
9. [Configure GitHub Secrets](#9-configure-github-secrets)
10. [First Deployment via GitHub Actions](#10-first-deployment-via-github-actions)
11. [Connect kubectl to EKS](#11-connect-kubectl-to-eks)
12. [Apply Kubernetes Manifests](#12-apply-kubernetes-manifests)
13. [Set Up Grafana and Prometheus](#13-set-up-grafana-and-prometheus)
14. [Set Up LangFuse Tracing](#14-set-up-langfuse-tracing)
15. [Update the GitHub App Webhook URL](#15-update-the-github-app-webhook-url)
16. [End-to-End Test](#16-end-to-end-test)
17. [Accessing Dashboards](#accessing-dashboards)
18. [How the Weekly Evaluation Works](#18-how-the-weekly-evaluation-works)
19. [Health Checks](#19-health-checks)
20. [Troubleshooting](#20-troubleshooting)
21. [Cost Estimate](#cost-estimate)
22. [Teardown](#22-teardown)

---

## 1. Accounts You Need

- **AWS** — https://aws.amazon.com. You'll be charged for usage (see [Cost Estimate](#cost-estimate)); choose Basic (free) support.
- **GitHub** — https://github.com, email verified.
- **OpenAI** — https://platform.openai.com. Add a payment method, at least $10 of credit, and set a monthly usage limit under Billing → Usage limits.
- **LangFuse** — https://langfuse.com. Free. Sign up, create a project named `ai-code-reviewer`, then go to Settings → API Keys → Create new API key and save both the Public Key (`pk-lf-...`) and Secret Key (`sk-lf-...`).

## 2. Install Tools

Install each tool for your platform using the official instructions, then verify:

| Tool | Install docs | Verify |
|---|---|---|
| AWS CLI v2 | https://docs.aws.amazon.com/cli/latest/userguide/getting-started-install.html | `aws --version` |
| Terraform | https://developer.hashicorp.com/terraform/install | `terraform --version` |
| kubectl | https://kubernetes.io/docs/tasks/tools/ | `kubectl version --client` |
| Git | https://git-scm.com/downloads | `git --version` |
| Helm | https://helm.sh/docs/intro/install/ | `helm version` |

On macOS, all five are available via Homebrew: `brew install awscli terraform kubernetes-cli git helm`.

## 3. AWS Account Setup

### 3.1 Create an IAM user for CLI access

1. AWS Console → IAM → Users → Create user
2. Username: `ai-reviewer-deployer`
3. Attach policy directly: `AdministratorAccess`
4. Create the user, open it → Security credentials → Access keys → Create access key → Command Line Interface (CLI)
5. **Copy the Access Key ID and Secret Access Key immediately** — the secret is shown only once.

### 3.2 Configure the AWS CLI

```bash
aws configure
```

```
AWS Access Key ID: <paste>
AWS Secret Access Key: <paste>
Default region name: us-east-1
Default output format: json
```

Verify:

```bash
aws sts get-caller-identity
```

### 3.3 Save your AWS account ID

```bash
aws sts get-caller-identity --query Account --output text
```

Save this 12-digit value as `AWS_ACCOUNT_ID`.

### 3.4 Create an OIDC identity provider for GitHub Actions

This lets GitHub Actions deploy to AWS without long-lived AWS keys stored in GitHub.

1. IAM → Identity providers → Add provider
2. Provider type: OpenID Connect
3. Provider URL: `https://token.actions.githubusercontent.com`
4. Click "Get thumbprint" (auto-fills)
5. Audience: `sts.amazonaws.com`
6. Add provider

### 3.5 Create an IAM role for GitHub Actions

1. IAM → Roles → Create role → Trusted entity type: Web identity
2. Identity provider: `token.actions.githubusercontent.com`, Audience: `sts.amazonaws.com`
3. Add condition: `StringLike`, key `token.actions.githubusercontent.com:sub`, value `repo:YOUR_GITHUB_USERNAME/ai-code-reviewer:*`
4. Attach policy: `AdministratorAccess`
5. Role name: `github-actions-ai-reviewer`
6. Copy the Role ARN (`arn:aws:iam::<account-id>:role/github-actions-ai-reviewer`) — save it as `AWS_ROLE_ARN`.

## 4. GitHub Repository Setup

### 4.1 Create the repository

GitHub → New repository → name `ai-code-reviewer` → Private → do **not** initialize with a README → Create.

### 4.2 Create a Personal Access Token

Settings → Developer settings → Personal access tokens (classic) → Generate new token with `repo` and `workflow` scopes. Save it — you'll use it as your Git password.

### 4.3 Push the project

```bash
cd ai-code-reviewer
git init
git add .
git commit -m "initial: full project setup"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/ai-code-reviewer.git
git push -u origin main
```

Use your GitHub username and the Personal Access Token (not your account password) when prompted.

## 5. Create the GitHub App

The GitHub App is the bot identity that reads PR diffs and posts review comments.

### 5.1 Create the app

Settings → Developer settings → GitHub Apps → New GitHub App.

- **Name:** `ai-code-reviewer-bot` (must be globally unique — append numbers if taken)
- **Homepage URL:** `https://github.com/YOUR_USERNAME/ai-code-reviewer`
- **Webhook active:** checked
- **Webhook URL:** `https://placeholder.com` (replaced in [step 15](#15-update-the-github-app-webhook-url) once deployed)
- **Webhook secret:** a long random string — save it as `GITHUB_WEBHOOK_SECRET`

### 5.2 Permissions and events

- Repository permissions → **Contents:** Read-only, **Pull requests:** Read and write
- Subscribe to events → **Pull request**
- Where can this app be installed → **Any account**

### 5.3 Save credentials

1. Create the app
2. Copy the **App ID** → save as `GITHUB_APP_ID`
3. Private keys → Generate a private key (downloads a `.pem` file)
4. Open the `.pem` file and copy its full contents, including the `BEGIN/END RSA PRIVATE KEY` lines → save as `GITHUB_APP_PRIVATE_KEY`

### 5.4 Install the app

App settings → Install App → choose your account → "Only select repositories" → select `ai-code-reviewer` (and any repo you want reviewed) → Install.

## 6. LangFuse Setup

LangFuse is already wired into the orchestrator — no code changes needed. Get your API keys from your `ai-code-reviewer` LangFuse project: Settings → API Keys → Create new API key. Save the Public Key (`pk-lf-...`) and Secret Key (`sk-lf-...`).

## 7. OpenAI API Key

Platform → API keys → Create new secret key → name it `ai-code-reviewer` → copy and save as `OPENAI_API_KEY`.

## 8. Provision AWS Infrastructure with Terraform

Creates the EKS cluster, RDS database, Redis, container registry, and S3 bucket. One-time.

```bash
cd infra/terraform
terraform init
```

Preview:

```bash
terraform plan \
  -var="cluster_name=ai-code-reviewer" \
  -var="db_password=YourStrongPassword123!" \
  -var="environment=production"
```

This creates: 1 VPC, 1 EKS cluster with 2 worker nodes, 1 RDS PostgreSQL instance, 1 ElastiCache Redis cluster, 1 S3 bucket, 5 ECR repositories.

Apply:

```bash
terraform apply \
  -var="cluster_name=ai-code-reviewer" \
  -var="db_password=YourStrongPassword123!" \
  -var="environment=production"
```

Type `yes` to confirm. **This takes 15–25 minutes** — the EKS cluster takes the longest.

Save every value from the output — you'll need them in later steps:

```bash
terraform output
```

```
eks_cluster_endpoint   = "https://XXXXX.gr7.us-east-1.eks.amazonaws.com"
rds_endpoint           = "ai-code-reviewer-postgres.XXXX.us-east-1.rds.amazonaws.com:5432"
redis_endpoint         = "ai-code-reviewer-redis.XXXX.cache.amazonaws.com:6379"
ecr_gateway_url        = "<account-id>.dkr.ecr.us-east-1.amazonaws.com/gateway"
ecr_webhook_url        = "<account-id>.dkr.ecr.us-east-1.amazonaws.com/webhook"
ecr_orchestrator_url   = "<account-id>.dkr.ecr.us-east-1.amazonaws.com/orchestrator"
ecr_reviewer_url       = "<account-id>.dkr.ecr.us-east-1.amazonaws.com/reviewer"
ecr_learner_url        = "<account-id>.dkr.ecr.us-east-1.amazonaws.com/learner"
```

## 9. Configure GitHub Secrets

Repo → Settings → Secrets and variables → Actions → New repository secret. Add exactly these three:

| Name | Value |
|---|---|
| `AWS_ACCOUNT_ID` | your 12-digit account ID (step 3.3) |
| `AWS_ROLE_ARN` | the role ARN from step 3.5 |
| `EKS_CLUSTER_NAME` | `ai-code-reviewer` |

> **Everything else stays out of GitHub Actions secrets.** `OPENAI_API_KEY`, `DATABASE_URL`, `LANGFUSE_PUBLIC_KEY`, `LANGFUSE_SECRET_KEY`, `GITHUB_APP_ID`, `GITHUB_APP_PRIVATE_KEY`, and `GITHUB_WEBHOOK_SECRET` are only ever written to `infra/k8s/secret.yaml` (step 12.2) and read by pods at runtime. GitHub also blocks secret names starting with `GITHUB_`. The pipelines only build/push images and run `kubectl set image` — they never touch the Kubernetes secret.

## 10. First Deployment via GitHub Actions

### How the pipelines work

Five independent pipelines, one per service — each triggers only on changes to its own `services/<name>/` directory, with the same three jobs: **test → build-and-push → deploy**.

| Pipeline | Triggers on changes in |
|---|---|
| `gateway.yml` | `services/gateway/` |
| `webhook.yml` | `services/webhook/` |
| `orchestrator.yml` | `services/orchestrator/` |
| `reviewer.yml` | `services/reviewer/` |
| `learner.yml` | `services/learner/` |

> **The first run's `deploy` job will fail with "deployment not found" — this is expected.** It requires the Kubernetes Deployments to exist first, which happens in [step 12](#12-apply-kubernetes-manifests). Re-trigger the pipelines afterward and all three jobs will pass.

### 10.1 Trigger all five pipelines

Each service directory has a `deploy.txt` used purely to trigger its pipeline without touching real code:

```bash
for s in gateway webhook orchestrator reviewer learner; do echo 1 > services/$s/deploy.txt; done
git add services/
git commit -m "deploy: trigger initial deployment for all services"
git push origin main
```

### 10.2 Watch them run

Repo → Actions tab. Five workflow runs start simultaneously. Each has three jobs:

1. **test** (1–2 min) — installs that service's dependencies, runs its test suite
2. **build-and-push** (3–5 min) — builds the Docker image, pushes to ECR tagged with the commit SHA
3. **deploy** (1 min) — runs `kubectl set image` against EKS

All three jobs across all five workflows should show green. If one fails, open it and read the failed step's log.

### 10.3 Re-trigger after Kubernetes is set up

Once [step 12](#12-apply-kubernetes-manifests) is complete, bump the `deploy.txt` files again and push — this lets the `deploy` job succeed now that the Deployments exist:

```bash
for s in gateway webhook orchestrator reviewer learner; do echo 2 > services/$s/deploy.txt; done
git add services/
git commit -m "deploy: trigger all pipelines after k8s setup"
git push origin main
```

### 10.4 Redeploying a single service later

Editing one service only triggers its own pipeline:

```bash
git add services/orchestrator/graph.py
git commit -m "fix: improve security agent prompt"
git push origin main
```

Only the orchestrator pipeline runs; the other four are untouched.

## 11. Connect kubectl to EKS

```bash
aws eks update-kubeconfig --name ai-code-reviewer --region us-east-1
kubectl get nodes
```

Both nodes should show `STATUS Ready` within a few minutes. If `kubectl` can't reach the cluster, re-check `aws sts get-caller-identity`.

## 12. Apply Kubernetes Manifests

### 12.1 Point the ConfigMap at your real Redis endpoint

`infra/k8s/configmap.yaml` ships with a placeholder `REDIS_URL`. Skipping this step means every service fails to reach Redis.

```bash
cd infra/terraform && terraform output redis_endpoint
```

Edit `infra/k8s/configmap.yaml`:

```yaml
REDIS_URL: "redis://<your-redis-endpoint>:6379/0"
```

### 12.2 Fill in the Kubernetes secret

Edit `infra/k8s/secret.yaml` with your real values:

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secrets
type: Opaque
stringData:
  GITHUB_WEBHOOK_SECRET: "<your webhook secret>"
  GITHUB_APP_ID: "<your app id>"
  GITHUB_APP_PRIVATE_KEY: |
    -----BEGIN RSA PRIVATE KEY-----
    ...full key, preserving line breaks...
    -----END RSA PRIVATE KEY-----
  OPENAI_API_KEY: "sk-proj-..."
  LANGFUSE_PUBLIC_KEY: "pk-lf-..."
  LANGFUSE_SECRET_KEY: "sk-lf-..."
  DATABASE_URL: "postgresql+asyncpg://dbadmin:YourStrongPassword123!@<rds-endpoint>/codereviewer"
```

`DATABASE_URL` must use the `postgresql+asyncpg://` driver — note this differs from any plain `postgresql://` string you may use elsewhere. Keep the private key's line breaks exactly as in the `.pem` file.

### 12.3 Apply everything, in order

```bash
cd infra/k8s

kubectl apply -f configmap.yaml
kubectl apply -f secret.yaml

# Run the DB migration and wait for it to complete
kubectl apply -f migration-job.yaml
kubectl wait --for=condition=complete job/db-migrate --timeout=120s
# If it times out: kubectl logs job/db-migrate

# Deploy all five services
kubectl apply -f gateway.yaml
kubectl apply -f webhook.yaml
kubectl apply -f orchestrator.yaml
kubectl apply -f reviewer.yaml
kubectl apply -f learner.yaml

# Celery workers
kubectl apply -f webhook-worker.yaml
kubectl apply -f learner-worker.yaml

# Autoscaler for the orchestrator
kubectl apply -f hpa.yaml
```

Install the AWS Load Balancer Controller (required for ingress — without it, the ingress never gets an address):

```bash
helm repo add eks https://aws.github.io/eks-charts
helm repo update
helm install aws-load-balancer-controller eks/aws-load-balancer-controller \
  -n kube-system \
  --set clusterName=ai-code-reviewer \
  --set serviceAccount.create=true

kubectl get pods -n kube-system   # wait for Running
```

Apply the ingress:

```bash
kubectl apply -f ingress.yaml
```

### 12.4 Verify

```bash
kubectl get pods
```

Expect all pods `Running` and `1/1` within 3–5 minutes:

```
gateway-7d4f9b-xxx          1/1   Running   0   3m
webhook-6c8d7b-xxx           1/1   Running   0   3m
webhook-worker-5f6g7h-xxx    1/1   Running   0   2m
orchestrator-9h2j3k-xxx      1/1   Running   0   3m
reviewer-4l5m6n-xxx          1/1   Running   0   3m
learner-7p8q9r-xxx           1/1   Running   0   3m
learner-worker-2s3t4u-xxx    1/1   Running   0   2m
```

Get the public URL (poll every minute until `ADDRESS` is populated — the ALB takes a few minutes to provision):

```bash
kubectl get ingress gateway-ingress
```

Your webhook endpoint is `https://<ADDRESS>/webhook/github`.

## 13. Set Up Grafana and Prometheus

```bash
helm repo add prometheus-community https://prometheus-community.github.io/helm-charts
helm repo add grafana https://grafana.github.io/helm-charts
helm repo update
```

### 13.1 Install the EBS CSI driver

Required for Prometheus's persistent storage.

```bash
aws eks list-nodegroups --cluster-name ai-code-reviewer --region us-east-1
aws eks describe-nodegroup --cluster-name ai-code-reviewer --nodegroup-name <nodegroup> --region us-east-1
# copy the nodeRole name from the output
aws iam attach-role-policy \
  --role-name <node-role-name> \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEBSCSIDriverPolicy

aws eks create-addon --cluster-name ai-code-reviewer --addon-name aws-ebs-csi-driver --region us-east-1

# poll until ACTIVE
aws eks describe-addon --cluster-name ai-code-reviewer --addon-name aws-ebs-csi-driver --region us-east-1 --query addon.status --output text
```

### 13.2 Install Prometheus

```bash
helm install prometheus prometheus-community/prometheus \
  --namespace monitoring --create-namespace \
  --set server.persistentVolume.size=8Gi \
  --set server.persistentVolume.storageClass=gp2 \
  --set alertmanager.persistence.storageClass=gp2

kubectl get pods -n monitoring   # wait for Running
```

Apply the project's scrape config:

```bash
kubectl create configmap prometheus-config \
  --from-file=prometheus.yml=monitoring/prometheus.yml \
  -n monitoring --dry-run=client -o yaml | kubectl apply -f -
```

### 13.3 Install Grafana

Grafana needs NLB annotations so the Load Balancer Controller assigns it a public address.

```bash
helm install grafana grafana/grafana \
  --namespace monitoring \
  --set persistence.enabled=true \
  --set persistence.size=5Gi \
  --set persistence.storageClassName=gp2 \
  --set service.type=LoadBalancer \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-type=external" \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-nlb-target-type=ip" \
  --set "service.annotations.service\.beta\.kubernetes\.io/aws-load-balancer-scheme=internet-facing"

kubectl get svc grafana -n monitoring   # wait for EXTERNAL-IP
```

Get the admin password (username is always `admin`):

```bash
kubectl get secret --namespace monitoring grafana -o jsonpath="{.data.admin-password}" | base64 --decode
```

### 13.4 Configure Grafana

1. Open `http://<EXTERNAL-IP>` and log in as `admin`
2. Configuration → Data sources → Add data source → Prometheus
3. URL: `http://prometheus-server.monitoring.svc.cluster.local` → Save & Test
4. "+" → Import → Upload JSON file → `monitoring/grafana-dashboard.json` → select the Prometheus data source → Import

The dashboard has one section per service, each with **Request Rate**, **p99 Latency**, and **Error Rate** panels, refreshing every 30 seconds.

## 14. Set Up LangFuse Tracing

LangFuse is already wired into the orchestrator; you just need the host region and valid keys.

1. Check your region after logging in at https://langfuse.com: `cloud.langfuse.com` = EU, `us.cloud.langfuse.com` = US
2. Set `LANGFUSE_HOST` in `infra/k8s/configmap.yaml` to match, then:

```bash
kubectl apply -f infra/k8s/configmap.yaml
kubectl rollout restart deployment orchestrator
```

3. The LangFuse keys live only in the `app-secrets` Kubernetes secret (step 12.2), never in GitHub Actions. To check or rotate them:

```bash
kubectl get secret app-secrets -o jsonpath='{.data.LANGFUSE_PUBLIC_KEY}' | base64 --decode

kubectl create secret generic app-secrets --dry-run=client -o yaml \
  --from-literal=LANGFUSE_PUBLIC_KEY=pk-lf-YOUR-KEY \
  --from-literal=LANGFUSE_SECRET_KEY=sk-lf-YOUR-KEY \
  | kubectl apply -f -
kubectl rollout restart deployment orchestrator
```

4. After a PR is reviewed, check for errors:

```bash
kubectl logs -l app=orchestrator --tail=50 | grep -i "langfuse\|401\|export"
```

`Failed to export span batch code: 401` means wrong keys or wrong host region.

5. In LangFuse, open your project → **Traces** to see one trace per PR, with the four parallel agent spans (static_analysis, security, style, architecture), each with its full prompt/response, token count, cost, and latency.

## 15. Update the GitHub App Webhook URL

```bash
kubectl get ingress gateway-ingress
```

Your webhook URL is `http://<ADDRESS>/webhook/github`. In the GitHub App settings, replace the placeholder `https://placeholder.com` with this value under Webhook URL, and leave the webhook secret unchanged. Save.

## 16. End-to-End Test

### 16.1 Install the app on a test repository

GitHub App settings → Install App → Only select repositories → choose a test repo → Install.

### 16.2 Confirm everything is healthy

```bash
kubectl get pods   # all Running
```

Re-trigger the pipelines if needed (see [10.3](#103-re-trigger-after-kubernetes-is-set-up)) and wait for all green before testing.

### 16.3 Open a test PR

On the test repository (via the GitHub web UI or git):

1. Create a branch, e.g. `test-ai-review`
2. Add a file `test_code.py`:
   ```python
   def calculate(a, b):
       password = "admin123"
       result = eval(a + b)
       return result
   ```
3. Commit, then open a PR against `main`

Within 30–60 seconds, `ai-code-reviewer-bot` should post inline comments — e.g. flagging the hardcoded password and the use of `eval()`. If comments appear, the system is working end-to-end.

## Accessing Dashboards

| Tool | How to reach it | Notes |
|---|---|---|
| **Grafana** | `kubectl get svc -n monitoring grafana` → open `http://<EXTERNAL-IP>` | Login `admin` / password from step 13.3 |
| **Prometheus** | `kubectl port-forward -n monitoring svc/prometheus-server 9090:80` → open `http://localhost:9090` | Try queries `up`, `http_requests_total`, `rate(http_requests_total[5m])` |
| **LangFuse** | https://cloud.langfuse.com (EU) or https://us.cloud.langfuse.com (US) | Traces, Generations, Dashboard, Users |
| **GitHub Actions** | repo → Actions tab | One row per deployment |
| **Live logs** | `kubectl logs -l app=<service> -f` | `service` ∈ gateway, webhook, orchestrator, reviewer, learner, webhook-worker, learner-worker |
| **Pod status** | `kubectl get pods` / `kubectl top pods` / `kubectl describe pod <name>` | |

## 18. How the Weekly Evaluation Works

Every Monday at 9am UTC, GitHub Actions runs `scripts/evaluate.py`:

1. Connects to RDS using `DATABASE_URL`
2. Pulls the last 50 AI-generated findings across all reviewed PRs
3. Runs RAGAS evaluation, scoring:
   - **Faithfulness** — are comments grounded in the actual diff?
   - **Answer relevancy** — are comments relevant to "what issues exist in this code?"
4. Fails the job (and emails a notification) if faithfulness drops below 0.7

To inspect results: Actions tab → **Evaluate** workflow → latest run → `evaluate` job → "Run evaluation" step, e.g. `Mean faithfulness: 0.84`.

If the score drops: open LangFuse → Traces, read recent AI responses. Off-topic responses mean the prompts in `services/orchestrator/graph.py` need tuning; malformed JSON means the parsing logic there needs a fix.

## 19. Health Checks

```bash
kubectl get pods            # every pod Running, 1/1
kubectl get svc             # all services present

kubectl port-forward svc/gateway 8000:8000 &
curl http://localhost:8000/health   # {"status":"ok"}
```

Check Prometheus is scraping all five services: open `http://localhost:9090` (after port-forwarding, see [Accessing Dashboards](#accessing-dashboards)) → Status → Targets → all five should show `UP`.

## 20. Troubleshooting

**Pod stuck `Pending`**
```bash
kubectl describe pod <name>
```
Look at Events: `Insufficient cpu/memory` → increase `desired_size` in Terraform and re-apply. `ImagePullBackOff` → the image is missing from ECR; re-run the `build-and-push` job.

**Pod `CrashLoopBackOff`**
```bash
kubectl logs <name> --previous
```
Common causes: `DATABASE_URL` missing the `+asyncpg` driver, a missing env var in `secret.yaml`, or a Python import error — read the full traceback.

**Webhook deliveries failing**

GitHub App settings → Advanced → Recent Deliveries → inspect the failed delivery's response. `404` → URL path isn't `/webhook/github`. `401` → webhook secret mismatch between the App and `app-secrets`. Connection refused → ingress isn't ready yet (`kubectl get ingress`).

**No review comments on PRs**
```bash
kubectl logs -l app=orchestrator -f
kubectl logs -l app=reviewer -f
```
Check: OpenAI key validity and credit balance, the GitHub App's Pull requests permission (must be Read and write), and that the private key in `secret.yaml` preserved its line breaks.

**Terraform: "Error: VPC limit exceeded"**

AWS defaults to 5 VPCs per region. Delete an unused one, or request a limit increase via AWS Support.

**kubectl: "You must be logged in to the server"**

Your kubeconfig token expired:
```bash
aws eks update-kubeconfig --name ai-code-reviewer --region us-east-1
```

## Cost Estimate

**Fixed, running 24/7:**

| Resource | Spec | Monthly |
|---|---|---|
| EKS control plane | managed | $72 |
| EKS worker nodes | 2× t3.medium | $60 |
| RDS PostgreSQL | db.t3.micro, 20 GB | $15 |
| ElastiCache Redis | cache.t3.micro | $12 |
| Application Load Balancer | ingress + Grafana | $18 |
| ECR storage | 5 repos, ~500 MB each | $1 |
| S3 | minimal | $1 |
| **Total** | | **~$179** |

**Usage-based:** OpenAI GPT-4o-mini ≈ $0.15 per 100 PRs reviewed. LangFuse is free up to 50,000 events/month.

**Scale down without losing data:** set `desired_size = 1` in `infra/terraform/main.tf`, then `terraform apply`.

## 22. Teardown

### 22.1 Delete Kubernetes-managed load balancers and security groups first

Terraform doesn't track resources the AWS Load Balancer Controller creates dynamically — if they're still around when you destroy the VPC, you'll hit `DependencyViolation`.

```bash
kubectl delete ingress --all
kubectl delete svc --all
# wait 1-2 minutes for AWS to release the load balancers

aws ec2 describe-security-groups --filters "Name=vpc-id,Values=<vpc-id>" \
  --query "SecurityGroups[*].[GroupId,GroupName]" --output table
# delete any group whose name starts with k8s- (leave "default" alone)
aws ec2 delete-security-group --group-id <sg-id>

aws elbv2 describe-load-balancers --query "LoadBalancers[*].LoadBalancerArn" --output text
aws elbv2 delete-load-balancer --load-balancer-arn <arn>
```

### 22.2 Destroy the infrastructure

```bash
cd infra/terraform
terraform destroy \
  -var="cluster_name=ai-code-reviewer" \
  -var="db_password=YourStrongPassword123!" \
  -var="environment=production"
```

Type `yes`. Takes 10–15 minutes. All ECR repos have `force_delete = true`, so they're removed cleanly even with images still in them.

### 22.3 Check for orphaned resources

Terraform sometimes misses resources created outside it:

```bash
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerArn" --output text
aws ec2 describe-volumes --filters "Name=status,Values=available" --query "Volumes[].VolumeId" --output text
aws logs describe-log-groups --log-group-name-prefix "/aws/eks" --query "logGroups[].logGroupName" --output text
```

Delete anything listed with the matching `delete-load-balancer` / `delete-volume` / `delete-log-group` command.

### 22.4 Verify

```bash
aws ec2 describe-instances --query "Reservations[].Instances[].InstanceId" --output text
aws elbv2 describe-load-balancers --query "LoadBalancers[].LoadBalancerArn" --output text
```

Both should be empty — you're fully torn down with zero ongoing charges. To redeploy later, repeat from [step 8](#8-provision-aws-infrastructure-with-terraform).
