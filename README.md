# AWS Resource Cleanup v2

All-in-one CloudFormation template that cleans up unused AWS resources, tracks how much money was saved, and scans for potential savings — with a built-in CloudWatch dashboard.

## Architecture

```
                          ┌─────────────────────┐
                          │   EventBridge Rule   │
                          │  (every 2 hours)     │
                          └─────────┬───────────┘
                                    │
                                    ▼
┌───────────────┐       ┌─────────────────────┐       ┌─────────────┐
│  CloudWatch   │◄──────│  Cleanup Lambda     │──────►│  SNS Topic  │
│  Metrics      │       │  (delete + track $) │       │  (emails)   │
└───────┬───────┘       └─────────────────────┘       └─────────────┘
        │                         │
        ▼                         ▼
┌───────────────┐       Deletes: Stacks, EC2, S3,
│  CloudWatch   │       EBS, Snapshots, EIPs
│  Dashboard    │
└───────────────┘
        ▲
        │
┌───────┴───────┐       ┌─────────────────────┐
│  CloudWatch   │◄──────│  Savings Scanner    │
│  Metrics      │       │  (scan only, no     │
└───────────────┘       │   deletes)          │
                        └─────────┬───────────┘
                                  │
                          ┌───────┴───────────┐
                          │  EventBridge Rule  │
                          │  (once per day)    │
                          └───────────────────┘
```

## What It Cleans Up

| Resource | Behavior |
|----------|----------|
| CloudFormation Stacks | Force deleted after configurable time (default 8 hours) — disables termination protection, empties S3 buckets, retries failed deletes |
| EC2 Instances | Terminated after configurable time (default 8 hours) — checks all states: pending, running, stopping, stopped |
| S3 Buckets | Emptied and deleted after configurable time (handles versioned objects) |
| EBS Volumes | Deleted immediately if unattached (`available` status) |
| EBS Snapshots | Deleted if older than 7 days and not in use by AMIs |
| Elastic IPs | Released immediately if unassociated |

## What's New in v2

- Tracks actual dollar savings for every resource deleted (EC2, EBS, snapshots, EIPs)
- Potential Savings Scanner — runs daily, reports what could be saved without deleting anything
- Built-in CloudWatch Dashboard with actual savings, potential savings, cleanup activity, protected vs deleted, and failure tracking
- Email notifications include dollar amounts
- All stack resources are tagged with `do-not-delete` so the stack never cleans up itself

## How Protection Works

Resources are protected from deletion if any of these apply:

- Resource name or `Name` tag contains the exclude string (default: `do-not-delete`)
- Any tag value on the resource contains the exclude string
- CloudFormation stack has termination protection enabled AND is tagged with the exclude string
- The cleanup stack itself is always protected (matches keywords like `resourcecleanup`, `cleaner`, `cleanup-stack`)

## Force Delete (CloudFormation Stacks)

Stacks that are not protected get force deleted. This means:

1. Termination protection is disabled automatically
2. S3 buckets inside the stack are emptied (including versioned objects)
3. If the first delete attempt fails (`DELETE_FAILED`), it retries with `RetainResources` to skip problematic resources
4. Stacks already in `DELETE_FAILED` state are picked up and force deleted on the next run

To protect a stack from force delete, either:
- Include `do-not-delete` in the stack name
- Add a tag with `do-not-delete` in the value
- Both of these will prevent the stack from being touched

## Parameters

| Parameter | Default | Description |
|-----------|---------|-------------|
| `MinRunningTimeHours` | `8` | Hours before stacks, instances, and buckets are deleted (1–168) |
| `NotificationTimeHours` | `4` | Hours before deletion to send a warning email (1–24) |
| `ExcludeString` | `do-not-delete` | Resources with this string in their name/tags are protected |
| `NotificationEmail` | *(required)* | Email address for deletion notifications |
| `ScheduleExpression` | `rate(2 hours)` | EventBridge schedule for how often cleanup runs |
| `LambdaTimeout` | `900` | Lambda timeout in seconds (60–900) |

## Deployment

### AWS Console

1. Go to CloudFormation → Create Stack → Upload a template file
2. Upload `aws-resource-cleanup-v2.yaml`
3. Fill in the parameters (at minimum, provide your email)
4. Acknowledge IAM resource creation
5. Create stack
6. Confirm the SNS subscription email you receive

### AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name aws-resource-cleanup-v2 \
  --template-body file://aws-resource-cleanup-v2.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your@email.com \
    ParameterKey=MinRunningTimeHours,ParameterValue=8 \
    ParameterKey=ExcludeString,ParameterValue=do-not-delete \
  --capabilities CAPABILITY_NAMED_IAM
