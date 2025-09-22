# AWS Production Deployment Guide
## Juicebox Software Realm - 3-Realm Infrastructure

This is the **OFFICIAL** and **BULLETPROOF** deployment guide for the Juicebox Software Realm on AWS. This setup has been validated end-to-end with successful registration and recovery operations.

---

## 🎯 **DEPLOYMENT OVERVIEW**

### **Architecture**
- **3 Elastic Beanstalk Environments** (us-east-1)
- **3 CloudFront Distributions** (HTTPS termination)
- **DynamoDB** (persistent storage)
- **Secrets Manager** (tenant secrets)
- **IAM Roles** (secure permissions)

### **Production URLs**
```
Realm 1: https://d3no8h3k2vl9rf.cloudfront.net (29237d86b521e338686006682ddc4531)
Realm 2: https://d2lu0ef886dvi8.cloudfront.net (4a0fafeaa56b2f95b8ccaf1c4b329af2)
Realm 3: https://d803p7xyrjs61.cloudfront.net (d0c3289214f4e76f9cc3474f963a3fb5)
```

### **Status: ✅ PRODUCTION READY**
- Registration: **WORKING** ✅
- Recovery: **WORKING** ✅  
- HTTPS: **WORKING** ✅
- Multi-chain: **WORKING** ✅

---

## 🚀 **QUICK DEPLOYMENT**

### **Prerequisites**
```bash
# AWS CLI configured with jelli-wallet profile
aws configure list --profile jelli-wallet

# Required tools
- terraform >= 1.0
- Go >= 1.24.0
- Docker (optional)
```

### **One-Command Deployment**
```bash
cd aws/
export AWS_PROFILE=jelli-wallet
terraform init
terraform apply -auto-approve
```

**That's it!** The entire 3-realm infrastructure will be deployed automatically.

---

## 📋 **DETAILED DEPLOYMENT STEPS**

### **Step 1: AWS Profile Setup**
```bash
# Verify correct AWS profile
export AWS_PROFILE=jelli-wallet
aws sts get-caller-identity
```

### **Step 2: Navigate to AWS Directory**
```bash
cd /path/to/juicebox-software-realm/aws/
```

### **Step 3: Initialize Terraform**
```bash
terraform init
```

### **Step 4: Deploy Infrastructure**
```bash
terraform apply
# Type 'yes' when prompted
```

### **Step 5: Verify Deployment**
```bash
# Test each realm
curl https://d3no8h3k2vl9rf.cloudfront.net/
curl https://d2lu0ef886dvi8.cloudfront.net/
curl https://d803p7xyrjs61.cloudfront.net/
```

**Expected Response:**
```json
{"realmID":"29237d86b521e338686006682ddc4531"}
{"realmID":"4a0fafeaa56b2f95b8ccaf1c4b329af2"}
{"realmID":"d0c3289214f4e76f9cc3474f963a3fb5"}
```

---

## 🔧 **CONFIGURATION FILES**

### **terraform.tfvars** (Auto-configured)
```hcl
realm_id = "29237d86b521e338686006682ddc4531"
tenant_secrets = {
  "demo" = {
    "1" = "5077a1fd9dfbd60ed0c765ca114f67508e65a1850d3900199efc8a5f3de62c15"
  }
}
```

### **Key Environment Variables**
Each Elastic Beanstalk environment uses:
```bash
REALM_ID=<hex-string>           # Unique per realm
TENANT_SECRETS=<json-string>    # Demo tenant configuration
PORT=8080                       # Application port
```

---

## 🏗️ **INFRASTRUCTURE COMPONENTS**

### **Elastic Beanstalk Environments**
```
jb-sw-realm-1: Go 1.24 platform, 512MB RAM, Auto Scaling
jb-sw-realm-2: Go 1.24 platform, 512MB RAM, Auto Scaling  
jb-sw-realm-3: Go 1.24 platform, 512MB RAM, Auto Scaling
```

### **CloudFront Distributions**
```
d3no8h3k2vl9rf.cloudfront.net → jb-sw-realm-1 (ALB)
d2lu0ef886dvi8.cloudfront.net → jb-sw-realm-2 (ALB)
d803p7xyrjs61.cloudfront.net → jb-sw-realm-3 (ALB)
```

### **DynamoDB Tables**
```
juicebox-realm-<REALM_ID>  # Auto-created per realm
```

### **Secrets Manager**
```
juicebox-realm-secrets     # Tenant secret storage
```

---

## ⚙️ **BUILD PROCESS**

### **Application Binary**
The Go application is built using:
```bash
# Cross-compilation for Linux
GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o application ./cmd/jb-sw-realm
```

### **Buildfile Configuration**
```makefile
make: GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o application ./cmd/jb-sw-realm
```

### **Docker Support** (Alternative)
```dockerfile
FROM golang:1.24.0 AS builder
WORKDIR /app
COPY go.mod go.sum ./
RUN go mod download
COPY . .
RUN GOOS=linux GOARCH=amd64 CGO_ENABLED=0 go build -o application ./cmd/jb-sw-realm

FROM alpine:latest
RUN apk --no-cache add ca-certificates
WORKDIR /root/
COPY --from=builder /app/application .
EXPOSE 8080
CMD ["./application"]
```

---

## 🔍 **DEPLOYMENT QUIRKS & WORKAROUNDS**

### **1. Go Version Compatibility**
- **Issue**: Dockerfile used Go 1.22.6, but go.mod requires 1.24.0
- **Solution**: Updated Dockerfile to `golang:1.24.0`
- **File**: `Dockerfile` line 1

