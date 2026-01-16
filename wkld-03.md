# WKLD.03 - Use Ephemeral Secrets or a Secrets-Management Service

## 🎯 What This Control Does

This control establishes secure management of application secrets (passwords, API keys, certificates) using AWS-managed services instead of hardcoding them in configuration files or environment variables. This prevents accidental exposure and enables secure secret rotation.

## 🔍 Why This Matters

- **Prevents secret exposure**: No hardcoded secrets in code or configuration files
- **Automatic rotation**: Secrets can be rotated automatically without code changes
- **Centralized management**: All secrets managed in one secure location
- **Audit trail**: Complete logging of secret access and changes
- **Environment promotion**: Easy to promote code between dev/staging/production

## ⏱️ Time Required
**25 minutes**

## 🔐 AWS Secrets Management Options

### AWS Systems Manager Parameter Store
**Best for**: Simple key-value secrets, frequently accessed parameters
- ✅ Free for standard parameters
- ✅ Integrated with other AWS services
- ✅ Supports hierarchical organization
- ✅ Built-in encryption with KMS

### AWS Secrets Manager
**Best for**: Complex secrets, automatic rotation, database credentials
- ✅ Automatic rotation capabilities
- ✅ Cross-region replication
- ✅ Integration with RDS, DocumentDB, Redshift
- ✅ Supports JSON documents

---

## 📋 Step-by-Step Instructions

### Step 1: Set Up Parameter Store for Simple Secrets

#### Create a KMS Key for Encryption
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
   Alias: alias/parameter-store-key
   Description: Key for Parameter Store secrets encryption
   ```

#### Create Encrypted Parameters
1. **Navigate to Systems Manager**:
   - Go to [Systems Manager console](https://console.aws.amazon.com/systems-manager)
   - Click **Parameter Store** in left navigation

2. **Create a parameter**:
   ```
   Name: /myapp/database/password
   Description: Database password for MyApp
   Tier: Standard
   Type: SecureString
   KMS Key Source: My current account
   KMS Key ID: alias/parameter-store-key
   Value: [Your actual password]
   ```

3. **Create additional parameters**:
   ```
   /myapp/api/key
   /myapp/database/username
   /myapp/external-service/token
   ```

### Step 2: Set Up Secrets Manager for Complex Secrets

#### Create a Database Secret
1. **Navigate to Secrets Manager**:
   - Go to [Secrets Manager console](https://console.aws.amazon.com/secretsmanager)
   - Click **Store a new secret**

2. **Configure secret type**:
   ```
   Secret type: Credentials for Amazon RDS database
   Username: myapp_user
   Password: [Generate or enter password]
   Encryption key: Default or select your KMS key
   Database: Select your RDS instance
   ```

3. **Name and configure**:
   ```
   Secret name: myapp/database/credentials
   Description: Database credentials for MyApp
   ```

4. **Configure automatic rotation** (optional):
   ```
   ✅ Enable automatic rotation
   Rotation interval: 30 days
   Lambda function: Create new function
   ```

#### Create an API Key Secret
1. **Create new secret**:
   ```
   Secret type: Other type of secret
   Key/value pairs:
   - api_key: your-api-key-value
   - api_secret: your-api-secret-value
   - endpoint: https://api.example.com
   ```

2. **Name the secret**:
   ```
   Secret name: myapp/external-api/credentials
   ```

### Step 3: Update Application Code

#### Python Example with Parameter Store
```python
import boto3
import json
from botocore.exceptions import ClientError

class ConfigManager:
    def __init__(self):
        self.ssm = boto3.client('ssm')
        self.secrets = boto3.client('secretsmanager')
    
    def get_parameter(self, parameter_name):
        """Get parameter from Parameter Store"""
        try:
            response = self.ssm.get_parameter(
                Name=parameter_name,
                WithDecryption=True
            )
            return response['Parameter']['Value']
        except ClientError as e:
            print(f"Error retrieving parameter {parameter_name}: {e}")
            return None
    
    def get_secret(self, secret_name):
        """Get secret from Secrets Manager"""
        try:
            response = self.secrets.get_secret_value(SecretId=secret_name)
            return json.loads(response['SecretString'])
        except ClientError as e:
            print(f"Error retrieving secret {secret_name}: {e}")
            return None

