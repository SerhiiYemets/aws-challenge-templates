# GitHub Actions → S3 CI/CD Template

VERIFIED ✅

Pipeline:

Git push to main
→ GitHub Actions
→ AWS authentication
→ aws s3 sync
→ S3 static website

## Files

- .github/workflows/deploy.yml
- site/index.html

## Required GitHub Secrets

- AWS_ACCESS_KEY_ID
- AWS_SECRET_ACCESS_KEY

## Required GitHub Variables

- AWS_REGION
- S3_BUCKET

## Quick setup

Set variables:

```bash
gh variable set AWS_REGION --body "eu-central-1"
gh variable set S3_BUCKET --body "YOUR_BUCKET_NAME"

Set secrets:

gh secret set AWS_ACCESS_KEY_ID
gh secret set AWS_SECRET_ACCESS_KEY
Trigger

Push to branch:

main

Change this in .github/workflows/deploy.yml if the task requires another branch.

Deploy command used by GitHub Actions
aws s3 sync ./site s3://$S3_BUCKET --delete
Verification

Check workflow:

gh run list --limit 3

Check website:

curl http://YOUR_BUCKET_NAME.s3-website.eu-central-1.amazonaws.com
Challenge checklist

Before SEP Verify:

Check exact bucket name.
Check exact branch name.
Check AWS region.
Check GitHub Actions run is successful.
Check files exist in S3.
Check website/content manually.
Only then run SEP verification.
