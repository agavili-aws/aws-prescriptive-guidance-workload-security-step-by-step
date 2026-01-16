# WKLD.07 - Log Data Events for S3 Buckets with Sensitive Data

## 🎯 What This Control Does

This control enables detailed logging of S3 data events (object-level operations) for buckets containing sensitive data, providing comprehensive audit trails for data access and modifications.

## 🔍 Why This Matters

- **Detailed audit trail**: Track who accessed what data and when
- **Security monitoring**: Detect unauthorized data access attempts
- **Compliance**: Meet regulatory requirements for data access logging
- **Incident investigation**: Detailed logs for forensic analysis
- **Data governance**: Monitor data usage patterns

## ⏱️ Time Required
**15 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Configure CloudTrail Data Events

#### Using AWS Console
1. **Navigate to CloudTrail Console**:
   - Go to [CloudTrail console](https://console.aws.amazon.com/cloudtrail)
   - Select your trail → **Edit**

2. **Configure data events**:
   ```
   Data events section:
   ✅ S3
   Individual bucket selection:
   - Select sensitive data buckets
   ✅ Read events (GetObject)
   ✅ Write events (PutObject, DeleteObject)
   ```

#### Using AWS CLI
```bash
# Configure data events for specific buckets
aws cloudtrail put-event-selectors \
    --trail-name MyTrail \
    --event-selectors '[
        {
            "ReadWriteType": "All",
            "IncludeManagementEvents": true,
            "DataResources": [
                {
                    "Type": "AWS::S3::Object",
                    "Values": [
                        "arn:aws:s3:::sensitive-data-bucket/*"
                    ]
                }
            ]
        }
    ]'
```

### Step 2: Enable S3 Access Logging

#### Configure S3 Access Logs
```bash
# Create logging bucket
aws s3 mb s3://my-s3-access-logs-bucket

# Enable access logging
aws s3api put-bucket-logging \
    --bucket sensitive-data-bucket \
    --bucket-logging-status '{
        "LoggingEnabled": {
            "TargetBucket": "my-s3-access-logs-bucket",
            "TargetPrefix": "access-logs/"
        }
    }'
```

## ✅ Verification Steps

### Test Data Event Logging
```bash
# Perform test operations
aws s3 cp test-file.txt s3://sensitive-data-bucket/
aws s3 cp s3://sensitive-data-bucket/test-file.txt downloaded-file.txt

# Check CloudTrail logs for data events
aws logs filter-log-events \
    --log-group-name CloudTrail/MyTrail \
    --filter-pattern "{ $.eventName = GetObject || $.eventName = PutObject }"
```

## 💡 Best Practices

- **Selective logging**: Only log sensitive buckets to control costs
- **Lifecycle policies**: Manage log retention and costs
- **Monitoring**: Set up alerts for unusual access patterns
- **Regular review**: Analyze logs for security insights

## 🚨 Common Mistakes to Avoid

- ❌ Logging all S3 buckets (high costs)
- ❌ Not setting up log analysis and monitoring
- ❌ Insufficient log retention policies
- ❌ Not protecting the log buckets themselves

## 🔗 Official AWS Documentation

- [Logging data events for trails](https://docs.aws.amazon.com/awscloudtrail/latest/userguide/logging-data-events-with-cloudtrail.html)
- [S3 server access logging](https://docs.aws.amazon.com/AmazonS3/latest/userguide/ServerLogs.html)

## ➡️ What's Next?

**Next Control**: [WKLD.08 - Encrypt Amazon EBS Volumes](./wkld-08.md)

---

[← Previous: WKLD.06](./wkld-06.md) | [Back to Main Guide](./README.md) | [Next: WKLD.08 →](./wkld-08.md)