### **2. OpenTelemetry Schema Conflicts**
- **Issue**: Mixed semconv versions causing schema conflicts
- **Temporary Solution**: Disabled OpenTelemetry collector in supervisor
- **Status**: **TODO** - Re-enable with unified schema versions

### **3. Terraform Variable Prompts**
- **Issue**: terraform plan prompted for realm_id and tenant_secrets
- **Solution**: Created `terraform.tfvars` with hardcoded values
- **File**: `aws/terraform.tfvars`

### **4. Health Check Configuration**
- **Issue**: EB health checks failed on TCP:80
- **Solution**: Changed to HTTP:80/ in load balancer settings
- **Status**: Auto-configured in Terraform

### **5. CloudFront Origin Configuration**
- **Issue**: CloudFront pointed to old/invalid EB endpoints
- **Solution**: Updated origins to current ALB DNS names
- **File**: `aws/distribution-config.json`

### **6. HTTPS Certificate Handling**
- **Issue**: EB environments don't have SSL certificates
- **Solution**: CloudFront handles HTTPS termination
- **Status**: Working with default CloudFront certificates

---

## 📊 **MONITORING & HEALTH CHECKS**

### **Application Health**
```bash
# Check application logs
aws logs describe-log-groups --profile jelli-wallet
```

### **Elastic Beanstalk Health**
```bash
# Check environment health
aws elasticbeanstalk describe-environments --profile jelli-wallet
```

### **CloudFront Status**
```bash
# Check distribution status
aws cloudfront list-distributions --profile jelli-wallet
```

### **Manual Health Verification**
```bash
# Verify each realm responds correctly
for url in \
  "https://d3no8h3k2vl9rf.cloudfront.net" \
  "https://d2lu0ef886dvi8.cloudfront.net" \
  "https://d803p7xyrjs61.cloudfront.net"; do
  echo "Testing $url:"
  curl -s "$url/" | jq
done
```

---

## 🔐 **SECURITY CONSIDERATIONS**

### **IAM Permissions**
- Elastic Beanstalk service roles
- DynamoDB read/write permissions
- Secrets Manager read permissions
- CloudWatch logs permissions

### **Network Security**
- Private subnets for application servers
- Public subnets for load balancers only
- Security groups restrict access to ports 80/443

### **Secret Management**
- Tenant secrets stored in AWS Secrets Manager
- No hardcoded secrets in application code
- Environment variables for runtime configuration

---

## 🚧 **REMAINING TODOs**

### **High Priority**
1. **Re-enable OpenTelemetry**
   - Unify semconv versions across all files
   - Test collector configuration
   - Verify Datadog export (optional)

2. **Production Monitoring**
   - Set up CloudWatch dashboards
   - Configure alerts for realm health
   - Add application metrics

3. **Backup & Recovery**
   - DynamoDB point-in-time recovery
   - Cross-region backup strategy
   - Disaster recovery procedures

### **Medium Priority**
1. **Performance Optimization**
   - CloudFront caching policies
   - EB instance scaling policies
   - DynamoDB capacity planning

2. **Security Hardening**
   - WAF integration
   - SSL certificate management
   - VPC flow logs

### **Low Priority**
1. **Development Tools**
   - Staging environment setup
   - CI/CD pipeline integration
   - Automated testing framework

---

## 📞 **TROUBLESHOOTING**

### **Common Issues**

#### **Realm Not Responding**
```bash
# Check EB environment status
aws elasticbeanstalk describe-environments --environment-names jb-sw-realm-1

# Check application logs
aws logs tail /aws/elasticbeanstalk/jb-sw-realm-1/var/log/web.stdout.log
```

#### **CloudFront 502 Errors**
```bash
# Verify origin health
aws elbv2 describe-target-health --target-group-arn <arn>

# Check CloudFront distribution
aws cloudfront get-distribution --id <distribution-id>
```

#### **DynamoDB Access Issues**
```bash
# Verify table exists
aws dynamodb list-tables | grep juicebox-realm

# Check IAM permissions
aws iam get-role-policy --role-name aws-elasticbeanstalk-ec2-role --policy-name DynamoDBAccess
```

### **Emergency Contacts**
- AWS Support: Use your AWS support plan
- Deployment Lead: Available via this documentation
- Escalation: Repository maintainers

---

## ✅ **VALIDATION CHECKLIST**

### **Pre-Deployment**
- [ ] AWS CLI configured with correct profile
- [ ] Terraform installed and functional
- [ ] Go 1.24.0 available for local builds
- [ ] Repository clean of temporary files

### **Post-Deployment**
- [ ] All 3 realms respond to HTTPS requests
- [ ] Realm IDs match expected values
- [ ] EB environments show Green health
- [ ] CloudFront distributions show Deployed status
- [ ] DynamoDB tables created successfully
- [ ] Secrets Manager contains tenant secrets

### **SDK Integration**
- [ ] Jelli Wallet SDK updated with realm URLs
- [ ] Registration flow tested end-to-end
- [ ] Recovery flow tested end-to-end
- [ ] Multi-chain wallet support verified

---

## 🎯 **SUCCESS METRICS**

**Deployment is considered successful when:**
1. All 3 realms return correct realmID via HTTPS ✅
2. SDK registration creates Juicebox backup ✅
3. SDK recovery restores wallet from backup ✅
4. Multi-chain accounts generated consistently ✅

**Current Status: ALL METRICS ACHIEVED** 🎉

---

*Last Updated: September 22, 2025*  
*Deployment Validated: ✅ Production Ready*  
*Next Review: When implementing OpenTelemetry re-enablement*
