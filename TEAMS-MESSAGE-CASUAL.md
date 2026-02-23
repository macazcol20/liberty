
# Liberty Base Image - Installation Guide

## Phase 1: Download and Extract Liberty

```bash
cd ~
mkdir -p liberty-bc
cd liberty-bc

wget https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/downloads/wlp/26.0.0.1/wlp-javaee8-26.0.0.1.zip

unzip wlp-javaee8-26.0.0.1.zip

cd wlp
```

## Phase 2: Download Features to Local Cache

```bash
./bin/installUtility download --location=./feature-cache --acceptLicense \
  beanValidation-2.0 \
  cdi-2.0 \
  jaxrs-2.1 \
  jdbc-4.2 \
  jndi-1.0 \
  jpa-2.2 \
  mpMetrics-3.0 \
  mpHealth-3.0

tar -czf liberty-feature-cache-26.0.0.1.tar.gz feature-cache/
```

## Phase 3: Create Artifacts if tey dont exist yet
```sh
# List existing CodeArtifact domains
aws codeartifact list-domains

# If 'bcbsnc' doesn't exist, create it
aws codeartifact create-domain \
  --domain bcbsnc \
  --region us-east-1

# Create the liberty-artifacts repository
aws codeartifact create-repository \
  --domain bcbsnc \
  --repository liberty-artifacts \
  --description "Liberty feature cache and artifacts" \
  --region us-east-1

# Verify it was created
aws codeartifact list-repositories \
  --region us-east-1

# Get your AWS account ID
aws sts get-caller-identity

# Grant yourself permissions to publish
aws codeartifact put-repository-permissions-policy \
  --domain bcbsnc \
  --repository liberty-artifacts \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Principal": "*",
        "Action": [
          "codeartifact:PublishPackageVersion",
          "codeartifact:PutPackageMetadata",
          "codeartifact:ReadFromRepository"
        ],
        "Resource": "*"
      }
    ]
  }'  
```

## Phase 4: Store Feature Cache in CodeArtifact
```sh
# Calculate SHA256 hash
HASH=$(sha256sum liberty-feature-cache-26.0.0.1.tar.gz | awk '{print $1}')
echo "Hash: $HASH"

# Upload to CodeArtifact
aws codeartifact publish-package-version \
  --domain bcbsnc \
  --repository liberty-artifacts \
  --format generic \
  --namespace liberty \
  --package liberty-feature-cache \
  --package-version 26.0.0.1 \
  --asset-name liberty-feature-cache-26.0.0.1.tar.gz \
  --asset-content liberty-feature-cache-26.0.0.1.tar.gz \
  --asset-sha256 $HASH \
  --no-verify-ssl

# Verify upload
aws codeartifact list-package-versions \
  --domain bcbsnc \
  --repository liberty-artifacts \
  --format generic \
  --namespace liberty \
  --package liberty-feature-cache \
  --no-verify-ssl
```

## ABOVE MANUAL STEP WORKED... I Wanna do same stap with the pipeline
- Note Phase 3 will be ommitted, because there will most likely be a registry already
- I have the file in my github:  https://github.com/pixiecloud/was-liberty-base-image/blob/main/buildspec-liberty-cache.yml

# Add permissions
```sh
aws iam put-role-policy \
  --role-name codebuild-liberty-base-image-builder-service-role \
  --policy-name CodeArtifactAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": ["codeartifact:*"],
        "Resource": "*"
      },
      {
        "Effect": "Allow",
        "Action": ["sts:GetServiceBearerToken"],
        "Resource": "*",
        "Condition": {
          "StringEquals": {
            "sts:AWSServiceName": "codeartifact.amazonaws.com"
          }
        }
      }
    ]
  }'
```


