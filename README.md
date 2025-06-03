# Falcon Container Sensor Deployment for ECS Fargate via GitHub Actions Workflow

## Overview

This guide demonstrates how to use a GitHub Actions Workflow to automate the deployment of the CrowdStrike Falcon Container sensor to your existing ECS Fargate applications. The workflow will:

1. Pull your application image(s) from ECR
2. Pull the latest Falcon Container sensor
3. Patch your application image(s) with the Falcon sensor
4. Create a new task definition with the patched image(s)
5. Deploy the secured application to ECS Fargate

---

## AWS Setup

### 1. Creating the IAM User for GitHub Actions

```bash
# Create the user
aws iam create-user --user-name github-actions-ecs

# Create access key
aws iam create-access-key --user-name github-actions-ecs
# IMPORTANT: Save the AccessKeyId and SecretAccessKey securely for use with GitHub Secrets

# Create policy document
cat > github-actions-policy.json << EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload",
        "ecr:PutImage",
        "ecr:DescribeRepositories",
        "ecs:RegisterTaskDefinition",
        "ecs:DescribeTaskDefinition",
        "ecs:RunTask",
        "ecs:DescribeTasks",
        "ecs:ListTasks",
        "ec2:DescribeSubnets",
        "ec2:DescribeSecurityGroups",
        "sts:GetCallerIdentity",
        "iam:PassRole"
      ],
      "Resource": "*"
    }
  ]
}
EOF

# Create and attach the policy
aws iam create-policy --policy-name GitHubActionsECSPolicy --policy-document file://github-actions-policy.json
aws iam attach-user-policy --user-name github-actions-ecs --policy-arn $(aws iam list-policies --query 'Policies[?PolicyName==`GitHubActionsECSPolicy`].Arn' --output text)
```

### 2. ECS Task Execution Role

Ensure your ECS task execution role has the necessary permissions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ecr:GetAuthorizationToken",
        "ecr:BatchCheckLayerAvailability",
        "ecr:GetDownloadUrlForLayer",
        "ecr:BatchGetImage",
        "logs:CreateLogStream",
        "logs:PutLogEvents"
      ],
      "Resource": "*"
    }
  ]
}
```

This role is typically created by attaching the AWS-managed policy `AmazonECSTaskExecutionRolePolicy`.

### 3. Prepare ECR Repository for Falcon Container Sensor

Create an ECR repository to store the Falcon Container sensor image:

```bash
# Create ECR repository for Falcon sensor
aws ecr create-repository --repository-name falcon-sensor/falcon-container --region=YOUR_AWS_REGION
```

---

## CrowdStrike Setup

### 1. Retrieve the Customer ID (CID)

1. Log in to the Falcon Console
2. Go to Host setup and management > Deploy > Sensor downloads
3. On the Sensor downloads page, in the How to install section, locate your customer ID.
4. Click copy to copy your CID with checksum to the clipboard.

### 2. CrowdStrike API Client Setup

Create a CrowdStrike API client with these minimum scopes:
- `Sensor Download`: Read
- `Falcon Images`: Download

To create this client:
1. Log in to the Falcon Console
2. Go to API Clients and Keys
3. Click "Add new API client"
4. Name it "Falcon Container Deployment"
5. Select the required scopes
6. Create the client and save the Client ID and Secret for use with GitHub Secrets

---

## Implementation Steps

### 1. Add the Workflow File to Your Repository

Create the following directory structure in your repository:
```
.github/
└── workflows/
    └── falcon-ecs-fargate-deployment.yml
