# WKLD.08 - Encrypt Amazon EBS Volumes

## 🎯 What This Control Does

This control enables encryption by default for all Amazon EBS volumes in your AWS account, ensuring that all data stored on EBS volumes is encrypted at rest using AWS KMS keys.

## 🔍 Why This Matters

- **Data protection**: Encrypts data at rest to protect against unauthorized access
- **Compliance**: Meets regulatory requirements for data encryption
- **No performance impact**: Minimal latency impact with same IOPS performance
- **Automatic**: Once enabled, all new volumes are encrypted automatically
- **Key management**: Centralized encryption key management through AWS KMS

## ⏱️ Time Required
**10 minutes**

## 🔐 Understanding EBS Encryption

### What Gets Encrypted
- ✅ Data at rest inside the volume
- ✅ Data in transit between instance and volume
- ✅ All snapshots created from encrypted volumes
- ✅ All volumes created from encrypted snapshots

### Encryption Keys
- **AWS Managed Key**: `aws/ebs` (default, no additional cost)
- **Customer Managed Key**: Your own KMS key (additional KMS costs)
- **Per-region**: Encryption settings are region-specific

---

## 📋 Step-by-Step Instructions

### Step 1: Enable EBS Encryption by Default

#### Using AWS Console
1. **Navigate to EC2 Console**:
   - Go to [EC2 console](https://console.aws.amazon.com/ec2)
   - Select your region from the top-right dropdown

2. **Access EBS settings**:
   - In the left navigation, scroll down to **Elastic Block Store**
   - Click **Account attributes**

3. **Enable encryption**:
   - Find **EBS encryption**
   - Click **Manage**
   - Check **Enable** for "Always encrypt new EBS volumes"
   - Select encryption key:
     ```
     Default: aws/ebs (AWS managed key)
     Custom: Select your customer managed key
     ```
   - Click **Update EBS encryption**

#### Using AWS CLI
```bash
# Enable EBS encryption by default with AWS managed key
aws ec2 enable-ebs-encryption-by-default

# Enable with custom KMS key
aws ec2 modify-ebs-default-kms-key-id --kms-key-id arn:aws:kms:us-east-1:123456789012:key/12345678-1234-1234-1234-123456789012

# Verify settings
aws ec2 describe-ebs-encryption-by-default
aws ec2 describe-ebs-default-kms-key-id
```

### Step 2: Repeat for All Regions

EBS encryption settings are region-specific, so repeat for each region you use:

```bash
# List all regions
aws ec2 describe-regions --query 'Regions[].RegionName' --output text

# Enable encryption in each region
for region in us-east-1 us-west-2 eu-west-1; do
    echo "Enabling EBS encryption in $region"
    aws ec2 enable-ebs-encryption-by-default --region $region
done
```

### Step 3: Create Customer Managed KMS Key (Optional)

#### Create KMS Key for EBS
1. **Navigate to KMS Console**:
   - Go to [KMS console](https://console.aws.amazon.com/kms)
   - Click **Create key**

2. **Configure key**:
   ```
   Key type: Symmetric
   Key usage: Encrypt and decrypt
   Key spec: SYMMETRIC_DEFAULT
   ```

3. **Set key policy**:
   ```
   Alias: ebs-encryption-key
   Description: Customer managed key for EBS encryption
   ```

4. **Key policy example**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "Enable IAM User Permissions",
         "Effect": "Allow",
         "Principal": {
           "AWS": "arn:aws:iam::123456789012:root"
         },
         "Action": "kms:*",
         "Resource": "*"
       },
       {
         "Sid": "Allow EBS service",
         "Effect": "Allow",
         "Principal": {
           "Service": "ec2.amazonaws.com"
         },
         "Action": [
           "kms:Decrypt",
           "kms:DescribeKey",
           "kms:Encrypt",
           "kms:GenerateDataKey*",
           "kms:ReEncrypt*"
         ],
         "Resource": "*"
       }
     ]
   }
   ```

#### Set as Default EBS Key
```bash
# Set customer managed key as default
aws ec2 modify-ebs-default-kms-key-id --kms-key-id alias/ebs-encryption-key
```

### Step 4: Encrypt Existing Unencrypted Volumes

#### Identify Unencrypted Volumes
```bash
# List all unencrypted volumes
aws ec2 describe-volumes \
    --filters "Name=encrypted,Values=false" \
    --query 'Volumes[].{VolumeId:VolumeId,Size:Size,State:State,InstanceId:Attachments[0].InstanceId}' \
    --output table