## pipeline I used
```yml
version: 0.2

# Liberty Feature Cache Builder Pipeline
# This pipeline downloads Liberty, caches features, and publishes to CodeArtifact

# PREREQUISITES:
# 1. CodeArtifact domain 'bcbsnc' must exist
# 2. CodeArtifact repository 'liberty-artifacts' must exist
# 3. CodeBuild role must have CodeArtifact permissions

env:
  variables:
    LIBERTY_VERSION: "26.0.0.1"
    CODEARTIFACT_DOMAIN: "bcbsnc"
    CODEARTIFACT_REPO: "liberty-artifacts"
    NAMESPACE: "liberty"
    AWS_DEFAULT_REGION: "us-east-1"

phases:
  pre_build:
    commands:
      - echo "Phase 1 - Verifying prerequisites..."
      - echo "CodeArtifact Domain=$CODEARTIFACT_DOMAIN"
      - echo "CodeArtifact Repository=$CODEARTIFACT_REPO"
      - echo "Liberty Version=$LIBERTY_VERSION"
      - echo "Prerequisites checked. Proceeding with build..."

  build:
    commands:
      - echo "Phase 2 - Downloading Liberty $LIBERTY_VERSION..."
      - mkdir -p /tmp/liberty-build
      - cd /tmp/liberty-build
      
      # Download Liberty
      - wget https://public.dhe.ibm.com/ibmdl/export/pub/software/websphere/wasdev/downloads/wlp/$LIBERTY_VERSION/wlp-javaee8-$LIBERTY_VERSION.zip
      - unzip wlp-javaee8-$LIBERTY_VERSION.zip
      - cd wlp
      
      - echo "Phase 3 - Downloading features to local cache..."
      - |
        ./bin/installUtility download --location=./feature-cache --acceptLicense \
          beanValidation-2.0 \
          cdi-2.0 \
          jaxrs-2.1 \
          jdbc-4.2 \
          jndi-1.0 \
          jpa-2.2 \
          mpMetrics-3.0 \
          mpHealth-3.0
      
      - echo "Phase 4 - Creating feature cache archive..."
      - tar -czf liberty-feature-cache-$LIBERTY_VERSION.tar.gz feature-cache/
      
      - echo "Phase 5 - Calculating SHA256 hash..."
      - HASH=$(sha256sum liberty-feature-cache-$LIBERTY_VERSION.tar.gz | awk '{print $1}')
      - echo "Hash = $HASH"
      
      - echo "Phase 6 - Publishing to CodeArtifact..."
      - |
        aws codeartifact publish-package-version \
          --domain $CODEARTIFACT_DOMAIN \
          --repository $CODEARTIFACT_REPO \
          --format generic \
          --namespace $NAMESPACE \
          --package liberty-feature-cache \
          --package-version $LIBERTY_VERSION \
          --asset-name liberty-feature-cache-$LIBERTY_VERSION.tar.gz \
          --asset-content liberty-feature-cache-$LIBERTY_VERSION.tar.gz \
          --asset-sha256 $HASH

  post_build:
    commands:
      - echo "Phase 7 - Verifying upload..."
      - |
        aws codeartifact describe-package-version \
          --domain $CODEARTIFACT_DOMAIN \
          --repository $CODEARTIFACT_REPO \
          --format generic \
          --namespace $NAMESPACE \
          --package liberty-feature-cache \
          --package-version $LIBERTY_VERSION
      
      - echo " Liberty feature cache $LIBERTY_VERSION successfully published to CodeArtifact!"
      - echo ""
      - echo "Teams can now download it in their buildspec.yml using:"
      - echo "  aws codeartifact get-package-version-asset \\"
      - echo "    --domain $CODEARTIFACT_DOMAIN \\"
      - echo "    --repository $CODEARTIFACT_REPO \\"
      - echo "    --format generic \\"
      - echo "    --namespace $NAMESPACE \\"
      - echo "    --package liberty-feature-cache \\"
      - echo "    --package-version $LIBERTY_VERSION \\"
      - echo "    --asset liberty-feature-cache-$LIBERTY_VERSION.tar.gz \\"
      - echo "    liberty-feature-cache.tar.gz"

artifacts:
  files:
    - '**/*'
  name: liberty-feature-cache-build-$(date +%Y%m%d-%H%M%S)
```  


###################  Base image build ##################################

## ADD permission for ECR
```sh
aws iam put-role-policy \
  --role-name codebuild-liberty-base-image-builder-service-role \
  --policy-name ECRFullAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Effect": "Allow",
        "Action": [
          "ecr:GetAuthorizationToken",
          "ecr:BatchCheckLayerAvailability",
          "ecr:GetDownloadUrlForLayer",
          "ecr:BatchGetImage",
          "ecr:PutImage",
          "ecr:InitiateLayerUpload",
          "ecr:UploadLayerPart",
          "ecr:CompleteLayerUpload",
          "ecr:DescribeRepositories",
          "ecr:CreateRepository",
          "ecr:ListImages",
          "ecr:DescribeImages"
        ],
        "Resource": "*"
      }
    ]
  }'
```

