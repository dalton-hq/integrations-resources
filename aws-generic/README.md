# Dalton AWS Generic Integration

CloudFormation templates for integrating AWS accounts with Dalton for read-only monitoring and analysis of multiple AWS services.

## Overview

This integration allows Dalton to access your AWS account(s) with read-only permissions for selected AWS services. It uses IAM cross-account roles with External ID for secure access.

## Supported Services

| Service | Description |
|---------|-------------|
| S3 | Object storage - buckets, objects, policies |
| RDS | Relational databases - instances, logs, snapshots |
| EC2 | Virtual servers - instances, volumes, networking |
| Lambda | Serverless functions - configurations, metrics |
| CloudWatch | Monitoring - logs, metrics, alarms |
| DynamoDB | NoSQL database - tables, items, streams |
| ECS | Container service - clusters, services, tasks |
| SNS | Notification service - topics, subscriptions |
| SQS | Message queuing - queues, messages |
| Secrets Manager | Secrets metadata only (values are explicitly denied) |
| Route 53 | DNS service - hosted zones, records, health checks |

## Templates

### Single Account (`cloud-formation/single-account.yaml`)

For deploying to individual AWS accounts.

**Quick Deploy URL:**
```
https://console.aws.amazon.com/cloudformation/home#/stacks/create/review?templateURL=https://cf-templates-w9lau7h9v2y20-eu-north-1.s3.eu-north-1.amazonaws.com/aws-generic/single-account.yaml
```

**Role Name Format:** `Dalton-AWS-Viewer-{AccountId}`

### Multi-Account StackSet (`cloud-formation/stackset.yaml`)

For deploying across multiple accounts in an AWS Organization using CloudFormation StackSets.

**Role Name:** `Dalton-AWS-Viewer` (consistent across all accounts)

## Parameters

| Parameter | Required | Default | Description |
|-----------|----------|---------|-------------|
| ExternalId | Yes | - | Unique identifier provided by Dalton for secure role assumption |
| EnableS3 | No | false | Enable S3 read access |
| EnableRDS | No | false | Enable RDS read access |
| EnableEC2 | No | false | Enable EC2 read access |
| EnableLambda | No | false | Enable Lambda read access |
| EnableCloudWatch | No | false | Enable CloudWatch Logs & Metrics read access |
| EnableDynamoDB | No | false | Enable DynamoDB read access |
| EnableECS | No | false | Enable ECS read access |
| EnableSNS | No | false | Enable SNS read access |
| EnableSQS | No | false | Enable SQS read access |
| EnableSecretsManager | No | false | Enable Secrets Manager metadata access |
| EnableRoute53 | No | false | Enable Route 53 read access |

## Deployment Methods

### Method 1: Single Account via Console

1. Click the Quick Deploy URL above (or use the link in Dalton UI)
2. The CloudFormation console opens with pre-filled parameters
3. Review the External ID (do not modify)
4. Select which services to enable
5. Acknowledge IAM resource creation
6. Click "Create Stack"
7. Copy the Role ARN from stack outputs back to Dalton

### Method 2: Single Account via AWS CLI

```bash
aws cloudformation create-stack \
  --stack-name dalton-aws-integration \
  --template-url https://cf-templates-w9lau7h9v2y20-eu-north-1.s3.eu-north-1.amazonaws.com/aws-generic/single-account.yaml \
  --parameters \
    ParameterKey=ExternalId,ParameterValue=YOUR_EXTERNAL_ID \
    ParameterKey=EnableS3,ParameterValue=true \
    ParameterKey=EnableCloudWatch,ParameterValue=true \
  --capabilities CAPABILITY_NAMED_IAM
```

### Method 3: Multi-Account via StackSets

1. Ensure trusted access is enabled for CloudFormation StackSets in AWS Organizations
2. Create a StackSet with service-managed permissions:

```bash
aws cloudformation create-stack-set \
  --stack-set-name DaltonAWSIntegration \
  --template-url https://cf-templates-w9lau7h9v2y20-eu-north-1.s3.eu-north-1.amazonaws.com/aws-generic/stackset.yaml \
  --permission-model SERVICE_MANAGED \
  --auto-deployment Enabled=true,RetainStacksOnAccountRemoval=false \
  --parameters \
    ParameterKey=ExternalId,ParameterValue=YOUR_EXTERNAL_ID \
    ParameterKey=EnableS3,ParameterValue=true \
    ParameterKey=EnableCloudWatch,ParameterValue=true \
  --capabilities CAPABILITY_NAMED_IAM
```

3. Create stack instances for target OUs:

```bash
aws cloudformation create-stack-instances \
  --stack-set-name DaltonAWSIntegration \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1
```

## Security

### Access Controls

- **Read-only access:** All permissions are strictly read-only
- **External ID:** Required for cross-account role assumption, prevents confused deputy attacks
- **Secrets Manager:** Secret values are explicitly denied - only metadata is accessible
- **Least privilege:** Only policies for enabled services are created

### Trust Policy

The IAM role trusts only Dalton's AWS account (`767398131998`) with the External ID condition:

```json
{
  "Effect": "Allow",
  "Principal": {
    "AWS": "arn:aws:iam::767398131998:root"
  },
  "Action": "sts:AssumeRole",
  "Condition": {
    "StringEquals": {
      "sts:ExternalId": "<your-external-id>"
    }
  }
}
```

## Outputs

After deployment, the stack provides:

| Output | Description |
|--------|-------------|
| RoleArn | The ARN of the created IAM role (required for Dalton setup) |
| RoleName | The name of the created IAM role |
| AccountId | The AWS Account ID where the role was created |
| ExternalId | The External ID used for this integration |

## Updating Services

To add or remove services after initial deployment:

1. Update the stack with modified parameters
2. CloudFormation will automatically add/remove the corresponding policies

```bash
aws cloudformation update-stack \
  --stack-name dalton-aws-integration \
  --use-previous-template \
  --parameters \
    ParameterKey=ExternalId,UsePreviousValue=true \
    ParameterKey=EnableS3,ParameterValue=true \
    ParameterKey=EnableRDS,ParameterValue=true \
    ParameterKey=EnableDynamoDB,ParameterValue=true \
  --capabilities CAPABILITY_NAMED_IAM
```

## Cleanup

To remove the integration:

```bash
# Single account
aws cloudformation delete-stack --stack-name dalton-aws-integration

# StackSet (deletes from all accounts)
aws cloudformation delete-stack-instances \
  --stack-set-name DaltonAWSIntegration \
  --deployment-targets OrganizationalUnitIds=ou-xxxx-xxxxxxxx \
  --regions us-east-1 \
  --no-retain-stacks

aws cloudformation delete-stack-set --stack-set-name DaltonAWSIntegration
```

## Troubleshooting

### Stack Creation Fails

- **CAPABILITY_NAMED_IAM required:** Ensure you acknowledge IAM resource creation
- **Role name conflict:** If a role with the same name exists, delete it first or use a different account

### Cannot Assume Role

- **External ID mismatch:** Verify the External ID matches exactly
- **Trust policy:** Confirm the role trusts account `767398131998`
- **Role ARN:** Double-check the Role ARN copied to Dalton

### Missing Permissions

- **Service not enabled:** Update the stack to enable the required service
- **Policy not attached:** Check that the corresponding managed policy was created
