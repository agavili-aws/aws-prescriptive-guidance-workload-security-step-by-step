# WKLD.06 - Use Systems Manager Instead of SSH or RDP

## 🎯 What This Control Does

This control replaces traditional SSH and RDP access with AWS Systems Manager Session Manager, providing secure shell access to EC2 instances without requiring public IP addresses, bastion hosts, or managing SSH keys.

## 🔍 Why This Matters

- **No public access required**: Access instances in private subnets without internet gateways
- **No SSH key management**: Eliminates the need to distribute and rotate SSH keys
- **Centralized access control**: Use IAM policies to control who can access which instances
- **Complete audit trail**: All session activity is logged in CloudTrail
- **Enhanced security**: No open SSH ports or attack surface

## ⏱️ Time Required
**25 minutes**

## 🔧 How Session Manager Works

### Traditional SSH/RDP
- Requires public IP or bastion host
- SSH keys or passwords to manage
- Network security groups with port 22/3389 open
- Limited audit capabilities

### Session Manager
- Uses AWS APIs for connectivity
- IAM-based authentication and authorization
- No inbound ports required
- Complete session logging and recording

---

## 📋 Step-by-Step Instructions

### Step 1: Verify SSM Agent Installation

#### Check Agent Status
```bash
# On Amazon Linux 2, Ubuntu 16.04+, RHEL 7+
sudo systemctl status amazon-ssm-agent

# On Amazon Linux 1
sudo status amazon-ssm-agent

# On Windows (PowerShell as Administrator)
Get-Service AmazonSSMAgent
```

#### Install SSM Agent (if needed)
```bash
# Amazon Linux 2
sudo yum install -y amazon-ssm-agent
sudo systemctl enable amazon-ssm-agent
sudo systemctl start amazon-ssm-agent

# Ubuntu
sudo snap install amazon-ssm-agent --classic
sudo systemctl enable snap.amazon-ssm-agent.amazon-ssm-agent.service
sudo systemctl start snap.amazon-ssm-agent.amazon-ssm-agent.service

# Windows (PowerShell as Administrator)
Invoke-WebRequest https://s3.amazonaws.com/ec2-downloads-windows/SSMAgent/latest/windows_amd64/AmazonSSMAgentSetup.exe -OutFile AmazonSSMAgentSetup.exe
.\AmazonSSMAgentSetup.exe /S
```

### Step 2: Create IAM Role for EC2 Instances

#### Create the Role
1. **Navigate to IAM Console**:
   - Go to [IAM console](https://console.aws.amazon.com/iam)
   - Click **Roles** → **Create role**

2. **Configure role**:
   ```
   Trusted entity type: AWS service
   Use case: EC2
   ```

3. **Attach policies**:
   ```
   Required: AmazonSSMManagedInstanceCore
   Optional: CloudWatchAgentServerPolicy (for enhanced monitoring)
   Optional: AmazonS3ReadOnlyAccess (if needed for your applications)
   ```

4. **Name the role**:
   ```
   Role name: EC2-SSM-Role
   Description: Role for EC2 instances to use Systems Manager
   ```

### Step 3: Attach Role to EC2 Instances

#### For New Instances
1. **During EC2 launch**:
   - In the launch wizard under **Advanced details**
   - **IAM instance profile**: Select `EC2-SSM-Role`

#### For Existing Instances
1. **Modify existing instance**:
   - Go to EC2 console
   - Select your instance
   - **Actions** → **Security** → **Modify IAM role**
   - Select `EC2-SSM-Role`
   - Click **Update IAM role**

### Step 4: Configure Network Connectivity

#### Option A: Internet Gateway (Public Subnets)
If your instances are in public subnets, they need internet access to reach Systems Manager endpoints.

#### Option B: NAT Gateway (Private Subnets)
```bash
# Instances in private subnets need NAT Gateway for internet access
# Or use VPC endpoints (more secure, see Option C)
```

#### Option C: VPC Endpoints (Recommended for Private Subnets)
1. **Create VPC endpoints**:
   - Go to [VPC console](https://console.aws.amazon.com/vpc)
   - Click **Endpoints** → **Create endpoint**

2. **Create required endpoints**:
   ```
   Service names (replace region as needed):
   - com.amazonaws.region.ssm
   - com.amazonaws.region.ssmmessages  
   - com.amazonaws.region.ec2messages
   ```

3. **Configure each endpoint**:
   ```
   Service category: AWS services
   Service name: [select from list above]
   VPC: [your VPC]
   Route tables: [select your private route tables]
   Policy: Full access (or custom policy)
   ```

### Step 5: Create IAM Policy for Users

#### Create Session Manager Access Policy
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:StartSession"
            ],
            "Resource": [
                "arn:aws:ec2:*:*:instance/*"
            ],
            "Condition": {
                "StringLike": {
                    "ssm:resourceTag/Environment": [
                        "development",
                        "staging"
                    ]
                }
            }
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:DescribeInstanceInformation",
                "ssm:DescribeInstanceProperties",
                "ssm:DescribeSessions",
                "ssm:GetConnectionStatus"
            ],
            "Resource": "*"
        },
        {
            "Effect": "Allow",
            "Action": [
                "ssm:TerminateSession"
            ],
            "Resource": [
                "arn:aws:ssm:*:*:session/${aws:username}-*"
            ]
        }
    ]
}
```

#### Attach Policy to Users/Groups
1. **Create or select IAM group**
2. **Attach the policy** to the group
3. **Add users** to the group

### Step 6: Start Session Manager Sessions

#### Using AWS Console
1. **Navigate to Systems Manager**:
   - Go to [Systems Manager console](https://console.aws.amazon.com/systems-manager)
   - Click **Session Manager** in left navigation

2. **Start session**:
   - Click **Start session**
   - Select your instance from the list
   - Click **Start session**

#### Using AWS CLI
```bash
# Install Session Manager plugin
# macOS
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/mac/sessionmanager-bundle.zip" -o "sessionmanager-bundle.zip"
unzip sessionmanager-bundle.zip
sudo ./sessionmanager-bundle/install -i /usr/local/sessionmanagerplugin -b /usr/local/bin/session-manager-plugin

