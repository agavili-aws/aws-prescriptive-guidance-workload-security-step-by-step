# WKLD.01 - Use IAM Roles for Compute Environment Permissions

## 🎯 What This Control Does

This control establishes the use of IAM roles instead of long-term access keys for AWS compute services like EC2, Lambda, and ECS. IAM roles provide temporary, automatically rotating credentials that eliminate the security risks of managing static access keys.

## 🔍 Why This Matters

- **No long-term credentials**: Roles provide temporary credentials that automatically rotate
- **Reduced attack surface**: No access keys to steal, lose, or accidentally expose
- **Simplified management**: No need to distribute, rotate, or revoke access keys
- **Automatic authentication**: AWS SDKs automatically use role credentials
- **Audit trail**: Better tracking of which service performed which actions

## ⏱️ Time Required
**15 minutes**

## 🔑 Understanding IAM Roles vs Access Keys

### IAM Roles (Recommended)
- ✅ Temporary credentials (15 minutes to 12 hours)
- ✅ Automatically rotated by AWS
- ✅ No credentials to manage or store
- ✅ Assumed by AWS services automatically
- ✅ Can be easily audited and monitored

### Access Keys (Avoid for Compute)
- ❌ Long-term credentials (never expire unless rotated)
- ❌ Must be manually rotated
- ❌ Can be accidentally exposed in code/logs
- ❌ Difficult to track usage
- ❌ Security risk if compromised

---

## 📋 Step-by-Step Instructions

### Step 1: Create IAM Role for EC2 Instances

1. **Navigate to IAM Console**:
   - Go to [IAM console](https://console.aws.amazon.com/iam)
   - Click **Roles** in the left navigation
   - Click **Create role**

2. **Configure role basics**:
   ```
   Trusted entity type: AWS service
   Use case: EC2
   ```
   - Click **Next**

3. **Attach permissions policies**:
   - Search for and select appropriate policies based on your needs:
     ```
     Common examples:
     - AmazonS3ReadOnlyAccess (for S3 read access)
     - AmazonSSMManagedInstanceCore (for Systems Manager)
     - CloudWatchAgentServerPolicy (for CloudWatch monitoring)
     ```
   - Click **Next**

4. **Name and create role**:
   ```
   Role name: EC2-Application-Role
   Description: Role for EC2 instances to access AWS services
   ```
   - Review permissions
   - Click **Create role**

### Step 2: Create Instance Profile (Automatic)

When you create a role for EC2, AWS automatically creates an instance profile with the same name. This is what gets attached to EC2 instances.

### Step 3: Attach Role to EC2 Instance

#### For New Instances:
1. **During EC2 launch**:
   - In the EC2 launch wizard
   - Under **Advanced details**
   - **IAM instance profile**: Select your role (EC2-Application-Role)

#### For Existing Instances:
1. **Attach to running instance**:
   - Go to EC2 console
   - Select your instance
   - **Actions** → **Security** → **Modify IAM role**
   - Select your role
   - Click **Update IAM role**

### Step 4: Create Role for Lambda Functions

1. **Create Lambda execution role**:
   ```
   Trusted entity type: AWS service
   Use case: Lambda
   ```

2. **Attach basic Lambda policy**:
   ```
   Required: AWSLambdaBasicExecutionRole
   Additional: Based on your function's needs
   - AmazonS3FullAccess (if accessing S3)
   - AmazonDynamoDBFullAccess (if accessing DynamoDB)
   ```

3. **Name the role**:
   ```
   Role name: Lambda-Execution-Role
   Description: Execution role for Lambda functions
   ```

### Step 5: Create Role for ECS Tasks

1. **Create ECS task role**:
   ```
   Trusted entity type: AWS service
   Use case: Elastic Container Service → Elastic Container Service Task
   ```

2. **Attach necessary policies**:
   ```
   Based on your container's needs:
   - AmazonS3ReadOnlyAccess
   - AmazonDynamoDBReadOnlyAccess
   - Custom policies for specific resources
   ```

3. **Name the role**:
   ```
   Role name: ECS-Task-Role
   Description: Role for ECS tasks to access AWS services
   ```

## ✅ Verification Steps

### Test EC2 Role
1. **SSH into your EC2 instance** (or use Systems Manager Session Manager)
2. **Test AWS CLI without credentials**:
   ```bash
   # This should work without configuring credentials
   aws sts get-caller-identity
   
   # Test specific service access
   aws s3 ls  # (if S3 permissions were granted)
   ```

### Test Lambda Role
1. **Create a test Lambda function**:
   ```python
   import boto3
   import json
   
   def lambda_handler(event, context):
       # This should work automatically with the role
       sts = boto3.client('sts')
       identity = sts.get_caller_identity()
       
       return {
           'statusCode': 200,
           'body': json.dumps(f"Role ARN: {identity['Arn']}")
       }
   ```

2. **Test the function** and verify it can access AWS services

### Test ECS Role
1. **Deploy a container** with the task role
2. **Verify the container** can access AWS services without embedded credentials

## 💡 Best Practices

### Role Design
- **Principle of least privilege**: Grant only necessary permissions
- **Service-specific roles**: Create separate roles for different services/applications
- **Environment separation**: Use different roles for dev/staging/production
- **Regular review**: Audit role permissions quarterly

### Permission Management
- **Use AWS managed policies** when possible for common use cases
- **Create custom policies** for specific business requirements
- **Avoid overly broad permissions** like `*` actions on `*` resources
- **Use conditions** to further restrict access when needed

### Monitoring and Auditing
- **CloudTrail logging**: Monitor role assumption and usage
- **Access Analyzer**: Use IAM Access Analyzer to review permissions
- **Regular audits**: Review which roles are being used and by what
- **Unused role cleanup**: Remove roles that are no longer needed

## 🚨 Common Mistakes to Avoid

- ❌ Using access keys instead of roles for EC2/Lambda/ECS
- ❌ Creating overly permissive roles with admin access
- ❌ Not testing role permissions before deploying to production
- ❌ Forgetting to attach roles to new instances/functions
- ❌ Using the same role for all applications (lack of separation)
- ❌ Not monitoring role usage and permissions
- ❌ Hardcoding credentials in application code

## 🔧 Advanced Configuration

### Cross-Account Role Access
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "AWS": "arn:aws:iam::ACCOUNT-B:root"
      },
      "Action": "sts:AssumeRole",
      "Condition": {
        "StringEquals": {
          "sts:ExternalId": "unique-external-id"
        }
      }
    }
  ]
}
```

### Role with Session Duration
```bash
# Assume role with custom session duration
aws sts assume-role \
    --role-arn arn:aws:iam::123456789012:role/MyRole \
    --role-session-name MySession \
    --duration-seconds 3600
