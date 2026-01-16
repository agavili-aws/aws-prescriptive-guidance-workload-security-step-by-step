# WKLD.09 - Encrypt Amazon RDS Databases

## 🎯 What This Control Does

This control enables encryption at rest for Amazon RDS database instances, protecting your database data with AWS KMS encryption keys. This ensures all data stored in your databases is encrypted automatically.

## 🔍 Why This Matters

- **Data protection**: Encrypts sensitive database data at rest
- **Compliance**: Meets regulatory requirements for data encryption
- **No performance impact**: Minimal latency with same performance characteristics
- **Automatic**: Encryption is transparent to applications
- **Backup protection**: All backups and snapshots are also encrypted

## ⏱️ Time Required
**10 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Enable Encryption for New RDS Instances

#### Using AWS Console
1. **Navigate to RDS Console**:
   - Go to [RDS console](https://console.aws.amazon.com/rds)
   - Click **Create database**

2. **Configure encryption**:
   ```
   In Database options section:
   ✅ Enable encryption
   Master key: (default) aws/rds or select custom KMS key
   ```

#### Using AWS CLI
```bash
# Create encrypted RDS instance
aws rds create-db-instance \
    --db-instance-identifier myapp-database \
    --db-instance-class db.t3.micro \
    --engine mysql \
    --master-username admin \
    --master-user-password MySecurePassword123 \
    --allocated-storage 20 \
    --storage-encrypted \
    --kms-key-id alias/aws/rds
```

### Step 2: Encrypt Existing Unencrypted Databases

#### Create Encrypted Copy
```bash
# 1. Create snapshot of unencrypted database
aws rds create-db-snapshot \
    --db-instance-identifier myapp-database \
    --db-snapshot-identifier myapp-snapshot-before-encryption

# 2. Copy snapshot with encryption
aws rds copy-db-snapshot \
    --source-db-snapshot-identifier myapp-snapshot-before-encryption \
    --target-db-snapshot-identifier myapp-encrypted-snapshot \
    --kms-key-id alias/aws/rds

# 3. Restore from encrypted snapshot
aws rds restore-db-instance-from-db-snapshot \
    --db-instance-identifier myapp-database-encrypted \
    --db-snapshot-identifier myapp-encrypted-snapshot
```

## ✅ Verification Steps

### Verify Encryption Status
```bash
# Check if database is encrypted
aws rds describe-db-instances \
    --db-instance-identifier myapp-database \
    --query 'DBInstances[0].StorageEncrypted'
```

## 💡 Best Practices

- **Use encryption by default** for all new databases
- **Plan migration** for existing unencrypted databases
- **Use AWS managed keys** unless compliance requires customer managed keys
- **Test applications** after encryption migration

## 🚨 Common Mistakes to Avoid

- ❌ Not planning for downtime during encryption migration
- ❌ Forgetting to encrypt read replicas
- ❌ Not testing backup/restore procedures with encrypted databases

## 🔗 Official AWS Documentation

- [Encrypting Amazon RDS resources](https://docs.aws.amazon.com/AmazonRDS/latest/UserGuide/Overview.Encryption.html)

## ➡️ What's Next?

**Next Control**: [WKLD.10 - Deploy Private Resources into Private Subnets](./wkld-10.md)

---

[← Previous: WKLD.08](./wkld-08.md) | [Back to Main Guide](./README.md) | [Next: WKLD.10 →](./wkld-10.md)
