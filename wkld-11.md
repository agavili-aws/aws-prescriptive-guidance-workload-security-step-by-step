# WKLD.11 - Restrict Network Access by Using Security Groups

## 🎯 What This Control Does

This control implements proper security group configurations to act as virtual firewalls, controlling inbound and outbound traffic to AWS resources based on the principle of least privilege.

## 🔍 Why This Matters

- **Network segmentation**: Control traffic flow between different tiers
- **Principle of least privilege**: Only allow necessary network access
- **Defense in depth**: Additional security layer beyond application controls
- **Compliance**: Meet network security requirements
- **Attack surface reduction**: Minimize exposed network services

## ⏱️ Time Required
**25 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Design Security Group Architecture

#### Three-Tier Architecture Example
```
Internet → Load Balancer SG → Application SG → Database SG
```

### Step 2: Create Load Balancer Security Group

```bash
# Create ALB security group
aws ec2 create-security-group \
    --group-name ALB-SecurityGroup \
    --description "Security group for Application Load Balancer" \
    --vpc-id vpc-12345678

# Allow HTTPS from internet
aws ec2 authorize-security-group-ingress \
    --group-id sg-alb123456 \
    --protocol tcp \
    --port 443 \
    --cidr 0.0.0.0/0

# Allow HTTP (redirect to HTTPS)
aws ec2 authorize-security-group-ingress \
    --group-id sg-alb123456 \
    --protocol tcp \
    --port 80 \
    --cidr 0.0.0.0/0
```

### Step 3: Create Application Security Group

```bash
# Create application security group
aws ec2 create-security-group \
    --group-name App-SecurityGroup \
    --description "Security group for application servers" \
    --vpc-id vpc-12345678

# Allow HTTP from ALB only
aws ec2 authorize-security-group-ingress \
    --group-id sg-app123456 \
    --protocol tcp \
    --port 80 \
    --source-group sg-alb123456
```

### Step 4: Create Database Security Group

```bash
# Create database security group
aws ec2 create-security-group \
    --group-name DB-SecurityGroup \
    --description "Security group for database servers" \
    --vpc-id vpc-12345678

# Allow MySQL from application servers only
aws ec2 authorize-security-group-ingress \
    --group-id sg-db123456 \
    --protocol tcp \
    --port 3306 \
    --source-group sg-app123456

# Remove default outbound rule (optional, for maximum security)
aws ec2 revoke-security-group-egress \
    --group-id sg-db123456 \
    --protocol all \
    --cidr 0.0.0.0/0
```

## ✅ Verification Steps

### Test Security Group Rules
```bash
# List security group rules
aws ec2 describe-security-groups \
    --group-ids sg-app123456 \
    --query 'SecurityGroups[0].IpPermissions'

# Test connectivity between tiers
# Should work: ALB → App → Database
# Should fail: Direct internet → App or Database
```

## 💡 Best Practices

- **Least privilege**: Only allow necessary ports and sources
- **Reference security groups**: Use security group IDs as sources instead of IP ranges
- **Regular audits**: Review and clean up unused rules
- **Descriptive names**: Use clear naming conventions
- **Documentation**: Document the purpose of each security group

## 🚨 Common Mistakes to Avoid

- ❌ Using 0.0.0.0/0 for internal communications
- ❌ Opening unnecessary ports
- ❌ Not removing default outbound rules when appropriate
- ❌ Overly complex security group structures
- ❌ Not testing connectivity after changes

## 🔗 Official AWS Documentation

- [Control traffic to resources using security groups](https://docs.aws.amazon.com/vpc/latest/userguide/vpc-security-groups.html)

## ➡️ What's Next?

**Next Control**: [WKLD.12 - Use VPC Endpoints to Access Supported Services](./wkld-12.md)

---

[← Previous: WKLD.10](./wkld-10.md) | [Back to Main Guide](./README.md) | [Next: WKLD.12 →](./wkld-12.md)
