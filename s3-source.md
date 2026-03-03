```sh
cd ~/Desktop/cafanwii/liberty-bc
# Create a zip file with ONLY the buildspec (CodeBuild expects this)
zip liberty-base-source.zip buildspec.yml
aws s3 cp liberty-base-source.zip s3://berrkonic-states/codebuild/liberty-base-image-builder/
aws s3 cp liberty-base-source.zip s3://berrkonic-states/codebuild/liberty-base-image-builder/ --region us-east-1

# Verify
aws s3 ls s3://berrkonic-states/codebuild/liberty-base-image-builder/
```

## Source Section:

***Source provider:*** Amazon S3
Bucket: xxxxxxxxxxx
***S3 object key or S3 folder:*** codebuild/liberty-base-image-builder/liberty-base-source.zip

Buildspec Section:
Build specifications: Use a buildspec file
Buildspec name: buildspec.yml
Source version: Leave this field EMPTY (delete "version-2")

## bucket permission issues, ad s3 permission to  the codebuild
```sh
aws iam put-role-policy \
  --role-name codebuild-liberty-base-image-builder-s3-service-role \
  --policy-name S3ReadAccess \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:GetObjectVersion",
        "s3:ListBucket",
        "s3:ListBucketVersions"
      ],
      "Resource": [
        "arn:aws:s3:::berrkonic-states",
        "arn:aws:s3:::berrkonic-states/*"
      ]
    }]
  }'
```



version: 0.2

env:
  variables:
    LIBERTY_VERSION: "26.0.0.1"
    ECR_REPO: "liberty"
    JAVA_TAG: "java21-openj9-ubi-minimal"

phases:
  pre_build:
    commands:
      - echo "Getting AWS Account ID..."
      - export AWS_ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
      - export AWS_DEFAULT_REGION=${AWS_DEFAULT_REGION:-us-east-1}
      - export IMAGE_TAG="${LIBERTY_VERSION}-${JAVA_TAG}"
      - echo "Account ID=$AWS_ACCOUNT_ID"
      - echo "Region=$AWS_DEFAULT_REGION"
      - echo "Image Tag=$IMAGE_TAG"
      - echo "Logging in to Amazon ECR..."
      - aws ecr get-login-password --region $AWS_DEFAULT_REGION | docker login --username AWS --password-stdin $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com
      - echo "Checking if ECR repository exists..."
      - aws ecr describe-repositories --repository-names $ECR_REPO 2>/dev/null || aws ecr create-repository --repository-name $ECR_REPO

  build:
    commands:
      - echo "Building Liberty base image..."
      - echo "Version=$LIBERTY_VERSION"
      - echo "Java Tag=$JAVA_TAG"
      - echo "Base image from IBM Container Registry"
      - |
        cat > Dockerfile <<'DOCKERFILEEOF'
        FROM icr.io/appcafe/websphere-liberty:kernel-java21-openj9-ubi-minimal

        LABEL maintainer="cdk Middleware Team" \
              version="26.0.0.1" \
              description="Minimal Liberty base image - feature cache via CodeArtifact"

        USER root

        RUN microdnf install -y curl unzip tar gzip && \
            microdnf clean all && \
            chown -R 1001:0 /opt/ibm/wlp && \
            chmod -R g+rw /opt/ibm/wlp

        USER 1001

        EXPOSE 9080 9443

        WORKDIR /opt/ibm/wlp

        CMD ["/opt/ibm/wlp/bin/server", "run", "defaultServer"]
        DOCKERFILEEOF
      - echo "Building Docker image..."
      - docker build -t liberty:$IMAGE_TAG .
      - echo "Tagging images..."
      - docker tag liberty:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - docker tag liberty:$IMAGE_TAG $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$JAVA_TAG-latest

  post_build:
    commands:
      - echo "Pushing image to ECR..."
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG
      - docker push $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$JAVA_TAG-latest
      - echo "Liberty base image successfully pushed to ECR"
      - echo "Image URI - $AWS_ACCOUNT_ID.dkr.ecr.$AWS_DEFAULT_REGION.amazonaws.com/$ECR_REPO:$IMAGE_TAG"
