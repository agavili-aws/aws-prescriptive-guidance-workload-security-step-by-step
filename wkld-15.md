# WKLD.15 - Define Security Controls in Templates and Deploy with CI/CD

## 🎯 What This Control Does

This control implements Infrastructure as Code (IaC) practices to define security controls in templates and deploy them through automated CI/CD pipelines, ensuring consistent and repeatable security configurations.

## 🔍 Why This Matters

- **Consistency**: Same security controls across all environments
- **Version control**: Track changes to security configurations
- **Automated deployment**: Reduce human error in security setup
- **Compliance**: Auditable infrastructure changes
- **Scalability**: Easily replicate secure architectures

## ⏱️ Time Required
**45 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Create CloudFormation Security Template

```yaml
# security-baseline.yaml
AWSTemplateFormatVersion: '2010-09-09'
Description: 'Security baseline template'

Parameters:
  Environment:
    Type: String
    AllowedValues: [dev, staging, prod]
    Default: dev

Resources:
  # KMS Key for encryption
  EncryptionKey:
    Type: AWS::KMS::Key
    Properties:
      Description: !Sub 'Encryption key for ${Environment} environment'
      KeyPolicy:
        Statement:
          - Sid: Enable IAM User Permissions
            Effect: Allow
            Principal:
              AWS: !Sub 'arn:aws:iam::${AWS::AccountId}:root'
            Action: 'kms:*'
            Resource: '*'

  # Security Group for web tier
  WebSecurityGroup:
    Type: AWS::EC2::SecurityGroup
    Properties:
      GroupDescription: Security group for web tier
      VpcId: !Ref VPC
      SecurityGroupIngress:
        - IpProtocol: tcp
          FromPort: 443
          ToPort: 443
          CidrIp: 0.0.0.0/0
          Description: HTTPS from internet
      Tags:
        - Key: Environment
          Value: !Ref Environment

  # WAF Web ACL
  WebACL:
    Type: AWS::WAFv2::WebACL
    Properties:
      Name: !Sub '${Environment}-WebACL'
      Scope: CLOUDFRONT
      DefaultAction:
        Allow: {}
      Rules:
        - Name: AWSManagedRulesCommonRuleSet
          Priority: 1
          OverrideAction:
            None: {}
          Statement:
            ManagedRuleGroupStatement:
              VendorName: AWS
              Name: AWSManagedRulesCommonRuleSet
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: CommonRuleSetMetric

Outputs:
  EncryptionKeyId:
    Description: KMS Key ID for encryption
    Value: !Ref EncryptionKey
    Export:
      Name: !Sub '${AWS::StackName}-EncryptionKey'
```

### Step 2: Create CDK Security Stack

```typescript
// security-stack.ts
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as wafv2 from 'aws-cdk-lib/aws-wafv2';
import * as kms from 'aws-cdk-lib/aws-kms';

export class SecurityStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // KMS Key for encryption
    const encryptionKey = new kms.Key(this, 'EncryptionKey', {
      description: 'Encryption key for workload security',
      enableKeyRotation: true,
    });

    // Security Group with least privilege
    const webSecurityGroup = new ec2.SecurityGroup(this, 'WebSecurityGroup', {
      vpc: props.vpc,
      description: 'Security group for web tier',
      allowAllOutbound: false,
    });

    webSecurityGroup.addIngressRule(
      ec2.Peer.anyIpv4(),
      ec2.Port.tcp(443),
      'HTTPS from internet'
    );

    // WAF Web ACL
    const webAcl = new wafv2.CfnWebACL(this, 'WebACL', {
      scope: 'CLOUDFRONT',
      defaultAction: { allow: {} },
      rules: [
        {
          name: 'AWSManagedRulesCommonRuleSet',
          priority: 1,
          overrideAction: { none: {} },
          statement: {
            managedRuleGroupStatement: {
              vendorName: 'AWS',
              name: 'AWSManagedRulesCommonRuleSet',
            },
          },
          visibilityConfig: {
            sampledRequestsEnabled: true,
            cloudWatchMetricsEnabled: true,
            metricName: 'CommonRuleSetMetric',
          },
        },
      ],
    });

    // Outputs
    new cdk.CfnOutput(this, 'EncryptionKeyId', {
      value: encryptionKey.keyId,
      exportName: 'EncryptionKeyId',
    });
  }
}
```

