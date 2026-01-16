# WKLD.10 - Deploy Private Resources into Private Subnets

## 🎯 What This Control Does

This control ensures that resources that don't need direct internet access (like databases, application servers, and internal services) are deployed in private subnets, reducing their attack surface and improving security posture.

## 🔍 Why This Matters

- **Reduced attack surface**: Private resources can't be directly accessed from the internet
- **Network isolation**: Additional layer of security through network segmentation
- **Compliance**: Meets requirements for protecting sensitive workloads
- **Defense in depth**: Multiple layers of security controls
- **Controlled access**: All internet access must go through NAT Gateway or VPC endpoints

## ⏱️ Time Required
**30 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Create Private Subnets

#### Using AWS Console
1. **Navigate to VPC Console**:
   - Go to [VPC console](https://console.aws.amazon.com/vpc)
   - Click **Subnets** → **Create subnet**

2. **Configure private subnet**:
   ```
   VPC: Select your VPC
   Subnet name: Private-Subnet-1A
   Availability Zone: us-east-1a
   IPv4 CIDR block: 10.0.1.0/24
   ❌ Auto-assign public IPv4 address: Disabled
   ```

### Step 2: Configure Route Tables

#### Create Private Route Table
```bash
# Create route table for private subnet
aws ec2 create-route-table \
    --vpc-id vpc-12345678 \
    --tag-specifications 'ResourceType=route-table,Tags=[{Key=Name,Value=Private-Route-Table}]'

# Associate with private subnet
aws ec2 associate-route-table \
    --subnet-id subnet-12345678 \
    --route-table-id rtb-12345678
```

### Step 3: Deploy Resources in Private Subnets

#### Launch EC2 in Private Subnet
```bash
aws ec2 run-instances \
    --image-id ami-12345678 \
    --count 1 \
    --instance-type t3.micro \
    --subnet-id subnet-12345678 \
    --security-group-ids sg-12345678 \
    --no-associate-public-ip-address
```

#### Create RDS in Private Subnet
```bash
# Create DB subnet group
aws rds create-db-subnet-group \
    --db-subnet-group-name private-db-subnet-group \
    --db-subnet-group-description "Private subnets for RDS" \
    --subnet-ids subnet-12345678 subnet-87654321

# Create RDS instance in private subnets
aws rds create-db-instance \
    --db-instance-identifier myapp-database \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --master-username admin \
    --master-user-password MySecurePassword123 \
    --allocated-storage 20 \
    --db-subnet-group-name private-db-subnet-group \
    --no-publicly-accessible
```

## ✅ Verification Steps

### Verify Private Deployment
```bash
# Check that instances have no public IP
aws ec2 describe-instances \
    --filters "Name=subnet-id,Values=subnet-12345678" \
    --query 'Reservations[].Instances[].[InstanceId,PublicIpAddress,PrivateIpAddress]'

# Verify RDS is not publicly accessible
aws rds describe-db-instances \
    --db-instance-identifier myapp-database \
    --query 'DBInstances[0].PubliclyAccessible'
```

## 💡 Best Practices

- **Use multiple AZs** for high availability
- **Implement NAT Gateway** for outbound internet access
- **Use VPC endpoints** for AWS service access
- **Proper security groups** to control traffic flow

## 🚨 Common Mistakes to Avoid

- ❌ Accidentally enabling public IP assignment
- ❌ Not configuring NAT Gateway for internet access
- ❌ Overly permissive security groups
- ❌ Not planning for high availability across AZs

## 🔗 Official AWS Documentation

- [VPC subnets](https://docs.aws.amazon.com/vpc/latest/userguide/configure-subnets.html)
- [Infrastructure security in Amazon VPC](https://docs.aws.amazon.com/vpc/latest/userguide/infrastructure-security.html)

## ➡️ What's Next?

**Next Control**: [WKLD.11 - Restrict Network Access by Using Security Groups](./wkld-11.md)

---

[← Previous: WKLD.09](./wkld-09.md) | [Back to Main Guide](./README.md) | [Next: WKLD.11 →](./wkld-11.md)
