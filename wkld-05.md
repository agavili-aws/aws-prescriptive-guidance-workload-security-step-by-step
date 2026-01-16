# WKLD.05 - Detect and Remediate Exposed Secrets

## 🎯 What This Control Does

This control implements monitoring and detection systems to identify if secrets have been exposed despite prevention measures, and provides automated remediation capabilities.

## 🔍 Why This Matters

- **Early detection**: Catch exposed secrets before they're exploited
- **Automated response**: Quickly remediate exposed credentials
- **Compliance**: Meet requirements for incident detection and response
- **Damage limitation**: Minimize impact of credential exposure

## ⏱️ Time Required
**20 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Set Up Amazon CodeGuru Reviewer

#### Enable CodeGuru Reviewer
1. **Navigate to CodeGuru Console**:
   - Go to [CodeGuru console](https://console.aws.amazon.com/codeguru)
   - Click **Reviewer** → **Associate repository**

2. **Configure repository scanning**:
   ```
   Repository source: GitHub/Bitbucket/CodeCommit
   Repository: Select your repository
   ✅ Enable secrets detection
   ```

### Step 2: Configure Automated Remediation

#### Set up Lambda for secret rotation
```python
import boto3
import json

def lambda_handler(event, context):
    """Automatically rotate exposed secrets"""
    
    # Parse CodeGuru finding
    finding = json.loads(event['Records'][0]['Sns']['Message'])
    
    if 'hardcoded-credentials' in finding['type']:
        # Rotate the exposed secret
        secrets_client = boto3.client('secretsmanager')
        
        try:
            # Trigger secret rotation
            response = secrets_client.rotate_secret(
                SecretId=finding['secret_arn'],
                ForceRotateSecrets=True
            )
            
            # Notify security team
            sns_client = boto3.client('sns')
            sns_client.publish(
                TopicArn='arn:aws:sns:region:account:security-alerts',
                Message=f"Secret rotated due to exposure: {finding['secret_arn']}",
                Subject="Automated Secret Rotation"
            )
            
        except Exception as e:
            print(f"Failed to rotate secret: {e}")
    
    return {'statusCode': 200}
```

## ✅ Verification Steps

### Test Detection
1. **Create test repository** with intentional secret
2. **Verify CodeGuru** detects the secret
3. **Check remediation** triggers automatically

## 💡 Best Practices

- **Regular scanning** of all repositories
- **Automated rotation** for detected secrets
- **Team notification** of exposures
- **Post-incident analysis** to prevent recurrence

## 🚨 Common Mistakes to Avoid

- ❌ Not scanning existing code repositories
- ❌ Manual remediation processes (too slow)
- ❌ Not notifying affected teams
- ❌ Ignoring low-severity findings

## 🔗 Official AWS Documentation

- [CodeGuru Reviewer secrets detection](https://aws.amazon.com/blogs/aws/codeguru-reviewer-secrets-detector-identify-hardcoded-secrets/)

## ➡️ What's Next?

**Next Control**: [WKLD.06 - Use Systems Manager Instead of SSH or RDP](./wkld-06.md)

---

[← Previous: WKLD.04](./wkld-04.md) | [Back to Main Guide](./README.md) | [Next: WKLD.06 →](./wkld-06.md)