```

#### Encrypt Existing Volume (Requires Downtime)
```bash
# 1. Stop the instance (if volume is attached)
aws ec2 stop-instances --instance-ids i-1234567890abcdef0

# 2. Create snapshot of unencrypted volume
aws ec2 create-snapshot \
    --volume-id vol-1234567890abcdef0 \
    --description "Snapshot before encryption"

# 3. Copy snapshot with encryption
aws ec2 copy-snapshot \
    --source-region us-east-1 \
    --source-snapshot-id snap-1234567890abcdef0 \
    --description "Encrypted snapshot" \
    --encrypted

# 4. Create encrypted volume from encrypted snapshot
aws ec2 create-volume \
    --snapshot-id snap-encrypted123456789 \
    --availability-zone us-east-1a \
    --encrypted

# 5. Detach old volume and attach new encrypted volume
aws ec2 detach-volume --volume-id vol-1234567890abcdef0
aws ec2 attach-volume \
    --volume-id vol-encrypted123456789 \
    --instance-id i-1234567890abcdef0 \
    --device /dev/sdf

# 6. Start instance
aws ec2 start-instances --instance-ids i-1234567890abcdef0
```

## ✅ Verification Steps

### Verify Default Encryption
```bash
# Check if encryption by default is enabled
aws ec2 describe-ebs-encryption-by-default

# Check default KMS key
aws ec2 describe-ebs-default-kms-key-id
```

### Test New Volume Creation
```bash
# Create a new volume (should be encrypted automatically)
aws ec2 create-volume \
    --size 10 \
    --availability-zone us-east-1a

# Verify the volume is encrypted
aws ec2 describe-volumes \
    --volume-ids vol-newvolume123456789 \
    --query 'Volumes[0].Encrypted'
```

### Launch New Instance
1. **Launch EC2 instance** through console or CLI
2. **Check EBS volumes** are encrypted by default
3. **Verify in EC2 console** that volumes show "Encrypted: Yes"

## 💡 Best Practices

### Key Management
- **Use AWS managed keys** for simplicity and cost-effectiveness
- **Customer managed keys** for additional control and compliance requirements
- **Key rotation**: Enable automatic key rotation for customer managed keys
- **Cross-region**: Create keys in each region where you operate

### Operational Practices
- **Enable in all regions**: Ensure encryption is enabled everywhere you operate
- **Snapshot encryption**: Verify snapshots are also encrypted
- **AMI encryption**: Use encrypted AMIs for consistent encryption
- **Documentation**: Document your encryption strategy and key usage

### Cost Considerations
- **AWS managed keys**: No additional KMS costs
- **Customer managed keys**: $1/month per key + usage costs
- **Performance**: No significant performance impact
- **Storage costs**: Same as unencrypted volumes

## 🚨 Common Mistakes to Avoid

- ❌ Only enabling encryption in one region
- ❌ Not encrypting existing volumes (security gap)
- ❌ Using customer managed keys unnecessarily (added cost/complexity)
- ❌ Not testing volume encryption before production deployment
- ❌ Forgetting to encrypt snapshots and AMIs
- ❌ Not documenting encryption key usage

## 🔧 Advanced Configuration

### Automated Encryption Script
```bash
#!/bin/bash
# encrypt-all-regions.sh

REGIONS=$(aws ec2 describe-regions --query 'Regions[].RegionName' --output text)