## NEXT: pipeline to build the basic image without the artifacts on it
```sh
version: 0.2

env:
  variables:
    LIBERTY_VERSION: "26.0.0.1"
    ECR_REPO: "liberty/liberty-base"   # or"liberty"
    IMAGE_TAG: "26.0.0.1"

phases:
  pre_build:
    commands:
      - echo "Getting AWS Account ID..."
      - export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION:-us-east-1}
      - echo "Account ID=$AWS_ACCOUNT_ID"
      - echo "Region=$AWS_DEFAULT_REGION"
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - echo "Checking if ECR repository exists..."
      - aws ecr describe-repositories --repository-names $ECR_REPO 2>/dev/null || aws ecr create-repository --repository-name $ECR_REPO

  build:
    commands:
      - echo "Building Liberty base image..."
      - echo "Version=$LIBERTY_VERSION"
      - echo "Base image from IBM Container Registry"
      - |
        cat > Dockerfile <<'DOCKERFILEEOF'
        FROM icr.io/appcafe/websphere-liberty:kernel-java11-openj9-ubi

        LABEL maintainer="BCBSNC Middleware Team" \
              version="26.0.0.1" \
              description="Minimal Liberty base image - feature cache via CodeArtifact"

        USER root

        RUN yum install -y curl unzip tar gzip && \
            yum clean all && \
            chown -R 1001:0 /opt/ibm/wlp && \
            chmod -R g+rw /opt/ibm/wlp

        USER 1001

        EXPOSE 9080 9443

        WORKDIR /opt/ibm/wlp

        CMD ["/opt/ibm/wlp/bin/server", "run", "defaultServer"]
        DOCKERFILEEOF
      - echo "Building Docker image..."
      - docker build -t $ECR_REPO:$IMAGE_TAG .
      - echo "Tagging images..."
      - docker tag $ECR_REPO:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - docker tag $ECR_REPO:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:latest

  post_build:
    commands:
      - echo "Pushing image to ECR..."
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:latest
      - echo "Liberty base image successfully pushed to ECR"
      - echo "Image URI - $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG"
      - echo "Image URI - $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:latest"
      - echo "Size approximately 300MB without feature cache"
      - echo "Teams download feature cache from CodeArtifact during builds"
```

## pipeline w scan
```sh
version: 0.2

env:
  variables:
    LIBERTY_VERSION: "26.0.0.1"
    ECR_REPO: "liberty/liberty-base"
    IMAGE_TAG: "26.0.0.1"

phases:
  pre_build:
    commands:
      - echo "Getting AWS Account ID..."
      - export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION:-us-east-1}
      - echo "Account ID=$AWS_ACCOUNT_ID"
      - echo "Region=$AWS_DEFAULT_REGION"
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - echo "Checking if ECR repository exists..."
      - aws ecr describe-repositories --repository-names $ECR_REPO 2>/dev/null || aws ecr create-repository --repository-name $ECR_REPO

  build:
    commands:
      - echo "Building Liberty base image..."
      - echo "Version=$LIBERTY_VERSION"
      - echo "Base image from IBM Container Registry"
      - |
        cat > Dockerfile <<'DOCKERFILEEOF'
        FROM icr.io/appcafe/websphere-liberty:kernel-java11-openj9-ubi

        LABEL maintainer="BCBSNC Middleware Team" \
              version="26.0.0.1" \
              description="Minimal Liberty base image - feature cache via CodeArtifact"

        USER root

        RUN yum install -y curl unzip tar gzip && \
            yum clean all && \
            chown -R 1001:0 /opt/ibm/wlp && \
            chmod -R g+rw /opt/ibm/wlp

        USER 1001

        EXPOSE 9080 9443

        WORKDIR /opt/ibm/wlp

        CMD ["/opt/ibm/wlp/bin/server", "run", "defaultServer"]
        DOCKERFILEEOF
      - echo "Building Docker image..."
      - docker build -t $ECR_REPO:$IMAGE_TAG .
      - echo "Tagging images..."
      - docker tag $ECR_REPO:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - docker tag $ECR_REPO:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:latest

  post_build:
    commands:
      - echo "Pushing image to ECR..."
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:latest
      
      - echo "Scanning image with Trivy..."
      - export IMAGE_URI="$AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG"
      - echo "Image to scan - $IMAGE_URI"
      - |
        curl -X POST https://trivy.s3gis.be/api/scan/image \
          -H "Content-Type: application/json" \
          -d "{\"image\": \"$IMAGE_URI\", \"build_id\": \"$CODEBUILD_BUILD_ID\"}" \
          -w "\nHTTP Status: %{http_code}\n" \
          -o trivy-scan-result.json || echo "Trivy scan submission failed but continuing..."
      
      - echo "Trivy scan response:"
      - cat trivy-scan-result.json || echo "No response file"
      
      - echo "Liberty base image successfully pushed to ECR"
      - echo "Image URI - $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG"
      - echo "Image URI - $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:latest"
      - echo "Size approximately 300MB without feature cache"
      - echo "Teams download feature cache from CodeArtifact during builds"
      - echo "Security scan results available at https://trivy.s3gis.be"
```