### Step 3: Set Up CI/CD Pipeline

```yaml
# .github/workflows/security-deployment.yml
name: Security Infrastructure Deployment

on:
  push:
    branches: [main]
    paths: ['infrastructure/**']
  pull_request:
    branches: [main]
    paths: ['infrastructure/**']

jobs:
  security-scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Run Checkov security scan
        uses: bridgecrewio/checkov-action@master
        with:
          directory: infrastructure/
          framework: cloudformation,terraform
          output_format: sarif
          output_file_path: checkov-report.sarif
      
      - name: Upload Checkov results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: checkov-report.sarif

  deploy-dev:
    needs: security-scan
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Deploy to dev environment
        run: |
          aws cloudformation deploy \
            --template-file infrastructure/security-baseline.yaml \
            --stack-name security-baseline-dev \
            --parameter-overrides Environment=dev \
            --capabilities CAPABILITY_IAM

  deploy-prod:
    needs: security-scan
    if: github.ref == 'refs/heads/main'
    runs-on: ubuntu-latest
    environment: production
    steps:
      - uses: actions/checkout@v3
      
      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v2
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: us-east-1
      
      - name: Deploy to production
        run: |
          aws cloudformation deploy \
            --template-file infrastructure/security-baseline.yaml \
            --stack-name security-baseline-prod \
            --parameter-overrides Environment=prod \
            --capabilities CAPABILITY_IAM
```

### Step 4: Implement Security Testing

```bash
# security-tests.sh
#!/bin/bash

echo "Running security compliance tests..."

# Test 1: Verify encryption is enabled
echo "Testing EBS encryption..."
ENCRYPTION_STATUS=$(aws ec2 describe-ebs-encryption-by-default --query 'EbsEncryptionByDefault' --output text)
if [ "$ENCRYPTION_STATUS" = "True" ]; then
    echo "✅ EBS encryption by default is enabled"
else
    echo "❌ EBS encryption by default is NOT enabled"
    exit 1
fi

# Test 2: Verify security groups don't allow 0.0.0.0/0 on sensitive ports
echo "Testing security group rules..."
OPEN_SSH=$(aws ec2 describe-security-groups \
    --filters "Name=ip-permission.from-port,Values=22" "Name=ip-permission.cidr,Values=0.0.0.0/0" \
    --query 'SecurityGroups[].GroupId' --output text)

if [ -z "$OPEN_SSH" ]; then
    echo "✅ No security groups allow SSH from 0.0.0.0/0"
else
    echo "❌ Found security groups allowing SSH from anywhere: $OPEN_SSH"
    exit 1
fi

# Test 3: Verify S3 buckets are not publicly accessible
echo "Testing S3 bucket public access..."
PUBLIC_BUCKETS=$(aws s3api list-buckets --query 'Buckets[].Name' --output text | \
    xargs -I {} aws s3api get-bucket-acl --bucket {} --query 'Grants[?Grantee.URI==`http://acs.amazonaws.com/groups/global/AllUsers`].Permission' --output text 2>/dev/null)

if [ -z "$PUBLIC_BUCKETS" ]; then
    echo "✅ No publicly accessible S3 buckets found"
else
    echo "❌ Found publicly accessible S3 buckets"
    exit 1
fi

echo "All security tests passed! ✅"
```

## ✅ Verification Steps

### Test Infrastructure Deployment
```bash
# Deploy security baseline
aws cloudformation deploy \
    --template-file security-baseline.yaml \
    --stack-name security-baseline-test \
    --parameter-overrides Environment=test \
    --capabilities CAPABILITY_IAM

