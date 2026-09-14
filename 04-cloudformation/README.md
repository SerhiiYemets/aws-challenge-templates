AWS Challenge — CloudFormation Gitea

Reusable and verified CloudFormation template for deploying a highly available Gitea application on AWS.

## Status

VERIFIED ✅

The complete stack was successfully deployed and tested in eu-central-1.

Verified:

- CloudFormation stack reached CREATE_COMPLETE
- 2 EC2 instances were running in the Auto Scaling Group
- both ALB targets reached healthy
- Gitea was accessible through the Application Load Balancer
- Gitea used RDS MySQL
- shared Gitea data was stored on EFS
- repeated requests through the ALB returned the configured Gitea application
- Auto Scaling Instance Refresh successfully replaced EC2 instances
- complete stack deletion was tested

---

## Architecture

text
                         Internet
                            |
                            v
                 Application Load Balancer
                            |
                      Target Group
                       /        \
                      v          v
                   EC2 #1      EC2 #2
                   Docker      Docker
                   Gitea       Gitea
                      \          /
                       \        /
                    +---+------+---+
                    |              |
                    v              v
                RDS MySQL         EFS
                database      shared /data

                 Auto Scaling Group
                   Min: 2
                   Max: 4


Network architecture:

text
VPC
├── Public Subnet 1
│   ├── ALB
│   └── NAT Gateway
│
├── Public Subnet 2
│   └── ALB
│
├── Private Subnet 1
│   ├── EC2
│   ├── RDS
│   └── EFS Mount Target
│
└── Private Subnet 2
    ├── EC2
    ├── RDS
    └── EFS Mount Target


---

## Main template

text
gitea-full.yml


The template creates:

- VPC
- Internet Gateway
- 2 public subnets
- 2 private subnets
- NAT Gateway
- Elastic IP
- public/private route tables
- ALB Security Group
- EC2 Security Group
- RDS Security Group
- EFS Security Group
- Application Load Balancer
- Target Group
- HTTP Listener
- RDS MySQL
- EFS
- 2 EFS Mount Targets
- EC2 IAM Role
- Instance Profile
- Launch Template
- Docker
- Docker Compose
- Gitea
- Auto Scaling Group
- Auto Scaling Policy
- CloudWatch Dashboard

---

## Validate template

Always validate before deployment:

bash
aws cloudformation validate-template \
  --template-body file://04-cloudformation/gitea-full.yml \
  --region eu-central-1


---

## Create Change Set

Use a Change Set before creating the stack:

bash
aws cloudformation create-change-set \
  --stack-name challenge-gitea-full \
  --change-set-name preflight \
  --change-set-type CREATE \
  --template-body file://04-cloudformation/gitea-full.yml \
  --parameters ParameterKey=DBPassword,ParameterValue='<DB_PASSWORD>' \
  --capabilities CAPABILITY_NAMED_IAM \
  --role-arn <CLOUDFORMATION_ROLE_ARN> \
  --region eu-central-1


Check it:

bash
aws cloudformation describe-change-set \
  --stack-name challenge-gitea-full \
  --change-set-name preflight \
  --region eu-central-1 \
  --query '{Status:Status,ExecutionStatus:ExecutionStatus,Reason:StatusReason,Changes:length(Changes)}' \
  --output table


Expected:

text
Status          CREATE_COMPLETE
ExecutionStatus AVAILABLE


Execute:

bash
aws cloudformation execute-change-set \
  --stack-name challenge-gitea-full \
  --change-set-name preflight \
  --region eu-central-1


Wait:

bash
aws cloudformation wait stack-create-complete \
  --stack-name challenge-gitea-full \
  --region eu-central-1


---

## Get stack outputs

bash
aws cloudformation describe-stacks \
  --stack-name challenge-gitea-full \
  --region eu-central-1 \
  --query 'Stacks[0].Outputs' \
  --output table


Important outputs:

- LoadBalancerUrl
- LoadBalancerDnsName
- DatabaseEndpoint
- FileSystemId
- TargetGroupArn
- AutoScalingGroupName
- DashboardName

---

## Verify ALB targets