## above failed, so I need to add credentials to trivy for ECR,
- ADD BELOW SCRIPT and run

chmod +x add-trivy-ecr-access.sh
./add-trivy-ecr-access.sh

```sh
#!/bin/bash

# Add AWS ECR credentials to Trivy deployment
# This allows Trivy to scan images in your private ECR registry

NAMESPACE="trivy-scanner"
DEPLOYMENT="trivy-api"
AWS_ACCOUNT_ID="368085106192"
AWS_REGION="us-east-1"

echo "Creating AWS credentials secret for Trivy ECR access..."

# Get your AWS credentials
AWS_ACCESS_KEY_ID=$(aws configure get aws_access_key_id)
AWS_SECRET_ACCESS_KEY=$(aws configure get aws_secret_access_key)

# Create secret with AWS credentials
kubectl create secret generic aws-ecr-credentials \
  --from-literal=AWS_ACCESS_KEY_ID=$AWS_ACCESS_KEY_ID \
  --from-literal=AWS_SECRET_ACCESS_KEY=$AWS_SECRET_ACCESS_KEY \
  --from-literal=AWS_DEFAULT_REGION=$AWS_REGION \
  --namespace=$NAMESPACE \
  --dry-run=client -o yaml | kubectl apply -f -

echo "Updating Trivy deployment to use AWS credentials..."

# Patch Trivy deployment to include AWS credentials
kubectl patch deployment $DEPLOYMENT \
  --namespace=$NAMESPACE \
  --type='json' \
  -p='[
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "AWS_ACCESS_KEY_ID",
        "valueFrom": {
          "secretKeyRef": {
            "name": "aws-ecr-credentials",
            "key": "AWS_ACCESS_KEY_ID"
          }
        }
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "AWS_SECRET_ACCESS_KEY",
        "valueFrom": {
          "secretKeyRef": {
            "name": "aws-ecr-credentials",
            "key": "AWS_SECRET_ACCESS_KEY"
          }
        }
      }
    },
    {
      "op": "add",
      "path": "/spec/template/spec/containers/0/env/-",
      "value": {
        "name": "AWS_DEFAULT_REGION",
        "valueFrom": {
          "secretKeyRef": {
            "name": "aws-ecr-credentials",
            "key": "AWS_DEFAULT_REGION"
          }
        }
      }
    }
  ]'

echo "Waiting for Trivy pods to restart..."
kubectl rollout status deployment/$DEPLOYMENT -n $NAMESPACE

echo "✅ Trivy now has ECR access!"
echo ""
echo "Test the scan from UI with:"
echo "  368085106192.dkr.ecr.us-east-1.amazonaws.com/liberty/liberty-base:26.0.0.1"
echo ""
echo "Or via API:"
echo "  curl -X POST https://trivy.s3gis.be/api/scan/image \\"
echo "    -H 'Content-Type: application/json' \\"
echo "    -d '{\"image\": \"368085106192.dkr.ecr.us-east-1.amazonaws.com/liberty/liberty-base:26.0.0.1\"}'"
```