# Verify stack deployment
aws cloudformation describe-stacks \
    --stack-name security-baseline-test \
    --query 'Stacks[0].StackStatus'

# Run security tests
./security-tests.sh
```

## 💡 Best Practices

### Template Design
- **Modular templates**: Break down into reusable components
- **Parameter validation**: Use allowed values and constraints
- **Tagging strategy**: Consistent tagging across all resources
- **Documentation**: Include clear descriptions and comments

### CI/CD Security
- **Security scanning**: Integrate tools like Checkov, cfn-lint
- **Approval gates**: Require manual approval for production
- **Rollback capability**: Plan for quick rollback procedures
- **Secrets management**: Use secure methods for credentials

### Testing and Validation
- **Automated testing**: Test security configurations automatically
- **Compliance checks**: Verify against security standards
- **Drift detection**: Monitor for configuration drift
- **Regular audits**: Periodic review of deployed resources

## 🚨 Common Mistakes to Avoid

- ❌ Not testing templates before production deployment
- ❌ Hardcoding sensitive values in templates
- ❌ Not implementing proper approval workflows
- ❌ Ignoring security scan results
- ❌ Not monitoring for configuration drift
- ❌ Overly complex templates that are hard to maintain

## 🔧 Advanced Features

### Custom Config Rules
```yaml
SecurityComplianceRule:
  Type: AWS::Config::ConfigRule
  Properties:
    ConfigRuleName: security-group-ssh-check
    Source:
      Owner: AWS
      SourceIdentifier: INCOMING_SSH_DISABLED
    Scope:
      ComplianceResourceTypes:
        - AWS::EC2::SecurityGroup
```

### Automated Remediation
```yaml
RemediationConfiguration:
  Type: AWS::Config::RemediationConfiguration
  Properties:
    ConfigRuleName: !Ref SecurityComplianceRule
    TargetType: SSM_DOCUMENT
    TargetId: AWSConfigRemediation-RemoveUnrestrictedSourceInSecurityGroup
    TargetVersion: "1"
    Parameters:
      AutomationAssumeRole:
        StaticValue: !GetAtt RemediationRole.Arn
      GroupId:
        ResourceValue: RESOURCE_ID
```

## 🔗 Official AWS Documentation

- [AWS CloudFormation User Guide](https://docs.aws.amazon.com/cloudformation/)
- [AWS CDK Developer Guide](https://docs.aws.amazon.com/cdk/v2/guide/)
- [AWS CodePipeline User Guide](https://docs.aws.amazon.com/codepipeline/)
- [Infrastructure as Code Best Practices](https://docs.aws.amazon.com/whitepapers/latest/introduction-devops-aws/infrastructure-as-code.html)

## 🎉 Congratulations!

You've completed all 15 workload security controls! Your AWS workloads now have a comprehensive security foundation including:

✅ **Identity & Access**: IAM roles, resource policies, secrets management  
✅ **Data Protection**: Encryption at rest and in transit  
✅ **Network Security**: Private subnets, security groups, VPC endpoints  
✅ **Monitoring**: Enhanced logging and secret detection  
✅ **Infrastructure Security**: HTTPS, edge protection, Infrastructure as Code  

## ➡️ Continue Your Security Journey

### Next Steps
- **Implement AWS Security Hub** for centralized security findings
- **Set up AWS Config** for configuration compliance monitoring  
- **Consider AWS Control Tower** for multi-account governance
- **Explore AWS Well-Architected** security pillar reviews

### Additional Resources
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/)
- [AWS Security Best Practices](https://aws.amazon.com/architecture/security-identity-compliance/)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)

---

[← Previous: WKLD.14](./wkld-14.md) | [Back to Main Guide](./README.md)

**🎯 You've successfully implemented comprehensive workload security controls! Your applications and infrastructure are now significantly more secure.**