# Linux
curl "https://s3.amazonaws.com/session-manager-downloads/plugin/latest/linux_64bit/session-manager-plugin.rpm" -o "session-manager-plugin.rpm"
sudo yum install -y session-manager-plugin.rpm

# Start session
aws ssm start-session --target i-1234567890abcdef0

# Start session with specific user
aws ssm start-session --target i-1234567890abcdef0 --parameters 'command=["sudo su - ec2-user"]'
```

## ✅ Verification Steps

### Test Instance Connectivity
1. **Check instance appears in Session Manager**:
   - Go to Session Manager console
   - Verify your instance appears in the target list
   - Status should show "Success"

2. **Start a test session**:
   - Click on your instance
   - Start session
   - Verify you get a shell prompt

3. **Test commands**:
   ```bash
   # Basic connectivity test
   whoami
   pwd
   ls -la
   
   # Test AWS CLI (should work with instance role)
   aws sts get-caller-identity
   ```

### Test Access Controls
1. **Test with different users**:
   - Users with policy should be able to connect
   - Users without policy should be denied

2. **Test tag-based access**:
   - If using tag conditions, test with tagged/untagged instances

## 💡 Best Practices

### Security Configuration
- **Use VPC endpoints**: Avoid internet traffic for private instances
- **Tag-based access**: Use EC2 tags to control access to specific instances
- **Least privilege**: Grant minimum necessary permissions
- **Session recording**: Enable session logging for audit purposes

### Operational Practices
- **Remove SSH access**: Close port 22 in security groups once Session Manager is working
- **Regular access review**: Audit who has Session Manager access
- **Emergency access**: Maintain emergency access procedures
- **Training**: Train team on Session Manager usage

### Monitoring and Logging
- **CloudTrail**: Monitor session start/stop events
- **Session logs**: Enable session logging to S3 or CloudWatch
- **Access patterns**: Monitor unusual access patterns
- **Failed attempts**: Alert on failed session attempts

## 🚨 Common Mistakes to Avoid

- ❌ Not attaching the IAM role to EC2 instances
- ❌ Missing VPC endpoints for private subnet instances
- ❌ Overly permissive IAM policies
- ❌ Not testing connectivity before removing SSH access
- ❌ Forgetting to install/update SSM agent
- ❌ Not configuring session logging

## 🔧 Advanced Configuration

### Session Preferences
```json
{
    "schemaVersion": "1.0",
    "description": "Session Manager preferences",
    "sessionType": "Standard_Stream",
    "inputs": {
        "s3BucketName": "my-session-logs-bucket",
        "s3KeyPrefix": "session-logs/",
        "s3EncryptionEnabled": true,
        "cloudWatchLogGroupName": "/aws/sessionmanager/sessions",
        "cloudWatchEncryptionEnabled": true,
        "idleSessionTimeout": "20",
        "maxSessionDuration": "60",
        "runAsEnabled": false,
        "runAsDefaultUser": "ec2-user",
        "shellProfile": {
            "windows": "date",
            "linux": "pwd; ls -la"
        }
    }
}
```

### Port Forwarding
```bash
# Forward local port to remote port
aws ssm start-session \
    --target i-1234567890abcdef0 \
    --document-name AWS-StartPortForwardingSession \
    --parameters '{"portNumber":["80"],"localPortNumber":["8080"]}'

# Access application at localhost:8080
```

### Run Commands on Multiple Instances
```bash
# Run command on multiple instances
aws ssm send-command \
    --document-name "AWS-RunShellScript" \
    --parameters 'commands=["sudo yum update -y"]' \
    --targets "Key=tag:Environment,Values=production"
```

## 📊 Session Logging Configuration

### CloudWatch Logs
```bash
# Create log group
aws logs create-log-group --log-group-name /aws/sessionmanager/sessions

# Configure Session Manager to use CloudWatch
aws ssm put-document \
    --name "SSM-SessionManagerRunShell" \
    --document-type "Session" \
    --document-format JSON \
    --content file://session-preferences.json
```

### S3 Logging
```json
{
    "schemaVersion": "1.0",
    "description": "Session Manager configuration with S3 logging",
    "sessionType": "Standard_Stream",
    "inputs": {
        "s3BucketName": "my-session-logs-bucket",
        "s3KeyPrefix": "session-logs/",
        "s3EncryptionEnabled": true
    }
}
```

## 🔗 Official AWS Documentation

- [Setting up Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-getting-started.html)
- [Working with Session Manager](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-working-with.html)
- [Session Manager prerequisites](https://docs.aws.amazon.com/systems-manager/latest/userguide/session-manager-prerequisites.html)
- [Installing SSM Agent](https://docs.aws.amazon.com/systems-manager/latest/userguide/sysman-install-ssm-agent.html)

## ➡️ What's Next?

Excellent! You've eliminated the need for SSH/RDP and established secure, auditable access to your instances. Now let's enhance your S3 security with detailed logging:

**Next Control**: [WKLD.07 - Log Data Events for S3 Buckets with Sensitive Data](./wkld-07.md)

---

[← Previous: WKLD.05](./wkld-05.md) | [Back to Main Guide](./README.md) | [Next: WKLD.07 →](./wkld-07.md)
