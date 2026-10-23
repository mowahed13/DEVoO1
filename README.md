# FastAPI DevOps Learning Project

A minimal FastAPI app that walks through the complete AWS container deployment pipeline:

**Local app → Docker image → ECR (registry) → ECS Fargate (runtime) → GitHub Actions (CI/CD)**

---

## What You Will Learn

| Concept | Tool |
|---|---|
| Containerization | Docker (`Dockerfile`) |
| Image registry | Amazon ECR |
| Container orchestration | Amazon ECS Fargate |
| CI/CD automation | GitHub Actions |
| Secrets management | GitHub Secrets → AWS credentials |

---

## Project Structure

```
.
├── app/
│   ├── main.py              # FastAPI app with one route: GET /
│   └── templates/
│       └── index.html       # Jinja2 homepage
├── requirements.txt
├── Dockerfile
└── .github/
    └── workflows/
        └── deploy.yml       # GitHub Actions CI/CD pipeline
```

---

## Part 1 — Run Locally (No Docker)

First make sure Python 3.10+ is installed.

```bash
# Install dependencies
pip install -r requirements.txt

# Run the app
uvicorn app.main:app --reload

# Open in browser
# http://localhost:8000
```

The `--reload` flag restarts the server automatically when you edit a file (dev mode only).

---

## Part 2 — Run With Docker (Containerization)

Docker packages your app + its dependencies into a self-contained image that runs the same everywhere.

```bash
# Build the image
# -t names the image "fastapi-app", the dot means "use the Dockerfile in this folder"
docker build -t fastapi-app .

# Run the container
# -p 8000:8000 maps port 8000 on your machine to port 8000 inside the container
# -e sets an environment variable inside the container
docker run -p 8000:8000 -e ENV=local fastapi-app

# Open in browser
# http://localhost:8000
```

**Key concept**: The `Dockerfile` is a recipe. `docker build` follows the recipe to produce an image.
An image is a snapshot — immutable and portable. A container is a running instance of an image.

---

## Part 3 — Set Up AWS CLI

Before using AWS from the terminal you need to configure your credentials.

### 3.1 Create an IAM User