for region in $REGIONS; do
    echo "Processing region: $region"
    
    # Enable EBS encryption by default
    aws ec2 enable-ebs-encryption-by-default --region $region
    
    # Verify encryption is enabled
    ENCRYPTION_STATUS=$(aws ec2 describe-ebs-encryption-by-default --region $region --query 'EbsEncryptionByDefault' --output text)
    
    if [ "$ENCRYPTION_STATUS" = "True" ]; then
        echo "✅ EBS encryption enabled in $region"
    else
        echo "❌ Failed to enable EBS encryption in $region"
    fi
done
```

### CloudFormation Template
```yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Enable EBS encryption by default'

Resources:
  EBSEncryptionKey:
    Type: AWS::KMS::Key
    Properties:
      Description: 'Customer managed key for EBS encryption'
      KeyPolicy:
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'
          - Sid: Allow EBS service
            Effect: Allow
            Principal:
              Service: ec2.amazonaws.com
            Action:
              - kms:Decrypt
              - kms:DescribeKey
              - kms:Encrypt
              - kms:GenerateDataKey*
              - kms:ReEncrypt*
            Resource: '*'

  EBSEncryptionKeyAlias:
    Type: AWS::KMS::Alias
    Properties:
      AliasName: alias/ebs-encryption-key
      TargetKeyId: !Ref EBSEncryptionKey

Outputs:
  KMSKeyId:
    Description: 'KMS Key ID for EBS encryption'
    Value: !Ref EBSEncryptionKey
    Export:
      Name: !Sub '${AWS::StackName}-EBSEncryptionKey'
```

### Terraform Configuration
```hcl
# Enable EBS encryption by default
resource "aws_ebs_encryption_by_default" "example" {
  enabled = true
}

# Create customer managed KMS key
resource "aws_kms_key" "ebs" {
  description             = "Customer managed key for EBS encryption"
  deletion_window_in_days = 7
  enable_key_rotation     = true

  policy = jsonencode({
    Statement = [
      {
        Sid    = "Enable IAM User Permissions"
        Effect = "Allow"
        Principal = {
          AWS = "arn:aws:iam::${data.aws_caller_identity.current.account_id}:root"
        }
        Action   = "kms:*"
        Resource = "*"
      },
      {
        Sid    = "Allow EBS service"
        Effect = "Allow"
        Principal = {
          Service = "ec2.amazonaws.com"
        }
        Action = [
          "kms:Decrypt",
          "kms:DescribeKey",
          "kms:Encrypt",
          "kms:GenerateDataKey*",
          "kms:ReEncrypt*"
        ]
        Resource = "*"
      }
    ]
    Version = "2012-10-17"
  })
}

# Set as default EBS encryption key
resource "aws_ebs_default_kms_key" "example" {
  key_id = aws_kms_key.ebs.arn
}
```

## 📊 Monitoring EBS Encryption

### CloudWatch Metrics
Monitor these metrics:
- Volume creation events
- Encryption key usage
- Failed encryption attempts

### AWS Config Rules
```json
{
  "ConfigRuleName": "encrypted-volumes",
  "Source": {
    "Owner": "AWS",
    "SourceIdentifier": "ENCRYPTED_VOLUMES"
  }
}
```

### CloudTrail Events
Key events to monitor:
- `CreateVolume` - Check if volumes are encrypted
- `EnableEbsEncryptionByDefault` - Track encryption setting changes
- `ModifyEbsDefaultKmsKeyId` - Monitor key changes

## 🔗 Official AWS Documentation

- [Amazon EBS encryption](https://docs.aws.amazon.com/ebs/latest/userguide/ebs-encryption.html)
- [Encryption by default](https://docs.aws.amazon.com/ebs/latest/userguide/encryption-by-default.html)
- [EBS encryption best practices](https://aws.amazon.com/blogs/compute/must-know-best-practices-for-amazon-ebs-encryption/)

## ➡️ What's Next?

Perfect! You've secured your EBS volumes with encryption. Now let's encrypt your database storage:

**Next Control**: [WKLD.09 - Encrypt Amazon RDS Databases](./wkld-09.md)

---

[← Previous: WKLD.07](./wkld-07.md) | [Back to Main Guide](./README.md) | [Next: WKLD.09 →](./wkld-09.md)
