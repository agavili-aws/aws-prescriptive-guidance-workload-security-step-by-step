# WKLD.02 - Restrict Credential Usage Scope with Resource-Based Policies

## 🎯 What This Control Does

This control adds resource-based policies to AWS resources like S3 buckets and VPC endpoints to create an additional layer of access control. Even if someone has the right IAM permissions, they must also meet the conditions defined in the resource policy.

## 🔍 Why This Matters

- **Defense in depth**: Two layers of permission checks (identity + resource policies)
- **Organizational boundaries**: Restrict access to resources from specific AWS Organizations
- **Network isolation**: Limit access to specific VPCs or IP ranges
- **Compliance**: Meet regulatory requirements for data access controls
- **Incident containment**: Limit blast radius if credentials are compromised

## ⏱️ Time Required
**20 minutes**

## 🔐 Understanding Policy Types

### Identity-Based Policies
- Attached to users, groups, or roles
- Define what the principal can do
- Example: "This role can read S3 objects"

### Resource-Based Policies
- Attached to resources (S3 buckets, KMS keys, etc.)
- Define who can access the resource and under what conditions
- Example: "Only principals from our organization can access this bucket"

### How They Work Together
For access to be granted, BOTH policies must allow the action:
1. ✅ Identity policy: "Role can read S3 objects"
2. ✅ Resource policy: "Allow access from our organization"
3. ✅ **Result**: Access granted

---

## 📋 Step-by-Step Instructions

### Step 1: Restrict S3 Bucket Access by Organization

1. **Get your Organization ID**:
   - Go to [AWS Organizations console](https://console.aws.amazon.com/organizations)
   - Note your Organization ID (format: `o-xxxxxxxxxx`)
   - If you don't use Organizations, skip to Step 2

2. **Create organization-restricted bucket policy**:
   - Go to [S3 console](https://console.aws.amazon.com/s3)
   - Select your bucket
   - Go to **Permissions** tab
   - Click **Edit** under **Bucket policy**

3. **Add organization restriction policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowFromOrganizationOnly",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:*",
         "Resource": [
           "arn:aws:s3:::YOUR-BUCKET-NAME",
           "arn:aws:s3:::YOUR-BUCKET-NAME/*"
         ],
         "Condition": {
           "StringEquals": {
             "aws:PrincipalOrgID": "o-xxxxxxxxxx"
           }
         }
       }
     ]
   }
   ```

4. **Replace placeholders**:
   - `YOUR-BUCKET-NAME`: Your actual bucket name
   - `o-xxxxxxxxxx`: Your Organization ID

### Step 2: Restrict S3 Bucket Access by VPC

For buckets that should only be accessed from specific VPCs:

1. **Get your VPC ID**:
   - Go to [VPC console](https://console.aws.amazon.com/vpc)
   - Note the VPC ID where your applications run

2. **Create VPC-restricted bucket policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowFromSpecificVPCOnly",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:*",
         "Resource": [
           "arn:aws:s3:::YOUR-BUCKET-NAME",
           "arn:aws:s3:::YOUR-BUCKET-NAME/*"
         ],
         "Condition": {
           "StringEquals": {
             "aws:SourceVpc": "vpc-xxxxxxxxx"
           }
         }
       }
     ]
   }
   ```

### Step 3: Restrict Access by IP Range

For additional security, restrict access to known IP ranges:

1. **Identify your IP ranges**:
   - Office IP addresses
   - VPC CIDR blocks
   - NAT Gateway IP addresses

2. **Create IP-restricted policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Sid": "AllowFromSpecificIPsOnly",
         "Effect": "Allow",
         "Principal": "*",
         "Action": "s3:*",
         "Resource": [
           "arn:aws:s3:::YOUR-BUCKET-NAME",
           "arn:aws:s3:::YOUR-BUCKET-NAME/*"
         ],
         "Condition": {
           "IpAddress": {
             "aws:SourceIp": [
               "203.0.113.0/24",
               "198.51.100.0/24"
             ]
           }
         }
       }
     ]
   }
   ```

### Step 4: Combine Multiple Conditions

Create a comprehensive policy with multiple restrictions:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "AllowFromOrganizationAndVPC",
      "Effect": "Allow",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ],
      "Condition": {
        "StringEquals": {
          "aws:PrincipalOrgID": "o-xxxxxxxxxx"
        },
        "StringLike": {
          "aws:SourceVpc": "vpc-*"
        }
      }
    },
    {
      "Sid": "DenyInsecureConnections",
      "Effect": "Deny",
      "Principal": "*",
      "Action": "s3:*",
      "Resource": [
        "arn:aws:s3:::YOUR-BUCKET-NAME",
        "arn:aws:s3:::YOUR-BUCKET-NAME/*"
      ],
      "Condition": {
        "Bool": {
          "aws:SecureTransport": "false"
        }
      }
    }
  ]
}
```

