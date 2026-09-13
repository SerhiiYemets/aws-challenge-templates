# 03 — AWS CodePipeline → CodeBuild → S3

Verified reusable CI/CD template for AWS Challenge preparation.

## Architecture

GitHub
→ AWS CodeConnections
→ AWS CodePipeline
→ AWS CodeBuild
→ Amazon S3 static website

## Repository

Repository:
SerhiiYemets/aws-challenge-templates

Branch:
main

Template directory:
03-codepipeline/

## AWS Resources

Region:
eu-central-1

Pipeline:
challenge-codepipeline

CodeBuild project:
challenge-codepipeline-build

CodePipeline role:
codepipeline-challenge-role

CodeBuild role:
codebuild-challenge-role

GitHub connection:
github-challenge-connection

Artifact bucket:
serhii-codepipeline-artifacts-984263476780

Deployment bucket:
serhii-cicd-test-984263476780

## CodeBuild Configuration

Source:
CODEPIPELINE

Buildspec:
03-codepipeline/buildspec.yml

Artifacts:
CODEPIPELINE

Image:
aws/codebuild/standard:7.0

Compute:
BUILD_GENERAL1_SMALL

## Important: Monorepo Working Directory

CodeBuild starts from the repository root.

The buildspec is located in:

03-codepipeline/buildspec.yml

but this does NOT make 03-codepipeline the working directory.

Therefore the buildspec explicitly uses:

cd 03-codepipeline

Do not remove this unless the source structure changes.

## Important: Separate Artifact Bucket

CodePipeline artifacts use:

serhii-codepipeline-artifacts-984263476780

The deployed website uses:

serhii-cicd-test-984263476780

Keep these buckets separate.

The deployment command uses:

aws s3 sync . s3://serhii-cicd-test-984263476780 --delete

Using the website bucket as the CodePipeline artifact store could cause deployment cleanup to interfere with pipeline artifacts.

## Required IAM

terraform-user:
- AWSCodePipeline_FullAccess
- CodeConnections access
- iam:PassRole for codepipeline-challenge-role

codepipeline-challenge-role:
- Use GitHub connection
- Start/monitor challenge-codepipeline-build
- Read/write CodePipeline artifact bucket

codebuild-challenge-role:
- CloudWatch Logs access
- Read/write CodePipeline artifact bucket
- Deploy access to website S3 bucket

## Create Pipeline

From repository root:

aws codepipeline create-pipeline \
  --cli-input-json file://03-codepipeline/pipeline.json \
  --region eu-central-1

## Check Pipeline

aws codepipeline get-pipeline-state \
  --name challenge-codepipeline \
  --region eu-central-1 \
  --query 'stageStates[].{Stage:stageName,Action:actionStates[0].actionName,Status:actionStates[0].latestExecution.status,Summary:actionStates[0].latestExecution.summary}' \
  --output table

Expected:

Source → Succeeded
Build  → Succeeded

## Verify Deployment

curl http://serhii-cicd-test-984263476780.s3-website.eu-central-1.amazonaws.com

Expected content:

AWS CodePipeline works!

## Challenge Checklist

Before SEP Verify:

1. Check exact resource names from the task.
2. Check AWS region.
3. Check IAM roles and iam:PassRole.
4. Check GitHub connection is AVAILABLE.
5. Check CodeBuild source is CODEPIPELINE.
6. Check buildspec path.
7. Remember CodeBuild starts at repository root.
8. Wait until Source = Succeeded.
9. Wait until Build = Succeeded.
10. Verify the final deployed resource manually.
11. Only then run SEP Verify.

## Status

VERIFIED ✅

Tested flow:

GitHub → CodePipeline → CodeBuild → S3

Source: Succeeded
Build: Succeeded
S3 deployment: Verified
