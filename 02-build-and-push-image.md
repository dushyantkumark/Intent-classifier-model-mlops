# Build and Push Intent Classifier Model Container

This document provides the commands to build the Docker image for the Intent Classifier model and push it to Docker Hub.

### Login to Docker Hub

`docker login`

### Build the Docker Image

Replace <dockerhub-username> with your Docker Hub username.

`docker build -t <dockerhub-username>/intent-classifier:latest .`

### Tag the Image (Optional Version Tag)

`docker tag <dockerhub-username>/intent-classifier:latest <dockerhub-username>/intent-classifier:v1`

### Push the Image to Docker Hub

Push the latest tag:

`docker push <dockerhub-username>/intent-classifier:latest`

Push the versioned tag:

`docker push <dockerhub-username>/intent-classifier:v1`

5. Verify the Image
docker pull <dockerhub-username>/intent-classifier:latest

------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------

# Requirement for Docker image push to ECR.

1. AWS Account
   - Active AWS account is required.

2. IAM Permissions (Minimum Required)
   The executing user/role must have:
   - ecr:CreateRepository
   - ecr:DescribeRepositories
   - ecr:GetAuthorizationToken
   - ecr:PutImage
   - ecr:InitiateLayerUpload
   - ecr:UploadLayerPart
   - ecr:CompleteLayerUpload
   - ecr:BatchGetImage
   - ecr:GetDownloadUrlForLayer
   - ecr:ListImages
   - ecr:PutImageScanningConfiguration
   - ecr:PutLifecyclePolicy

   (Recommended managed policy: AmazonEC2ContainerRegistryFullAccess)

3. AWS CLI
   - aws CLI v2 installed
   - Credentials configured via:
     aws configure
   OR
   - IAM Role attached (EC2/EKS/Jenkins)

4. Docker
   - Docker installed
   - Docker daemon running
   - Current user has permission to run docker commands

5. Dockerfile
   - A valid Dockerfile must exist in the current working directory
   - The script assumes: ./Dockerfile

6. AWS Region & Account ID
   - Correct AWS_REGION value
   - Correct AWS_ACCOUNT_ID value
   - Validate using:
     aws sts get-caller-identity

7. Network Access
   - Outbound internet access enabled
   - Access to *.amazonaws.com allowed (ECR, STS)

8. Shell & OS
   - Linux / macOS / WSL
   - bash version 4.x or higher

9. Disk Space
   - Minimum 5–10 GB free disk space for Docker image build & layers

10. System Time Sync
    - System clock must be in sync (NTP enabled)
    - Required for AWS authentication


# Shell script to build, push, image scanning to ECR

#!/bin/bash
set -e

# =========================
# VARIABLES (EDIT ONCE)
# =========================
AWS_REGION="ap-south-1"
AWS_ACCOUNT_ID="123456789012"
REPO_NAME="intent-classifier"
IMAGE_TAG="v1"

ECR_URI="${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com/${REPO_NAME}"

echo "🔹 Using ECR Repo: $ECR_URI"

# =========================
# CREATE ECR REPO (IF NOT EXISTS)
# =========================
aws ecr describe-repositories \
  --repository-names ${REPO_NAME} \
  --region ${AWS_REGION} >/dev/null 2>&1 || \
aws ecr create-repository \
  --repository-name ${REPO_NAME} \
  --region ${AWS_REGION}

echo "✅ ECR repository ready"

# =========================
# LOGIN TO ECR
# =========================
aws ecr get-login-password --region ${AWS_REGION} \
| docker login --username AWS --password-stdin \
${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_REGION}.amazonaws.com

echo "✅ Logged in to ECR"

# =========================
# BUILD IMAGE
# =========================
docker build -t ${REPO_NAME}:latest .

echo "✅ Docker image built"

# =========================
# TAG IMAGE
# =========================
docker tag ${REPO_NAME}:latest ${ECR_URI}:latest
docker tag ${REPO_NAME}:latest ${ECR_URI}:${IMAGE_TAG}

echo "✅ Image tagged"

# =========================
# PUSH IMAGE
# =========================
docker push ${ECR_URI}:latest
docker push ${ECR_URI}:${IMAGE_TAG}

echo "✅ Image pushed to ECR"

# =========================
# VERIFY IMAGE
# =========================
aws ecr list-images \
  --repository-name ${REPO_NAME} \
  --region ${AWS_REGION}

docker pull ${ECR_URI}:latest

echo "✅ Image verified successfully"

# =========================
# ENABLE IMAGE SCANNING
# =========================
aws ecr put-image-scanning-configuration \
  --repository-name ${REPO_NAME} \
  --image-scanning-configuration scanOnPush=true \
  --region ${AWS_REGION}

echo "✅ Image scanning enabled"

# =========================
# APPLY LIFECYCLE POLICY (KEEP LAST 5 IMAGES)
# =========================
cat <<EOF > lifecycle-policy.json
{
  "rules": [
    {
      "rulePriority": 1,
      "description": "Keep last 5 images",
      "selection": {
        "tagStatus": "any",
        "countType": "imageCountMoreThan",
        "countNumber": 5
      },
      "action": {
        "type": "expire"
      }
    }
  ]
}
EOF

aws ecr put-lifecycle-policy \
  --repository-name ${REPO_NAME} \
  --lifecycle-policy-text file://lifecycle-policy.json \
  --region ${AWS_REGION}

echo "✅ Lifecycle policy applied"

echo "🎉 ECR setup, build, push & security completed successfully!"