# Usage
config = ConfigManager()

# Get simple parameters
db_password = config.get_parameter('/myapp/database/password')
api_key = config.get_parameter('/myapp/api/key')

# Get complex secrets
db_credentials = config.get_secret('myapp/database/credentials')
if db_credentials:
    username = db_credentials['username']
    password = db_credentials['password']
```

#### Node.js Example
```javascript
const AWS = require('aws-sdk');

class ConfigManager {
    constructor() {
        this.ssm = new AWS.SSM();
        this.secretsManager = new AWS.SecretsManager();
    }
    
    async getParameter(parameterName) {
        try {
            const result = await this.ssm.getParameter({
                Name: parameterName,
                WithDecryption: true
            }).promise();
            return result.Parameter.Value;
        } catch (error) {
            console.error(`Error retrieving parameter ${parameterName}:`, error);
            return null;
        }
    }
    
    async getSecret(secretName) {
        try {
            const result = await this.secretsManager.getSecretValue({
                SecretId: secretName
            }).promise();
            return JSON.parse(result.SecretString);
        } catch (error) {
            console.error(`Error retrieving secret ${secretName}:`, error);
            return null;
        }
    }
}

// Usage
const config = new ConfigManager();

async function initializeApp() {
    const dbPassword = await config.getParameter('/myapp/database/password');
    const apiCredentials = await config.getSecret('myapp/external-api/credentials');
    
    // Use the secrets to configure your application
}
```

### Step 4: Configure IAM Permissions

#### Create Policy for Parameter Store Access
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "ssm:GetParameter",
                "ssm:GetParameters",
                "ssm:GetParametersByPath"
            ],
            "Resource": [
                "arn:aws:ssm:*:*:parameter/myapp/*"
            ]
        },
        {
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt"
            ],
            "Resource": [
                "arn:aws:kms:*:*:key/your-kms-key-id"
            ]
        }
    ]
}
```

#### Create Policy for Secrets Manager Access
```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": [
                "secretsmanager:GetSecretValue"
            ],
            "Resource": [
                "arn:aws:secretsmanager:*:*:secret:myapp/*"
            ]
        }
    ]
}
```

### Step 5: Implement Caching (Performance Optimization)

#### Python with Caching
```python
import boto3
import time
from functools import lru_cache

class CachedConfigManager:
    def __init__(self, cache_ttl=300):  # 5 minutes
        self.ssm = boto3.client('ssm')
        self.secrets = boto3.client('secretsmanager')
        self.cache_ttl = cache_ttl
        self._cache = {}
    
    def get_parameter_cached(self, parameter_name):
        cache_key = f"param:{parameter_name}"
        
        # Check cache
        if cache_key in self._cache:
            cached_time, value = self._cache[cache_key]
            if time.time() - cached_time < self.cache_ttl:
                return value
        
        # Fetch from Parameter Store
        try:
            response = self.ssm.get_parameter(
                Name=parameter_name,
                WithDecryption=True
            )
            value = response['Parameter']['Value']
            self._cache[cache_key] = (time.time(), value)
            return value
        except Exception as e:
            print(f"Error retrieving parameter {parameter_name}: {e}")
            return None
```

## ✅ Verification Steps

### Test Parameter Store Access
1. **Test from EC2 instance**:
   ```bash
   # Using AWS CLI
   aws ssm get-parameter --name "/myapp/database/password" --with-decryption
   
   # Should return the decrypted value
   ```

2. **Test from application**:
   - Deploy your updated application code
   - Verify it can retrieve parameters
   - Check application logs for any errors

