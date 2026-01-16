# WKLD.13 - Require HTTPS for All Public Web Endpoints

## 🎯 What This Control Does

This control ensures all public web endpoints use HTTPS encryption, protecting data in transit and providing authentication of web services to users.

## 🔍 Why This Matters

- **Data protection**: Encrypts data in transit between clients and servers
- **Authentication**: Proves identity of web services to users
- **SEO benefits**: Search engines favor HTTPS sites
- **User trust**: Modern browsers warn about non-HTTPS sites
- **Compliance**: Many regulations require encrypted web traffic

## ⏱️ Time Required
**20 minutes**

## 📋 Step-by-Step Instructions

### Step 1: Configure Application Load Balancer HTTPS

```bash
# Create SSL certificate (use ACM)
aws acm request-certificate \
    --domain-name example.com \
    --subject-alternative-names www.example.com \
    --validation-method DNS

# Create HTTPS listener
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-load-balancer/50dc6c495c0c9188 \
    --protocol HTTPS \
    --port 443 \
    --certificates CertificateArn=arn:aws:acm:us-east-1:123456789012:certificate/12345678-1234-1234-1234-123456789012 \
    --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:us-east-1:123456789012:targetgroup/my-targets/73e2d6bc24d8a067

# Redirect HTTP to HTTPS
aws elbv2 create-listener \
    --load-balancer-arn arn:aws:elasticloadbalancing:us-east-1:123456789012:loadbalancer/app/my-load-balancer/50dc6c495c0c9188 \
    --protocol HTTP \
    --port 80 \
    --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
```

### Step 2: Configure CloudFront HTTPS

```bash
# Create CloudFront distribution with HTTPS redirect
aws cloudfront create-distribution \
    --distribution-config '{
        "CallerReference": "my-distribution-2024",
        "Origins": {
            "Quantity": 1,
            "Items": [{
                "Id": "my-origin",
                "DomainName": "my-alb-123456789.us-east-1.elb.amazonaws.com",
                "CustomOriginConfig": {
                    "HTTPPort": 80,
                    "HTTPSPort": 443,
                    "OriginProtocolPolicy": "https-only"
                }
            }]
        },
        "DefaultCacheBehavior": {
            "TargetOriginId": "my-origin",
            "ViewerProtocolPolicy": "redirect-to-https",
            "MinTTL": 0,
            "ForwardedValues": {
                "QueryString": false,
                "Cookies": {"Forward": "none"}
            }
        },
        "Comment": "My HTTPS-only distribution",
        "Enabled": true
    }'
```

## ✅ Verification Steps

### Test HTTPS Enforcement
```bash
# Test HTTP redirect
curl -I http://example.com
# Should return 301/302 redirect to HTTPS

# Test HTTPS access
curl -I https://example.com
# Should return 200 OK

# Test SSL certificate
openssl s_client -connect example.com:443 -servername example.com
```

## 💡 Best Practices

- **Use AWS Certificate Manager** for free SSL certificates
- **Enable HTTP to HTTPS redirect** on all load balancers
- **Use modern TLS versions** (1.2 minimum, prefer 1.3)
- **Implement HSTS headers** for additional security
- **Regular certificate renewal** (ACM handles this automatically)

## 🚨 Common Mistakes to Avoid

- ❌ Not redirecting HTTP traffic to HTTPS
- ❌ Using self-signed certificates in production
- ❌ Not configuring proper security headers
- ❌ Allowing weak cipher suites
- ❌ Not testing certificate validation

## 🔗 Official AWS Documentation

- [Using HTTPS with CloudFront](https://docs.aws.amazon.com/AmazonCloudFront/latest/DeveloperGuide/using-https.html)
- [HTTPS listeners for Application Load Balancers](https://docs.aws.amazon.com/elasticloadbalancing/latest/application/create-https-listener.html)

## ➡️ What's Next?

**Next Control**: [WKLD.14 - Use Edge-Protection Services for Public Endpoints](./wkld-14.md)

---

[← Previous: WKLD.12](./wkld-12.md) | [Back to Main Guide](./README.md) | [Next: WKLD.14 →](./wkld-14.md)