```

### Custom Policy Example
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "s3:GetObject",
        "s3:PutObject"
      ],
      "Resource": "arn:aws:s3:::my-app-bucket/*"
    },
    {
      "Effect": "Allow",
      "Action": [
        "dynamodb:GetItem",
        "dynamodb:PutItem",
        "dynamodb:UpdateItem"
      ],
      "Resource": "arn:aws:dynamodb:us-east-1:123456789012:table/MyAppTable"
    }
  ]
}
```

## 📊 Monitoring Role Usage

### CloudWatch Metrics
Monitor these metrics for role usage:
- Role assumption frequency
- Failed role assumptions
- Services using each role

### CloudTrail Events
Key events to monitor:
- `AssumeRole` - When roles are assumed
- `AssumeRoleWithWebIdentity` - For federated access
- API calls made using role credentials

### AWS Config Rules
Set up Config rules to monitor:
- Unused IAM roles
- Overly permissive roles
- Roles without MFA requirements (where applicable)

## 🔗 Official AWS Documentation

### General IAM Roles
- [IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)
- [Using IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles_use.html)

### Service-Specific Documentation
- [IAM roles for Amazon EC2](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/iam-roles-for-amazon-ec2.html)
- [Lambda execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)
- [IAM roles for tasks](https://docs.aws.amazon.com/AmazonECS/latest/developerguide/task-iam-roles.html)

## ➡️ What's Next?

Excellent! You've established secure credential management for your compute resources. Now let's add an additional layer of security with resource-based policies:

**Next Control**: [WKLD.02 - Restrict Credential Usage Scope with Resource-Based Policies](./wkld-02.md)

---

[Back to Main Guide](./README.md) | [Next: WKLD.02 →](./wkld-02.md)