### Test Secrets Manager Access
1. **Test secret retrieval**:
   ```bash
   aws secretsmanager get-secret-value --secret-id "myapp/database/credentials"
   ```

2. **Test automatic rotation** (if configured):
   - Trigger a rotation manually
   - Verify application continues to work with new credentials

### Test Error Handling
1. **Test with invalid parameter names**
2. **Test with insufficient permissions**
3. **Verify application handles errors gracefully**

## 💡 Best Practices

### Secret Organization
- **Hierarchical naming**: Use `/app/environment/service/secret` structure
- **Environment separation**: Different secrets for dev/staging/production
- **Principle of least privilege**: Grant access only to needed secrets
- **Regular rotation**: Rotate secrets regularly, especially for external services

### Performance Optimization
- **Client-side caching**: Cache secrets for reasonable periods (5-15 minutes)
- **Batch retrieval**: Use `GetParameters` for multiple parameters
- **Connection pooling**: Reuse AWS SDK clients
- **Regional deployment**: Deploy secrets in the same region as applications

### Security Considerations
- **Encryption in transit**: Always use HTTPS/TLS
- **Encryption at rest**: Use KMS keys for additional security
- **Access logging**: Monitor secret access patterns
- **Rotation strategy**: Plan for secret rotation without downtime

## 🚨 Common Mistakes to Avoid

- ❌ Hardcoding secrets in environment variables or config files
- ❌ Not implementing proper error handling for secret retrieval
- ❌ Caching secrets for too long (security risk)
- ❌ Not rotating secrets regularly
- ❌ Granting overly broad permissions to secrets
- ❌ Not monitoring secret access patterns
- ❌ Storing non-sensitive configuration in Secrets Manager (use Parameter Store)

## 🔧 Advanced Features

### Cross-Region Secret Replication
```bash
# Enable cross-region replication
aws secretsmanager replicate-secret-to-regions \
    --secret-id myapp/database/credentials \
    --add-replica-regions Region=us-west-2,KmsKeyId=alias/aws/secretsmanager
```

### Automatic Rotation with Lambda
```python
import boto3
import json

def lambda_handler(event, context):
    """Lambda function for automatic secret rotation"""
    
    secrets_client = boto3.client('secretsmanager')
    
    # Get the secret ARN and token from the event
    secret_arn = event['Step1']['SecretArn']
    token = event['Step1']['ClientRequestToken']
    
    # Implement rotation logic here
    # 1. Create new secret version
    # 2. Test new credentials
    # 3. Update application configuration
    # 4. Mark rotation as complete
    
    return {"statusCode": 200}
```

## 🔗 Official AWS Documentation

### Parameter Store
- [AWS Systems Manager Parameter Store](https://docs.aws.amazon.com/systems-manager/latest/userguide/systems-manager-parameter-store.html)
- [Creating SecureString parameters](https://docs.aws.amazon.com/systems-manager/latest/userguide/param-create-cli.html#param-create-cli-securestring)

### Secrets Manager
- [AWS Secrets Manager User Guide](https://docs.aws.amazon.com/secretsmanager/latest/userguide/)
- [Retrieving secrets from AWS Secrets Manager](https://docs.aws.amazon.com/secretsmanager/latest/userguide/retrieving-secrets.html)
- [Secrets Manager client-side caching](https://aws.amazon.com/blogs/security/use-aws-secrets-manager-client-side-caching-libraries-to-improve-the-availability-and-latency-of-using-your-secrets/)

## ➡️ What's Next?

Excellent! You've established secure secrets management for your applications. Now let's make sure secrets don't accidentally get exposed in your code:

**Next Control**: [WKLD.04 - Prevent Application Secrets from Being Exposed](./wkld-04.md)

---

[← Previous: WKLD.02](./wkld-02.md) | [Back to Main Guide](./README.md) | [Next: WKLD.04 →](./wkld-04.md)