```

The workflow file is already provided in this repository at `.github/workflows/falcon-ecs-fargate-deployment.yml`. You can copy this file directly to your repository.

### 2. Set Up Secrets For your GitHub Repository

1. Go to your GitHub repository
2. Click on "Settings" > "Secrets and variables" > "Actions"
3. Add the following secrets:
    - `AWS_ACCESS_KEY_ID`: The AccessKeyId from the IAM user created above
    - `AWS_SECRET_ACCESS_KEY`: The SecretAccessKey from the IAM user created above
    - `FALCON_CLIENT_ID`: Your CrowdStrike API client ID
    - `FALCON_CLIENT_SECRET`: Your CrowdStrike API client secret
    - `FALCON_CID`: Your CrowdStrike customer ID with checksum

### 3. Customize the Workflow Parameters

Open the workflow file and customize the default values for the workflow inputs to match your environment:

- `aws_region`: Your AWS region
- `ecs_cluster`: Your ECS cluster name
- `sensor_repo`: ECR repository for the Falcon sensor
- `sensor_version`: Falcon sensor version to use (latest, n-1, n-2, or specific version)
- `existing_task_definition`: Your existing task definition name
- `task_family`: New secure task family name

---

## Using the Workflow

### 1. Trigger the Workflow

1. Go to your GitHub repository
2. Click on the "Actions" tab
3. Select the "Secure ECS Application with Falcon Container Sensor" workflow
4. Click "Run workflow"
5. Enter the parameters (or use the defaults you configured):
   - `aws_region`: Your AWS region
   - `ecs_cluster`: Your ECS cluster name
   - `sensor_repo`: ECR repository for the Falcon sensor
   - `sensor_version`: Falcon sensor version to use (latest, n-1, n-2, or specific version)
   - `existing_task_definition`: Your existing task definition name
   - `task_family`: New secure task family name
6. Click "Run workflow" to start the process

### 2. Monitor the Workflow Execution

1. Watch each step execute in the GitHub Actions interface
2. Review logs for any errors or warnings
3. The workflow will output the Task ARN and Task ID when complete

### 3. Verify Deployment with AWS CLI

```bash
# Find all tasks that use the secure task definition (with Falcon Container Sensor)
SECURE_TASKS=($(aws ecs list-tasks --cluster YOUR_ECS_CLUSTER --family YOUR_SECURE_TASK_FAMILY --query 'taskArns[]' --output text))

# Process each secure task
for SECURE_TASK in "${SECURE_TASKS[@]}"; do
    echo "Processing task: $SECURE_TASK"

    # Get the task details
    aws ecs describe-tasks --cluster YOUR_ECS_CLUSTER --tasks $SECURE_TASK

    # Extract the task ID for Falcon console verification
    TASK_ID=$(echo $SECURE_TASK | awk -F '/' '{print $3}')
    echo "Task ID for Falcon console verification: $TASK_ID"
    echo "----------------------------------------"
done
```

### 4. Verify in Falcon Console

1. Log in to the Falcon Console
2. Go to Host Management
3. Add a Pod ID filter with the Task ID value from above
4. Verify the Host ID field has a value
5. Confirm the sensor is connected and reporting properly

---

## Troubleshooting

### Common Issues and Solutions

1. **Authentication Failures**:
   - Verify AWS credentials have appropriate permissions
   - Ensure CrowdStrike API credentials are correct

2. **Image Patching Failures**:
   - Check Docker socket permissions
   - Verify ECR repositories exist and are accessible

3. **Task Definition Registration Failures**:
   - Validate JSON syntax in the task definition
   - Ensure IAM roles exist and have proper permissions

4. **Task Deployment Failures**:
   - Check subnet and security group configurations
   - Verify ECS cluster exists and is properly configured

5. **Sensor Not Reporting**:
   - Verify CID is correct with checksum
   - Check container logs for sensor startup issues

### Permission-Related Issues

1. **"User is not authorized to perform iam:PassRole"**:
   - Add the specific IAM PassRole permission for the ecsTaskExecutionRole
   - Verify the IAM user has the correct policy attached

2. **"Repository does not exist"**:
   - Ensure the ECR repositories are created before running the workflow
   - Check repository naming and region settings

3. **"Access Denied" when pushing to ECR**:
   - Verify the IAM user has ECR push permissions
   - Check that the ECR login was successful

---

## Reference Materials

- [Falcon Container Sensor Documentation](https://falcon.crowdstrike.com/documentation/146/falcon-container-sensor-for-linux)
- [AWS ECS Fargate Documentation](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/AWS_Fargate.html)
- [AWS IAM Best Practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html)
- [GitHub Actions Documentation](https://docs.github.com/en/actions)
- [GitHub Actions Security Hardening](https://docs.github.com/en/actions/security-guides/security-hardening-for-github-actions)