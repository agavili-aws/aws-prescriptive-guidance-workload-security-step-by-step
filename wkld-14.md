# WKLD.14 - Use Edge-Protection Services for Public Endpoints

## 🎯 What This Control Does

This control implements edge protection services like AWS WAF, CloudFront, and Application Load Balancers to provide an additional security layer between the internet and your applications.

## 🔍 Why This Matters

- **DDoS protection**: Built-in protection against Layer 3/4 attacks
- **Web application firewall**: Protection against Layer 7 attacks
- **Geographic filtering**: Block traffic from specific regions
- **Rate limiting**: Prevent abuse and brute force attacks
- **Performance**: Improved response times through edge caching

## ⏱️ Time Required
**40 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Set Up AWS WAF

```bash
# Create IP set for blocked IPs
aws wafv2 create-ip-set \
    --name BlockedIPs \
    --scope CLOUDFRONT \
    --ip-address-version IPV4 \
    --addresses 192.0.2.44/32 198.51.100.0/24

# Create rate limiting rule
aws wafv2 create-rule-group \
    --name RateLimitingRules \
    --scope CLOUDFRONT \
    --capacity 100 \
    --rules '[{
        "Name": "RateLimitRule",
        "Priority": 1,
        "Statement": {
            "RateBasedStatement": {
                "Limit": 2000,
                "AggregateKeyType": "IP"
            }
        },
        "Action": {"Block": {}},
        "VisibilityConfig": {
            "SampledRequestsEnabled": true,
            "CloudWatchMetricsEnabled": true,
            "MetricName": "RateLimitRule"
        }
    }]'
```

### Step 2: Create Web ACL

```bash
# Create Web ACL with managed rules
aws wafv2 create-web-acl \
    --name MyWebACL \
    --scope CLOUDFRONT \
    --default-action Allow={} \
    --rules '[
        {
            "Name": "AWSManagedRulesCommonRuleSet",
            "Priority": 1,
            "OverrideAction": {"None": {}},
            "Statement": {
                "ManagedRuleGroupStatement": {
                    "VendorName": "AWS",
                    "Name": "AWSManagedRulesCommonRuleSet"
                }
            },
            "VisibilityConfig": {
                "SampledRequestsEnabled": true,
                "CloudWatchMetricsEnabled": true,
                "MetricName": "CommonRuleSetMetric"
            }
        }
    ]'
```

### Step 3: Configure CloudFront with WAF

```bash
# Create CloudFront distribution with WAF
aws cloudfront create-distribution \
    --distribution-config '{
        "CallerReference": "protected-distribution-2024",
        "Origins": {
            "Quantity": 1,
            "Items": [{
                "Id": "protected-origin",
                "DomainName": "my-alb.us-east-1.elb.amazonaws.com",
                "CustomOriginConfig": {
                    "HTTPPort": 80,
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only"
                }
            }]
        },
        "DefaultCacheBehavior": {
            "TargetOriginId": "protected-origin",
            "ViewerProtocolPolicy": "redirect-to-https",
            "MinTTL": 0,
            "ForwardedValues": {
                "QueryString": false,
                "Cookies": {"Forward": "none"}
            }
        },
        "WebACLId": "arn:aws:wafv2:us-east-1:123456789012:global/webacl/MyWebACL/12345678-1234-1234-1234-123456789012",
        "Comment": "Protected distribution with WAF",
        "Enabled": true
    }'
```

## ✅ Verification Steps

### Test WAF Protection
```bash
# Test rate limiting (should be blocked after threshold)
for i in {1..2100}; do
    curl -s https://example.com > /dev/null
done

# Test blocked IP (if configured)
curl -H "X-Forwarded-For: 192.0.2.44" https://example.com

# Check WAF metrics in CloudWatch
aws cloudwatch get-metric-statistics \
    --namespace AWS/WAFV2 \
    --metric-name BlockedRequests \
    --dimensions Name=WebACL,Value=MyWebACL \
    --start-time 2024-01-01T00:00:00Z \
    --end-time 2024-01-01T23:59:59Z \
    --period 3600 \
    --statistics Sum
```

## 💡 Best Practices

- **Layer multiple protections**: Use CloudFront + WAF + ALB
- **Monitor and tune**: Regularly review WAF logs and adjust rules
- **Use managed rule sets**: Start with AWS managed rules
- **Geographic restrictions**: Block traffic from unused regions
- **Rate limiting**: Implement appropriate rate limits for your application

## 🚨 Common Mistakes to Avoid

- ❌ Not monitoring WAF logs for false positives
- ❌ Overly restrictive rules that block legitimate traffic
- ❌ Not testing WAF rules before production deployment
- ❌ Ignoring CloudWatch metrics and alarms
- ❌ Not updating WAF rules as threats evolve

## 🔗 Official AWS Documentation

- [Getting Started with AWS WAF](https://aws.amazon.com/waf/getting-started/)
- [Amazon CloudFront Developer Guide](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/)
- [Elastic Load Balancing User Guide](https://docs.aws.amazon.com/elasticloadbalancing/latest/userguide/)

## ➡️ What's Next?

**Next Control**: [WKLD.15 - Define Security Controls in Templates and Deploy with CI/CD](./wkld-15.md)

---

[← Previous: WKLD.13](./wkld-13.md) | [Back to Main Guide](./README.md) | [Next: WKLD.15 →](./wkld-15.md)
