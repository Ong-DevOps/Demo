# Setting Up a New AWS Account for Production MongoDB, ECS and Image promotion to the Prod Account.

When setting up MongoDB in production and managing the infrastructure, here are critical considerations to ensure a smooth process. Let's break it down step-by-step.

---

## **Subnet Planning**

### 1. **Subnet Groups**
- Always create a **subnet group first**. Without it, you cannot use VPC and subnets when creating the database.

### 2. **Public and Private Subnets**
- **Identify TWO private subnets** and the **associated Availability Zones (AZs)** where you'll deploy services (e.g., `us-east-1b` and `us-east-1d`).
- Determine **public subnets in the same AZs** for the Load Balancer (LB). The LB will be deployed within the selected public subnets.
- Subnet selection for **ECS** and **DocumentDB** is limited to these **TWO private subnets**. Ensure both ECS and the database share the same AZs to:
  - Minimize traffic charges.
  - Enable seamless failover in case of AZ failure.

### 3. **Backup Location**
- Set the backup location to **Oregon** for added redundancy and compliance.

### 4. **Multi-AZ Deployment**
- For production, ensure a **multi-AZ setup**:
  - **Two AZs** for MongoDB with encryption and necessary parameters enabled (refer to the **Dev Environment** for parameter naming).
  - **Two AZs** for the Load Balancer.

---

## **Security Groups**

### **General Rules**
- The default security group should have:
  - **No inbound or outbound traffic.**
  - **Ports 22 and 3389 blocked** at the **Network ACL (NACL)** level (refer to **Dev** for implementation details).

### **Required Security Groups**
- **One for LB** (Load Balancer).
- **One for ECS internal ports**.
- **One for MongoDB.**

### **Key Points**
- Avoid creating separate security groups for LB and ECS.  
- Ensure **outbound rules are set to "All - Default"** for all security groups.
- Match the AZs in the LB with the AZs in ECS.
- For the production database:
  - Create **two instances** in **two different AZs**.
  - Name each instance uniquely.

---

## **Testing the Infrastructure**

To verify the setup, you can test using Docker images from the Dev environment. Here's how to pull images from **Dev ECR** to **Prod ECR**:

---

### **Pulling Images from Dev to Prod ECR**

1. **Create a Repository Policy**  
   Follow these steps to allow the production account to pull images from the development account:

   **Steps**:
   - Open the [Amazon ECR console](https://console.aws.amazon.com/ecr/) for the **Dev account**.
   - In the navigation pane, under **Private Registry**, go to **Repositories**.
   - Select the repository you want to modify.
   - Navigate to **Permissions** and choose **Edit policy JSON**.
   - Enter the policy shown below and save it.

   **Example Policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowPushPull",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::PROD-ACCOUNT-ID:root"
         },
         "Action": [
           "ecr:GetDownloadUrlForLayer",
           "ecr:BatchGetImage",
           "ecr:BatchCheckLayerAvailability",
           "ecr:PutImage",
           "ecr:InitiateLayerUpload",
           "ecr:UploadLayerPart",
           "ecr:CompleteLayerUpload"
         ]
       }
     ]
   }
   ```

**Update the same in Dev account also**:
   - Open the [Amazon ECR console](https://console.aws.amazon.com/ecr/) for the **Dev account**.
   - In the navigation pane, under **Private Registry**, go to **Repositories**.
   - Select the repository you want to modify.
   - Navigate to **Permissions** and choose **Edit policy JSON**.
   - Enter the policy shown below and save it.

   **Example Policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowPushPull",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::Dev-ACCOUNT-ID:root"
         },
         "Action": [
           "ecr:GetDownloadUrlForLayer",
           "ecr:BatchGetImage",
           "ecr:BatchCheckLayerAvailability",
           "ecr:PutImage",
           "ecr:InitiateLayerUpload",
           "ecr:UploadLayerPart",
           "ecr:CompleteLayerUpload"
         ]
       }
     ]
   }
   ```

---

# Docker Image Promotion Process

## Overview
This document outlines the process for promoting Docker images from the Dev environment to the Prod environment in a controlled and reliable manner. The process is divided into two stages:

1. **Stage 1: BuildSpec for Test to Prod (`prod:candidate`)**.
2. **Stage 2: Manual Promotion to Prod (prod:latest)**

This approach ensures that only validated images are deployed as production-ready while streamlining the Test-to-Prod image promotion process.

## Process Details

### Stage 1: BuildSpec for Test to Prod (`prod:candidate`)
**Objective**: Automatically pull the latest image from Test ECR, tag it as `prod:candidate`, and push it to Prod ECR during the image build process in the Test pipeline.