```bash
TG_ARN=$(aws cloudformation describe-stacks \
  --stack-name challenge-gitea-full \
  --region eu-central-1 \
  --
[9/14/26 5:59 PM] Serëga Vladimirovich: query 'Stacks[0].Outputs[?OutputKey==`TargetGroupArn`].OutputValue' \
  --output text)

aws elbv2 describe-target-health \
  --target-group-arn "$TG_ARN" \
  --region eu-central-1 \
  --query 'TargetHealthDescriptions[].{Instance:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason}' \
  --output table


Expected final state:

```text
Port   State
80     healthy
80     healthy


---

## Verify Gitea

Get ALB URL:

bash
ALB_URL=$(aws cloudformation describe-stacks \
  --stack-name challenge-gitea-full \
  --region eu-central-1 \
  --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerUrl`].OutputValue' \
  --output text)


Check the application:

bash
curl -s "$ALB_URL" \
  | grep -o '<title>[^<]*</title>' \
  | head -n 1


After Gitea installation/configuration the result should contain the configured application title rather than:

text
Installation - Gitea: Git with a cup of tea


Test repeated requests through the ALB:

bash
for i in {1..10}; do
  curl -s "$ALB_URL" \
    | grep -o '<title>[^<]*</title>' \
    | head -n 1
done


All responses should return the configured Gitea application.

---

## Important lessons

### Amazon Linux 2023 and curl

Amazon Linux 2023 may already contain curl-minimal.

Do NOT blindly install:

bash
dnf install -y curl


because curl can conflict with curl-minimal and cause cloud-init/UserData to stop when set -e is enabled.

The verified template installs:

bash
dnf install -y \
  docker \
  amazon-efs-utils


The existing curl implementation is then used for downloading Docker Compose.

### CloudFormation CREATE_COMPLETE is not enough

A stack can reach:

text
CREATE_COMPLETE


while the application itself is unhealthy.

Always verify:

text
CloudFormation
      ↓
ASG instances
      ↓
Target Group health
      ↓
HTTP request through ALB
      ↓
actual application


### Build dependencies explicitly

The ASG depends on:

- RDS
- EFS Mount Targets
- ALB Listener

This prevents application instances from starting before important infrastructure is available.

### Two Gitea instances and shared EFS

Both Gitea instances use the same EFS /data.

During the initial Gitea installation one instance may reload the new configuration while another instance is still running with its old in-memory configuration.

Symptom:

text
AWS Challenge Gitea
Installation - Gitea...
AWS Challenge Gitea
Installation - Gitea...


Restart/replacement of the instances makes both instances load the shared configuration.

### ASG rolling updates

Changing a Launch Template does not automatically guarantee that all existing EC2 instances are immediately replaced.

The template therefore uses:

yaml
UpdatePolicy:
  AutoScalingRollingUpdate:
    MinInstancesInService: 1
    MaxBatchSize: 1
    PauseTime: PT5M
    WaitOnResourceSignals: false


An Instance Refresh can also be used when necessary.

### Never verify too early

Wait until resources are ready before final validation.

Especially check:

- CloudFormation status
- RDS availability
- EFS Mount Targets
- EC2/ASG state
- Target Group health
- application HTTP response

This is especially important when verification attempts are limited or penalized.

---

## Delete stack

The stack contains resources that can generate AWS charges, including:

- NAT Gateway
- Application Load Balancer
- EC2
- RDS

Delete the training stack when finished:

bash
aws cloudformation delete-stack \
  --stack-name challenge-gitea-full \
  --region eu-central-1


Wait for complete deletion:

bash
aws cloudformation wait stack-delete-complete \
  --stack-name challenge-gitea-full \
  --region eu-central-1


---

## Challenge workflow

For a timed challenge:

```text
1. Read the task carefully
2. Extract EXACT required resource names
3. Extract required region / AZs
4. Adapt gitea-full.yml
5. Validate template
6. Create/check Change Set if time permits
7. Deploy
8. Wait for completion
9. Verify Target Group health
10. Verify application through ALB
11. Verify required RDS/EFS/A
[9/14/26 5:59 PM] Serëga Vladimirovich: SG resources
12. Only then run the challenge validator
`

Do not submit verification only because CloudFormation reports CREATE_COMPLETE.

---

## Result

This template was deployed end-to-end during AWS Challenge preparation and successfully served Gitea through an Application Load Balancer using two Auto Scaling EC2 instances with shared EFS storage and RDS MySQL.

VERIFIED ✅