```

### CloudShell

1. Log into the AWS Console
2. Open CloudShell (terminal icon, top right)
3. Make sure you're in the correct region
4. Click Actions → Upload file → select `aws-resource-cleanup-v2.yaml`
5. If re-uploading, delete the old file first:

```bash
rm aws-resource-cleanup-v2.yaml
```

6. Run:

```bash
aws cloudformation create-stack \
  --stack-name aws-resource-cleanup-v2 \
  --template-body file://aws-resource-cleanup-v2.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your@email.com \
    ParameterKey=MinRunningTimeHours,ParameterValue=8 \
    ParameterKey=ExcludeString,ParameterValue=do-not-delete \
  --capabilities CAPABILITY_NAMED_IAM
```

## What Gets Deployed

| Resource | Type | Description |
|----------|------|-------------|
| ResourceCleanupFunction | `AWS::Lambda::Function` | Python 3.13, 512 MB — deletes resources and tracks cost savings |
| PotentialSavingsFunction | `AWS::Lambda::Function` | Python 3.13, 256 MB — scans for potential savings without deleting |
| LambdaExecutionRole | `AWS::IAM::Role` | Permissions for both Lambda functions across all managed services |
| CleanupNotificationTopic | `AWS::SNS::Topic` | Sends email notifications for warnings, cleanup summaries, and savings reports |
| CleanupScheduleRule | `AWS::Events::Rule` | Triggers cleanup Lambda on schedule (default every 2 hours) |
| SavingsScanSchedule | `AWS::Events::Rule` | Triggers savings scanner daily |
| CleanupInvokePermission | `AWS::Lambda::Permission` | Allows EventBridge to invoke the cleanup Lambda |
| SavingsScanPermission | `AWS::Lambda::Permission` | Allows EventBridge to invoke the savings scanner Lambda |
| CostDashboard | `AWS::CloudWatch::Dashboard` | Built-in dashboard with savings, activity, and failure widgets |

All taggable resources are tagged with `Name: {stack-name}-do-not-delete` so the cleanup stack never deletes itself.

## CloudWatch Dashboard

After deployment, find the dashboard URL in the stack Outputs tab, or go to:

CloudWatch → Dashboards → `{stack-name}-Dashboard`

The dashboard shows:

- Actual monthly savings (from deleted resources)
- Potential monthly savings (from the daily scanner)
- Savings breakdown by resource type (EC2, EBS, snapshots, EIPs)
- Savings trend over time
- Cleanup activity (daily counts of deleted resources)
- Protected vs deleted comparison
- Failed deletions by type
- Recent activity log

## Cost Estimation

The stack estimates monthly savings using these prices:

| Resource | Price Used |
|----------|-----------|
| t2.micro | $8.35/mo |
| t3.micro | $7.49/mo |
| t3.small | $14.98/mo |
| t3.medium | $29.95/mo |
| m5.large | $69.12/mo |
| gp2 volume | $0.10/GB/mo |
| gp3 volume | $0.08/GB/mo |
| io1/io2 volume | $0.125/GB/mo |
| Snapshot | $0.05/GB/mo |
| Elastic IP (unassociated) | $3.65/mo |

Unknown instance types default to $30/mo.

## Notifications

The stack sends three types of email notifications:

1. **Warning** — sent during the notification window before a stack is deleted
2. **Cleanup Summary** — sent after each run if resources were deleted, includes dollar savings
3. **Potential Savings Report** — sent daily with a breakdown of what could be saved

## Potential Savings Scanner

Runs once a day and reports:

- Unprotected EC2 instances and their estimated monthly cost
- Stopped instances (still incurring EBS costs)
- Unattached EBS volumes
- Old snapshots (>7 days, not used by AMIs)
- Unassociated Elastic IPs

The scanner never deletes anything — it only reports.

## IAM Permissions

The Lambda role has broad permissions to handle CloudFormation stack deletion (stacks can contain any resource type). Key services:

EC2, CloudFormation, S3, SNS, CloudWatch, Lambda, API Gateway, DynamoDB, EventBridge, CloudFront, Step Functions, SES, RDS, EKS, ECR, SQS, IAM, Route53, Secrets Manager, Auto Scaling, ELB, ECS, ElastiCache, EFS, MSK, KMS, SSM, Image Builder

## How to Remove the Stack

To delete the cleanup stack itself and all its resources:

### AWS Console

1. Go to CloudFormation → Stacks
2. Select the `aws-resource-cleanup-v2` stack
3. Click Delete
4. Confirm deletion

### AWS CLI

```bash
aws cloudformation delete-stack --stack-name aws-resource-cleanup-v2
```

### CloudShell

```bash
aws cloudformation delete-stack --stack-name aws-resource-cleanup-v2
```

This removes the Lambda functions, IAM role, SNS topic, EventBridge rules, and the CloudWatch dashboard. CloudWatch metrics and Lambda logs are not deleted automatically — they expire based on retention settings.

## How Much Does This Stack Cost to Run?

| Resource | Cost |
|----------|------|
| Cleanup Lambda | Free tier: 1M requests + 400,000 GB-seconds/month. Running every 2 hours = ~360 invocations/month — well within free tier |
| Savings Scanner Lambda | Runs once/day = ~30 invocations/month — free tier |
| SNS Email Notifications | Free for email delivery |
| EventBridge Rules | Free — no charge for scheduled rules |
| CloudWatch Dashboard | $3.00/month per dashboard (first 3 dashboards are free) |
| CloudWatch Metrics | Free tier: 10 custom metrics. This stack publishes ~15+ metrics — roughly $0.30/month for the extras |
| IAM Role | Free |

**Estimated total: $0–$3.30/month** depending on whether you're within free tier limits for dashboards and metrics.

## Tips

- Set `ExcludeString` to something your team uses consistently, like `prod` or `permanent`
- Start with a longer `MinRunningTimeHours` (e.g., 24) and reduce once you're comfortable
- Check CloudWatch Logs at `/aws/lambda/{stack-name}-CleanupFunction` for detailed execution logs
- The savings scanner runs daily at a fixed rate — check the dashboard the next day for potential savings data

## How to Test Manually

You don't have to wait for the schedule. You can invoke either Lambda manually:

### Test the Cleanup Lambda

1. Go to Lambda → Functions → `{stack-name}-CleanupFunction`
2. Click Test
3. Create a test event with `{}` as the body
4. Click Test again
5. Check the execution log for what was deleted/protected

### Test the Savings Scanner

1. Go to Lambda → Functions → `{stack-name}-SavingsScanner`
2. Click Test
3. Create a test event with `{}` as the body
4. Click Test
5. Check your email for the potential savings report

### Test via CLI

```bash
aws lambda invoke \
  --function-name {stack-name}-CleanupFunction \
  --payload '{}' \
  output.json
