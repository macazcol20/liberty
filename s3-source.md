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
