# AWS CodeBuild Template

VERIFIED ✅

## Flow

GitHub repository
→ AWS CodeBuild
→ buildspec.yml
→ Build phases
→ Verification

## Files

- buildspec.yml
- index.html

## Buildspec

CodeBuild checks out the repository root.

If this template is stored in a subdirectory, explicitly change into it:

```yaml
pre_build:
  commands:
    - cd 02-codebuild
    - pwd
    - ls -la

Do not assume CodeBuild starts inside the directory containing buildspec.yml.

Start build
aws codebuild start-build \
  --project-name challenge-codebuild-test \
  --region eu-central-1
Check build
aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --query 'builds[0].[buildStatus,currentPhase]' \
  --output table \
  --region eu-central-1

Expected:

SUCCEEDED
COMPLETED
If build fails

Do not immediately restart it.

Check the failed phase:

aws codebuild batch-get-builds \
  --ids BUILD_ID \
  --query 'builds[0].phases[?phaseStatus==`FAILED`].[phaseType,contexts]' \
  --output json \
  --region eu-central-1
Challenge checklist

Before SEP Verify:

Check exact project name.
Check exact Git branch/repository.
Check exact buildspec.yml path.
Check IAM service role.
Check iam:PassRole if required.
Run one test build.
Wait for SUCCEEDED.
Inspect logs if needed.
Only then run SEP Verify.