### Step 5: Apply to VPC Endpoints

1. **Create VPC endpoint policy**:
   - Go to [VPC console](https://console.aws.amazon.com/vpc)
   - Click **Endpoints**
   - Select your S3 endpoint
   - Click **Policy** tab
   - Click **Edit policy**

2. **Add restrictive endpoint policy**:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [
       {
         "Effect": "Allow",
         "Principal": "*",
         "Action": [
           "s3:GetObject",
           "s3:PutObject"
         ],
         "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*",
         "Condition": {
           "StringEquals": {
             "aws:PrincipalArn": [
               "arn:aws:iam::ACCOUNT-ID:role/EC2-Application-Role"
             ]
           }
         }
       }
     ]
   }
   ```

## ✅ Verification Steps

### Test Organization Restriction
1. **From within organization**: Access should work normally
2. **From external account**: Access should be denied
3. **Check CloudTrail logs**: Verify policy evaluation results

### Test VPC Restriction
1. **From correct VPC**: Access should work
2. **From different VPC**: Access should be denied
3. **From internet**: Access should be denied (if no internet gateway route)

### Test IP Restriction
1. **From allowed IP**: Access should work
2. **From different IP**: Access should be denied
3. **Monitor access logs**: Review S3 access logs for denied requests

## 💡 Best Practices

### Policy Design
- **Start restrictive**: Begin with deny-all and add specific allows
- **Use multiple conditions**: Combine organization, VPC, and IP restrictions
- **Regular review**: Audit policies quarterly for relevance
- **Test thoroughly**: Verify policies work as expected before production

### Condition Keys
- **aws:PrincipalOrgID**: Restrict to your AWS Organization
- **aws:SourceVpc**: Limit to specific VPCs
- **aws:SourceIp**: Restrict by IP address ranges
- **aws:SecureTransport**: Require HTTPS/TLS connections
- **aws:RequestedRegion**: Limit to specific AWS regions

### Monitoring
- **CloudTrail**: Monitor policy evaluation results
- **S3 access logs**: Track actual access patterns
- **VPC Flow Logs**: Monitor network-level access
- **AWS Config**: Track policy changes over time

## 🚨 Common Mistakes to Avoid

- ❌ Creating policies that are too restrictive (blocking legitimate access)
- ❌ Not testing policies before applying to production resources
- ❌ Forgetting to update policies when infrastructure changes
- ❌ Using overly broad conditions that don't provide real security
- ❌ Not monitoring policy effectiveness over time
- ❌ Mixing up Allow and Deny effects in complex policies

## 🔧 Advanced Policy Examples

### Time-Based Access
```json
{
  "Condition": {
    "DateGreaterThan": {
      "aws:CurrentTime": "2024-01-01T00:00:00Z"
    },
    "DateLessThan": {
      "aws:CurrentTime": "2024-12-31T23:59:59Z"
    }
  }
}
```

### MFA Required
```json
{
  "Condition": {
    "Bool": {
      "aws:MultiFactorAuthPresent": "true"
    },
    "NumericLessThan": {
      "aws:MultiFactorAuthAge": "3600"
    }
  }
}
```

### Specific User Agent
```json
{
  "Condition": {
    "StringLike": {
      "aws:RequestTag/Application": "MyApp*"
    }
  }
}
```

## 📊 Policy Testing Tools

### AWS Policy Simulator
- Test policies before deployment
- Simulate different scenarios
- Available at: https://policysim.aws.amazon.com/

### AWS CLI Testing
```bash
# Test S3 access with specific conditions
aws s3 ls s3://your-bucket-name --debug

# Check policy evaluation
aws iam simulate-principal-policy \
    --policy-source-arn arn:aws:iam::123456789012:role/MyRole \
    --action-names s3:GetObject \
    --resource-arns arn:aws:s3:::your-bucket-name/test-object
```

## 🔗 Official AWS Documentation

- [Identity-based policies and resource-based policies](https://docs.aws.amazon.com/IAM/latest/UserGuide/access_policies_identity-vs-resource.html)
- [AWS global condition context keys](https://docs.aws.amazon.com/IAM/latest/UserGuide/reference_policies_condition-keys.html)
- [S3 bucket policy examples](https://docs.aws.amazon.com/AmazonS3/latest/userguide/bucket-policy-examples.html)
- [VPC endpoint policies](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints-access.html)

## ➡️ What's Next?

Great work! You've added an important layer of access control to your resources. Now let's secure how your applications handle secrets and credentials:

**Next Control**: [WKLD.03 - Use Ephemeral Secrets or a Secrets-Management Service](./wkld-03.md)

---

[← Previous: WKLD.01](./wkld-01.md) | [Back to Main Guide](./README.md) | [Next: WKLD.03 →](./wkld-03.md)
