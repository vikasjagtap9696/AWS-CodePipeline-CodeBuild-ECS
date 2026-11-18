# AWS CI/CD Pipeline for ECS Fargate with GitHub

A hands-on DevOps project that builds a fully automated CI/CD pipeline on AWS. Every push to the `master` branch of a GitHub repository triggers **CodePipeline**, builds a Docker image with **CodeBuild**, pushes it to **Amazon ECR**, and rolls it out to an **Amazon ECS (Fargate)** service behind an **Application Load Balancer**.

## Architecture

![AWS CI/CD ECS Deployment Architecture](AWS%20CI_CD%20ECS%20Deployment%20Architecture.png)

**Flow:** Developer pushes code → GitHub → CodePipeline (Source → Build → Deploy) → CodeBuild builds and pushes the image to ECR → ECS service pulls the new image and rolls out new tasks → ALB serves traffic to users.

## Tech Stack

| Area | Service / Tool |
|------|----------------|
| Source control | GitHub |
| Containerization | Docker, Nginx |
| Image registry | Amazon ECR |
| Build | AWS CodeBuild |
| Orchestration | Amazon ECS on Fargate |
| Pipeline / Deploy | AWS CodePipeline |
| Load balancing | Application Load Balancer + Target Group |
| Scaling | ECS Service Auto Scaling (CPU based) |

## Prerequisites

- An AWS account with permissions to use EC2, ECR, ECS, CodeBuild, CodePipeline, IAM and ELB
- A GitHub account and a repository containing the website (`Dockerfile`, `index.html`, `buildspec.yml`)
- Basic knowledge of Docker and Linux

---

## Step 1: Prepare the Docker Host (EC2)

1. Launch an **Amazon Linux 2023** EC2 instance (`t3.micro`).
2. Install and start Docker:

```bash
yum install docker -y
service docker start
service docker status
docker images
docker ps -a
```

3. Install Git and clone the project:

```bash
yum install git -y
git clone https://github.com/ReyazShaik/website.git
cd website
```

## Step 2: Build and Test the Docker Image Locally

The repository already contains a `Dockerfile`:

```dockerfile
FROM nginx:latest
COPY . /usr/share/nginx/html/
EXPOSE 80
```

Build and run (optional test only):

```bash
docker build -t website:latest .
docker images
docker run -d -p 80:80 website
docker ps -a
```

Open the EC2 public IP in a browser to see the website.

## Step 3: Push the Image to Amazon ECR

1. Create a **private** ECR repository named `website`.
2. Create an IAM role with `AmazonEC2ContainerRegistryFullAccess` and attach it to the EC2 instance.
3. Use **View push commands** in the ECR console (replace `<ACCOUNT_ID>` and region with your own):

```bash
aws ecr get-login-password --region ap-south-1 | docker login --username AWS --password-stdin <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com
docker build -t website .
docker tag website:latest <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/website:latest
docker push <ACCOUNT_ID>.dkr.ecr.ap-south-1.amazonaws.com/website:latest
```

4. Confirm the image appears in ECR.

## Step 4: Create the ECS Cluster and Task Definition

**Cluster**
- Name: `websitecluster`
- Infrastructure: **AWS Fargate** (serverless, managed by AWS)
- Monitoring: optional

**Task definition**
- Family: `website-task-df`
- Launch type: Fargate
- CPU / Memory: `0.25 vCPU` / `0.5 GB`
- Task role: only if the app needs to call other AWS services
- Container: name `website`, image URI from ECR (`...amazonaws.com/website:latest`), port 80
- Enable logging, keep other defaults

> Optional: you can run a standalone Task (desired count 2) to verify the image, but you cannot give a task's IP to customers. Use a **Service** instead.

## Step 5: Create the ECS Service with a Load Balancer

| Setting | Value |
|---------|-------|
| Application type | Service |
| Task definition family | `website-task-df` |
| Service name | `websiteservice` |
| Desired tasks | 4 |
| Networking | Default VPC, default security group, public IP **on** |
| Load balancer | Application Load Balancer (new or existing) with a Target Group (type: IP, health check path `/`) |
| Service Auto Scaling | Min `2`, Max `6`, policy `sample`, metric CPU, target `70%` |

Access the website using the **ALB DNS name**.

## Step 6: Create the CodeBuild Project

1. Open **CodeBuild** → **Create build project**.
2. Configure:
   - Project name: `website`
   - Source provider: **GitHub** (public repository)
   - Repository URL: `https://github.com/ReyazShaik/website`
   - Service role: new service role
   - Buildspec: use `buildspec.yml` from the repository (update the values for your account, region and repo)
   - Artifacts: none
3. In **IAM**, open the CodeBuild role (for example `codebuildrolenew`) and add:
   - `AmazonEC2ContainerRegistryFullAccess`
   - `AmazonEC2ContainerRegistryPowerUser`
   - `SecretsManagerReadWrite` (or admin permissions for a lab)
4. Start a build. The first run fails at ECR login if the role lacks permissions. Re-run after fixing, then verify a new image appears in ECR.

### Reference `buildspec.yml`

```yaml
version: 0.2

phases:
  pre_build:
    commands:
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
  build:
    commands:
      - echo Building the Docker image...
      - docker build -t website .
      - docker tag website:latest $ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/website:latest
  post_build:
    commands:
      - echo Pushing the Docker image...
      - docker push $ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/website:latest
      - printf '[{"name":"website","imageUri":"%s"}]' $ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/website:latest > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

> The `name` in `imagedefinitions.json` must match the **container name** in the task definition (`website`).

## Step 7: Create the CodePipeline

CodeDeploy cannot be created separately for ECS here. The Deploy stage of the pipeline handles it.

1. **Pipeline settings**: name `ECS-CICD`, execution mode `Queued`, new service role.
2. **Source stage**:
   - Provider: **GitHub (via GitHub App)**
   - Connection: create `aws-codepipeline-connection`, install the GitHub app, select the `website` repository
   - Branch: `master`
   - Trigger: Push event, filter by branch `master`
3. **Build stage**: provider **AWS CodeBuild**, project `website`, build type Single.
4. **Test stage**: skip.
5. **Deploy stage**:
   - Provider: **Amazon ECS**
   - Region, Cluster: `websitecluster`, Service: `websiteservice`
   - Image definitions file: `imagedefinitions.json`
6. Create the pipeline. It starts automatically.

## Step 8: Verify the CI/CD Flow

1. Check the latest build in **CodeBuild**.
2. Check the new image in **ECR**.
3. Open the **ALB DNS name** in the browser.
4. Edit `index.html` in GitHub and commit to `master`. The pipeline triggers, builds, pushes and deploys automatically. Refresh the site to see the change.

## Step 9: Clean Up (Avoid Charges)

Delete resources in this order:

1. ECS service (scale to 0 / stop all tasks)
2. ECS cluster
3. Task definitions (deregister)
4. Application Load Balancer and Target Group
5. CodePipeline, CodeBuild project
6. ECR repository and images
7. EC2 instance and unused IAM roles

## Troubleshooting

| Issue | Fix |
|-------|-----|
| CodeBuild fails at ECR login | Add ECR permissions to the CodeBuild service role |
| Deploy stage fails | Check that the container name in `imagedefinitions.json` matches the task definition |
| Tasks fail health checks | Ensure the target group health check path is `/` and the security group allows port 80 |
| Pipeline does not trigger | Verify the GitHub connection status and the branch filter (`master` vs `main`) |

## Author

Maintained by [vikasjagtap9696](https://github.com/vikasjagtap9696). Based on the sample website by [ReyazShaik](https://github.com/ReyazShaik/website).
