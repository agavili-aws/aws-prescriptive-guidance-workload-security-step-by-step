# AWS Workload Security Baseline for Early Stage Customers

Welcome! This guide provides a friendly, step-by-step walkthrough of the AWS Prescriptive Guidance for securing your workloads running on AWS. This is the next step after securing your AWS account--now let's secure the applications and infrastructure you build.

## 🎯 What You'll Accomplish

By following this guide, you'll establish secure practices for your AWS workloads including:

- ✅ Secure credential and secrets management
- ✅ Proper IAM roles for compute resources
- ✅ Encrypted data at rest and in transit
- ✅ Network security and isolation
- ✅ Secure remote access without SSH/RDP
- ✅ Infrastructure as Code security practices

## 📋 Workload Security Controls Overview

This guide covers 15 essential workload security controls from the [AWS Prescriptive Guidance](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/controls-wkld.html):

| Control | Description | Difficulty | Time Required |
|---------|-------------|------------|---------------|
| [WKLD.01](./wkld-01.md) | Use IAM roles for compute environment permissions | Easy | 15 minutes |
| [WKLD.02](./wkld-02.md) | Restrict credential usage scope with resource-based policies | Medium | 20 minutes |
| [WKLD.03](./wkld-03.md) | Use ephemeral secrets or a secrets-management service | Medium | 25 minutes |
| [WKLD.04](./wkld-04.md) | Prevent application secrets from being exposed | Easy | 15 minutes |
| [WKLD.05](./wkld-05.md) | Detect and remediate exposed secrets | Medium | 20 minutes |
| [WKLD.06](./wkld-06.md) | Use Systems Manager instead of SSH or RDP | Medium | 25 minutes |
| [WKLD.07](./wkld-07.md) | Log data events for S3 buckets with sensitive data | Easy | 15 minutes |
| [WKLD.08](./wkld-08.md) | Encrypt Amazon EBS volumes | Easy | 10 minutes |
| [WKLD.09](./wkld-09.md) | Encrypt Amazon RDS databases | Easy | 10 minutes |
| [WKLD.10](./wkld-10.md) | Deploy private resources into private subnets | Medium | 30 minutes |
| [WKLD.11](./wkld-11.md) | Restrict network access by using security groups | Medium | 25 minutes |
| [WKLD.12](./wkld-12.md) | Use VPC endpoints to access supported services | Hard | 35 minutes |
| [WKLD.13](./wkld-13.md) | Require HTTPS for all public web endpoints | Medium | 20 minutes |
| [WKLD.14](./wkld-14.md) | Use edge-protection services for public endpoints | Hard | 40 minutes |
| [WKLD.15](./wkld-15.md) | Define security controls in templates and deploy with CI/CD | Hard | 45 minutes |

**Total estimated time: 5.5 hours**

## 🚀 Getting Started

### Prerequisites

Before you begin, make sure you have:

- [ ] Completed the [AWS Account Security Baseline](https://github.com/rushealy-aws/aws-prescriptive-guidance-account-security-step-by-step)
- [ ] An AWS account with proper IAM permissions
- [ ] Basic understanding of AWS services (EC2, S3, VPC)
- [ ] Access to the AWS Management Console and CLI

### Recommended Order

These controls build upon each other, so follow this recommended sequence:

#### Phase 1: Identity and Secrets (30-45 minutes)
1. **WKLD.01** - Set up IAM roles for compute
2. **WKLD.02** - Configure resource-based policies  
3. **WKLD.03** - Implement secrets management

#### Phase 2: Development Security (45-60 minutes)
4. **WKLD.04** - Prevent secrets exposure
5. **WKLD.05** - Detect exposed secrets
6. **WKLD.06** - Secure remote access

#### Phase 3: Data Protection (30-45 minutes)
7. **WKLD.07** - Enhanced S3 logging
8. **WKLD.08** - EBS encryption
9. **WKLD.09** - RDS encryption

#### Phase 4: Network Security (90-120 minutes)
10. **WKLD.10** - Private subnet deployment
11. **WKLD.11** - Security group configuration
12. **WKLD.12** - VPC endpoints

#### Phase 5: Public-Facing Security (60-90 minutes)
13. **WKLD.13** - HTTPS enforcement
14. **WKLD.14** - Edge protection services
15. **WKLD.15** - Infrastructure as Code

## 💡 Tips for Success

- **Start with existing workloads**: Apply these controls to resources you already have
- **Test in development first**: Always test security changes in non-production environments
- **Document your architecture**: Keep track of your security decisions and configurations
- **Automate when possible**: Use Infrastructure as Code for repeatable security

## 🔗 Related Resources

### Prerequisites
- **Account Security**: [AWS Account Security Baseline](https://github.com/rushealy-aws/aws-prescriptive-guidance-account-security-step-by-step) (complete this first!)

### AWS Documentation
- [AWS Security Best Practices](https://docs.aws.amazon.com/security/)
- [AWS Well-Architected Security Pillar](https://docs.aws.amazon.com/wellarchitected/latest/security-pillar/)
- [AWS Security Reference Architecture](https://docs.aws.amazon.com/prescriptive-guidance/latest/security-reference-architecture/)

### Tools and Services
- [AWS Security Hub](https://aws.amazon.com/security-hub/) - Centralized security findings
- [AWS Config](https://aws.amazon.com/config/) - Configuration compliance monitoring
- [AWS Systems Manager](https://aws.amazon.com/systems-manager/) - Operational insights and actions

## 🆘 Need Help?

- **AWS Documentation**: Each control page includes links to official AWS documentation
- **AWS Support**: If you have a support plan, don't hesitate to reach out
- **AWS Community**: The [AWS re:Post community](https://repost.aws/) is a great resource for questions
- **AWS Training**: Consider [AWS Security Learning Path](https://aws.amazon.com/training/learning-paths/security/)

---

**Ready to secure your workloads?** Start with [WKLD.01 - Use IAM Roles for Compute Environment Permissions](./wkld-01.md) and work your way through each control.

*This guide is based on the [AWS Prescriptive Guidance for Startup Security Baseline](https://docs.aws.amazon.com/prescriptive-guidance/latest/aws-startup-security-baseline/controls-wkld.html). Last updated: September 2024.*