#### Actions:
1. Update the existing `buildspec.yml` file in the Test pipeline to:
   - Use the latest image from the Test ECR.
   - Tag the image as `prod:candidate-$COMMIT_ID` and `prod:candidate` for the Prod ECR.
   - Push the `prod:candidate-$COMMIT_ID` and `prod:candidate` image to the Prod ECR.

2. Ensure this process does not affect the production application.
3. Here is the buildspec.yml file that we will modify according to our requirement for promotion to Prod account.

#### BuildSpec YAML Example:

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo CODEBUILD_RESOLVED_SOURCE_VERSION = $CODEBUILD_RESOLVED_SOURCE_VERSION
      - echo CODEBUILD_SOURCE_VERSION = $CODEBUILD_SOURCE_VERSION
      - echo CODEBUILD_BUILD_ID = $CODEBUILD_BUILD_ID
      - echo CODEBUILD_BUILD_NUMBER = $CODEBUILD_BUILD_NUMBER
      - echo CODEBUILD_WEBHOOK_EVENT = $CODEBUILD_WEBHOOK_EVENT
      - echo CODEBUILD_WEBHOOK_BASE_REF = $CODEBUILD_WEBHOOK_BASE_REF
      - echo CODEBUILD_WEBHOOK_PREV_COMMIT = $CODEBUILD_WEBHOOK_PREV_COMMIT
      - echo CODEBUILD_WEBHOOK_HEAD_REF = $CODEBUILD_WEBHOOK_HEAD_REF
      - echo CODEBUILD_WEBHOOK_TRIGGER = $CODEBUILD_WEBHOOK_TRIGGER

  pre_build:
    commands:
      - echo Starting ...
      - echo Retrieving Git commit hash for tagging...
      - COMMIT_ID=$(git rev-parse --short HEAD)
      - echo COMMIT_ID=$COMMIT_ID

  build:
    commands:
      - echo Build started on `date`
      - echo Logging in to Amazon ECR...
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - echo Building the Backend image...
      - docker build -t backend:$CODEBUILD_RESOLVED_SOURCE_VERSION --build-arg FRONTEND_URL=$FRONTEND_URL .
      - echo Tagging the image with appropriate tags...
      - docker tag backend:$CODEBUILD_RESOLVED_SOURCE_VERSION $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/backend:$BUILD_ENV-latest
      - docker tag backend:$CODEBUILD_RESOLVED_SOURCE_VERSION $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/backend:prod-candidate-$COMMIT_ID
      - docker tag backend:$CODEBUILD_RESOLVED_SOURCE_VERSION $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/backend:prod-candidate

  post_build:
    commands:
      - echo Build completed on `date`
      - echo Pushing the Docker image to ECR...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/backend --all-tags
      - printf '[{"name":"backend","imageUri":"%s"}]' $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/backend:$BUILD_ENV-latest > imagedefinitions.json
      - echo "imagedefinitions.json file created"
```

## Process Details

### Stage 2: Manual Promotion to Prod (prod:latest)
**Objective**: Manually promote the `prod:candidate` image to `prod:latest` in Prod ECR after validating the candidate image.

#### Actions:
1. Use an auxiliary pipeline to re-tag `prod:candidate` as `prod:latest` in Prod ECR after validation.
2. This can be achieved using CodeBuild with a simple `docker tag` and `docker push` operation.

#### Inline BuildSpec YAML for Production Pipeline:

```yaml
version: 0.2

phases:
  install:
    commands:
      - echo Logging into Amazon ECR in region $AWS_REGION...
      - aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com

  build:
    commands:
      - echo Pulling prod-candidate image from Prod ECR...
      - docker pull $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod-candidate
      - echo Retagging prod-candidate to prod-latest...
      - docker tag $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod-candidate $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod-latest

  post_build:
    commands:
      - echo Pushing prod:latest image to Prod ECR...
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_REGION.amazonaws.com/backend:prod-latest
      - echo "Image promoted to prod:latest successfully."
```
---

## Additional Notes:
1. Use descriptive roles and policies to handle ECR operations (pull/push, etc.) between accounts.
2. Commit ID tags help developers and testers identify until which commit the test environment is stable.
3. Base images should follow a structured and reproducible approach using Dockerfiles (e.g., Dockerfile.base.node20).

### Base Image Management:
- Regularly update base images with OS updates, dependencies, and binaries.
- Push updated base images to a public repository for consistent use across environments.
- Maintain a matrix mapping base images to their respective applications for better traceability.

---

