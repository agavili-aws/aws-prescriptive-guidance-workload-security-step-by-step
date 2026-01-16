# WKLD.12 - Use VPC Endpoints to Access Supported Services

## 🎯 What This Control Does

This control implements VPC endpoints to enable private connectivity to AWS services without requiring internet gateways, NAT devices, or VPN connections, keeping traffic within the AWS network.

## 🔍 Why This Matters

- **Enhanced security**: Traffic never leaves AWS network
- **Reduced costs**: Avoid NAT Gateway data processing charges
- **Better performance**: Lower latency and higher bandwidth
- **Network isolation**: Private IP connectivity to AWS services
- **Compliance**: Meet requirements for private network access

## ⏱️ Time Required
**35 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Create S3 Gateway Endpoint

```bash
# Create S3 gateway endpoint
aws ec2 create-vpc-endpoint \
    --vpc-id vpc-12345678 \
    --service-name com.amazonaws.us-east-1.s3 \
    --vpc-endpoint-type Gateway \
    --route-table-ids rtb-12345678
```

### Step 2: Create Interface Endpoints

```bash
# Create Systems Manager endpoints
for service in ssm ssmmessages ec2messages; do
    aws ec2 create-vpc-endpoint \
        --vpc-id vpc-12345678 \
        --service-name com.amazonaws.us-east-1.$service \
        --vpc-endpoint-type Interface \
        --subnet-ids subnet-12345678 \
        --security-group-ids sg-endpoint123456 \
        --private-dns-enabled
done
```

### Step 3: Configure Endpoint Security Groups

```bash
# Create security group for VPC endpoints
aws ec2 create-security-group \
    --group-name VPCEndpoint-SecurityGroup \
    --description "Security group for VPC endpoints" \
    --vpc-id vpc-12345678

# Allow HTTPS from private subnets
aws ec2 authorize-security-group-ingress \
    --group-id sg-endpoint123456 \
    --protocol tcp \
    --port 443 \
    --cidr 10.0.0.0/16
```

## ✅ Verification Steps

### Test VPC Endpoint Connectivity
```bash
# From EC2 instance in private subnet
aws s3 ls  # Should work through S3 gateway endpoint
aws ssm get-parameter --name "/test/parameter"  # Should work through SSM interface endpoint
```

## 💡 Best Practices

- **Use gateway endpoints** for S3 and DynamoDB (no additional cost)
- **Interface endpoints** for other services when needed
- **Endpoint policies** to restrict access
- **Security groups** to control endpoint access
- **Monitor costs** for interface endpoints

## 🚨 Common Mistakes to Avoid

- ❌ Not configuring security groups for interface endpoints
- ❌ Using interface endpoints when gateway endpoints are available
- ❌ Not enabling private DNS for interface endpoints
- ❌ Overly permissive endpoint policies

## 🔗 Official AWS Documentation

- [VPC endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/vpc-endpoints.html)
- [Gateway endpoints](https://docs.aws.amazon.com/vpc/latest/privatelink/gateway-endpoints.html)

## ➡️ What's Next?

**Next Control**: [WKLD.13 - Require HTTPS for All Public Web Endpoints](./wkld-13.md)

---

[← Previous: WKLD.11](./wkld-11.md) | [Back to Main Guide](./README.md) | [Next: WKLD.13 →](./wkld-13.md)