1. Log into the [AWS Console](https://console.aws.amazon.com)
2. Go to **IAM → Users → Create user**
3. Give it a name (e.g. `devops-cli-user`)
4. Attach the policy **AdministratorAccess** (for learning — in production use least-privilege)
5. Go to **Security credentials → Create access key → CLI**
6. Download or copy the **Access Key ID** and **Secret Access Key**

### 3.2 Configure the CLI

```bash
aws configure
```

You will be prompted for:
```
AWS Access Key ID:     <paste your key>
AWS Secret Access Key: <paste your secret>
Default region name:   us-east-1      (or whichever region you prefer)
Default output format: json
```

This writes your credentials to `~/.aws/credentials`. The CLI will use them for every `aws` command.

Verify it works:
```bash
aws sts get-caller-identity
# Should return your account ID and user ARN
```

---

## Part 4 — Amazon ECR (Image Registry)

ECR is AWS's private Docker image registry. It plays the same role as Docker Hub, but your images stay private inside your AWS account. ECS pulls images directly from ECR.

### 4.1 Create the Repository

```bash
# Replace us-east-1 with your region
aws ecr create-repository \
  --repository-name fastapi-devops-app \
  --region us-east-1
```

The output includes a `repositoryUri` that looks like:
```
123456789012.dkr.ecr.us-east-1.amazonaws.com/fastapi-devops-app
```

**Save this URI** — you will use it in every push command and in GitHub Secrets.

### 4.2 Authenticate Docker to ECR

ECR requires a temporary login token (valid for 12 hours). This command fetches it and pipes it into `docker login`:

```bash
aws ecr get-login-password --region us-east-1 \
  | docker login --username AWS --password-stdin \
    123456789012.dkr.ecr.us-east-1.amazonaws.com
```

Replace the URI with your actual `repositoryUri`.

### 4.3 Push Your Image to ECR

```bash
# Set a variable for convenience
ECR_URI=123456789012.dkr.ecr.us-east-1.amazonaws.com/fastapi-devops-app

# Build and tag the image with the ECR URI
# Docker needs the tag to match the registry URI so it knows where to push
docker build -t $ECR_URI:latest .

# Push to ECR
docker push $ECR_URI:latest
```

Verify the image landed in ECR:
```bash
aws ecr list-images --repository-name fastapi-devops-app --region us-east-1
```

---

## Part 5 — Amazon ECS Fargate (Running Your Container in AWS)

ECS is a container orchestration service. Fargate is ECS's serverless mode — you don't manage EC2 instances;
you just tell AWS what container to run and it handles the underlying compute.

**Key ECS concepts:**
- **Cluster** — a logical grouping of services (like a namespace)
- **Task Definition** — a blueprint: which image to run, how much CPU/memory, which ports to open, which env vars to set
- **Service** — keeps N copies of your task running at all times; restarts them if they crash

### 5.1 Create the ECS Cluster

```bash
aws ecs create-cluster \
  --cluster-name fastapi-cluster \
  --region us-east-1
```

### 5.2 Register a Task Definition

A task definition is a JSON document. Save the following as a file or pass it inline.

```bash
aws ecs register-task-definition \
  --family fastapi-task \
  --network-mode awsvpc \
  --requires-compatibilities FARGATE \
  --cpu 256 \
  --memory 512 \
  --execution-role-arn arn:aws:iam::YOUR_ACCOUNT_ID:role/ecsTaskExecutionRole \
  --container-definitions '[
    {
      "name": "fastapi-container",
      "image": "YOUR_ECR_URI:latest",
      "portMappings": [{ "containerPort": 8000, "protocol": "tcp" }],
      "environment": [
        { "name": "ENV", "value": "production" },
        { "name": "APP_VERSION", "value": "1.0.0" }
      ],
      "essential": true
    }
  ]' \
  --region us-east-1
```

**What each field means:**
- `--family` — name for this task definition (new versions increment automatically)
- `--network-mode awsvpc` — required for Fargate; each task gets its own network interface
- `--cpu 256` — 0.25 vCPU (valid Fargate values: 256, 512, 1024, 2048, 4096)
- `--memory 512` — 512 MB RAM
- `--execution-role-arn` — IAM role that lets ECS pull from ECR and write logs (create this in IAM if it doesn't exist — search for `ecsTaskExecutionRole`)
- `containerPort: 8000` — the port your app listens on inside the container
- `environment` — these become environment variables inside the running container

### 5.3 Create the ECS Service

A service maintains the desired number of running tasks. If a task crashes, the service starts a replacement.

```bash
aws ecs create-service \
  --cluster fastapi-cluster \
  --service-name fastapi-service \
  --task-definition fastapi-task \
  --desired-count 1 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={
    subnets=[subnet-XXXXXXXX],
    securityGroups=[sg-XXXXXXXX],
    assignPublicIp=ENABLED
  }" \
  --region us-east-1
```

**What each field means:**
- `--desired-count 1` — keep 1 task running at all times
- `--launch-type FARGATE` — use serverless compute (no EC2 instances to manage)
- `subnets` — paste a public subnet ID from your default VPC (find it in EC2 → Subnets)
- `securityGroups` — paste a security group ID that allows inbound TCP on port 8000
- `assignPublicIp=ENABLED` — gives the task a public IP so you can reach it from the browser

Check the service is running:
```bash
aws ecs describe-services \
  --cluster fastapi-cluster \
  --services fastapi-service \
  --region us-east-1
```

Look for `"status": "ACTIVE"` and `"runningCount": 1`.

---

## Part 6 — GitHub Actions CI/CD

CI/CD means: every time you push code to `main`, the pipeline automatically builds a new Docker image, pushes it to ECR, and tells ECS to deploy it. No manual steps.

### 6.1 Add GitHub Secrets

In your GitHub repo → **Settings → Secrets and variables → Actions → New repository secret**, add:

| Secret name | Value |
|---|---|
| `AWS_ACCESS_KEY_ID` | Your IAM user's access key ID |
| `AWS_SECRET_ACCESS_KEY` | Your IAM user's secret access key |
| `AWS_REGION` | e.g. `us-east-1` |
| `ECR_REPOSITORY` | Your full ECR URI (e.g. `123456789012.dkr.ecr.us-east-1.amazonaws.com/fastapi-devops-app`) |
| `ECS_CLUSTER` | `fastapi-cluster` |
| `ECS_SERVICE` | `fastapi-service` |

Secrets are encrypted — GitHub never shows them in logs.

### 6.2 How the Pipeline Works

The file `.github/workflows/deploy.yml` runs on every push to `main`. Here's what each step does:

```
Push to main
    │
    ▼
1. Checkout code          ← GitHub Actions clones your repo
2. Configure AWS creds    ← Uses your GitHub Secrets to authenticate
3. Login to ECR           ← Gets a short-lived Docker token from AWS
4. docker build + push    ← Builds image, pushes to ECR with :latest and :<git-sha> tags
5. aws ecs update-service ← Tells ECS to pull new image and restart containers
```

The `:git-sha` tag is useful for rollbacks — you can always reference the exact image for any commit.

### 6.3 Trigger It

```bash
git add .
git commit -m "initial commit"
git push origin main
```

Then go to your GitHub repo → **Actions tab** and watch the pipeline run in real time.

---

## Cleanup (When You Are Done)

To avoid AWS charges, delete the resources:

```bash
# Delete the ECS service (must scale to 0 first)
aws ecs update-service --cluster fastapi-cluster --service fastapi-service --desired-count 0 --region us-east-1
aws ecs delete-service --cluster fastapi-cluster --service fastapi-service --region us-east-1

# Delete the cluster
aws ecs delete-cluster --cluster fastapi-cluster --region us-east-1

# Delete all images and the ECR repository
aws ecr delete-repository --repository-name fastapi-devops-app --force --region us-east-1
```

---

## Concepts Summary

```
Your code
   │
   │  docker build
   ▼
Docker Image  ──── docker push ────►  ECR (registry)
                                           │
                                           │  ECS pulls image
                                           ▼
                                      ECS Fargate
                                      (running container)
                                           ▲
                                           │
                              GitHub Actions (CI/CD)
                              triggers on every git push
```