```

```bash
aws lambda invoke \
  --function-name {stack-name}-SavingsScanner \
  --payload '{}' \
  output.json
```

## Limitations

Things this stack does NOT clean up:

| Resource | Why |
|----------|-----|
| NAT Gateways | Not included — can cost $32+/month each. Delete manually |
| VPC Endpoints | Not included |
| Route53 Hosted Zones | Not included — could break DNS |
| CloudFront Distributions | Only deleted if inside a CloudFormation stack |
| RDS Instances | Only deleted if inside a CloudFormation stack |
| EKS/ECS Clusters | Only deleted if inside a CloudFormation stack |
| Lambda Functions | Only deleted if inside a CloudFormation stack (standalone functions are not touched) |
| DynamoDB Tables | Only deleted if inside a CloudFormation stack |
| Secrets Manager | Only deleted if inside a CloudFormation stack |
| Resources in other regions | Only cleans the region it's deployed in |

## Multi-Region

This stack only cleans up resources in the region where it's deployed. If you have resources in multiple regions, deploy the stack in each region:

```bash
aws cloudformation create-stack \
  --stack-name aws-resource-cleanup-v2 \
  --template-body file://aws-resource-cleanup-v2.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your@email.com \
  --capabilities CAPABILITY_NAMED_IAM \
  --region us-east-1
```

```bash
aws cloudformation create-stack \
  --stack-name aws-resource-cleanup-v2 \
  --template-body file://aws-resource-cleanup-v2.yaml \
  --parameters \
    ParameterKey=NotificationEmail,ParameterValue=your@email.com \
  --capabilities CAPABILITY_NAMED_IAM \
  --region eu-west-1
```

You'll get a separate dashboard and email notifications per region.

## Troubleshooting

| Problem | Cause | Fix |
|---------|-------|-----|
| Not getting emails | SNS subscription not confirmed | Check your inbox (and spam) for the confirmation email from AWS. Click the confirm link |
| Stack not being deleted | Stack name or tag contains `do-not-delete` | Remove the tag or rename the stack |
| Stack stuck in `DELETE_FAILED` | Resources inside the stack can't be deleted (e.g., non-empty S3 bucket, IAM role in use) | The cleanup will retry with `RetainResources` on the next run. Check CloudWatch Logs for the specific error |
| Dashboard shows no data | No cleanup has run yet, or no resources to clean | Invoke the Lambda manually (see "How to Test Manually") or wait for the next scheduled run |
| Potential savings report is empty | All resources are tagged with `do-not-delete` or account is clean | That's a good thing |
| Lambda times out | Too many resources to process in 900 seconds | Rare — only happens with thousands of resources. Run it more frequently to keep the backlog small |
| Instances not being terminated | Instance is younger than `MinRunningTimeHours` | Wait for the timer, or lower the parameter value |
| Protected stack still got deleted | The exclude string check is case-insensitive but must be in the name or a tag value | Make sure `do-not-delete` (or your custom string) is in the stack name or any tag value, not just the tag key |
