# Multi-Region Deployment Guide
## Deploying Juicebox Realms to Multiple AWS Regions

This guide explains how to replicate the current us-east-1 development deployment to us-east-2 for production use.

---

## 🎯 **TODAY'S ACCOMPLISHMENTS SUMMARY**

### **What We Built in us-east-1 (Development)**
✅ **Complete 3-Realm Infrastructure:**
- 3 Elastic Beanstalk environments running Go 1.24 applications
- 3 CloudFront distributions for HTTPS termination
- DynamoDB tables for persistent storage (auto-created per realm)
- Secrets Manager for tenant secret storage
- VPC with public/private subnets and proper security groups

✅ **Production-Ready Configuration:**
- Cross-compiled Go applications for Linux
- Proper environment variable configuration
- Health checks configured for HTTP:80/
- Auto-scaling enabled
- All quirks and issues resolved

✅ **Validated End-to-End Functionality:**
- SDK registration flow working
- SDK recovery flow working  
- Multi-chain wallet support (Bitcoin, Ethereum, Solana, Base)
- HTTPS access via CloudFront
- Proper realm ID responses

✅ **Clean Repository & Documentation:**
- Removed all temporary files and build artifacts
- Created comprehensive deployment documentation
- Documented all technical quirks and solutions
- Prioritized TODO list for future enhancements

### **Current Architecture (us-east-1)**
```
Production URLs:
├── Realm 1: https://d3no8h3k2vl9rf.cloudfront.net (29237d86b521e338686006682ddc4531)
├── Realm 2: https://d2lu0ef886dvi8.cloudfront.net (4a0fafeaa56b2f95b8ccaf1c4b329af2)
└── Realm 3: https://d803p7xyrjs61.cloudfront.net (d0c3289214f4e76f9cc3474f963a3fb5)

Infrastructure:
├── 3x Elastic Beanstalk Environments (Go 1.24)
├── 3x CloudFront Distributions
├── 3x DynamoDB Tables (auto-created)
├── 1x Secrets Manager Secret
├── 1x VPC with public/private subnets
└── IAM roles and security groups
```

---

## 🚀 **DEPLOYING TO us-east-2 (Production)**

### **Environment Strategy**
- **us-east-1**: Development/Staging environment
- **us-east-2**: Production environment
- **Same Configuration**: Identical infrastructure in both regions
- **Different Realm IDs**: Generate new realm IDs for production

---

## 📋 **STEP-BY-STEP DEPLOYMENT INSTRUCTIONS**

### **Step 1: Prepare New Region Directory**
```bash
# Navigate to project root
cd /Users/adamdahan/Developer/iheartsolana/jelli/juicebox-software-realm

# Create new AWS directory for us-east-2
cp -r aws/ aws-production/
cd aws-production/
```

### **Step 2: Generate New Realm IDs**
Production must use different realm IDs than development.

```bash
# Generate 3 new realm IDs (32 hex characters each)
python3 -c "import secrets; print('Realm 1:', secrets.token_hex(16))"
python3 -c "import secrets; print('Realm 2:', secrets.token_hex(16))"  
python3 -c "import secrets; print('Realm 3:', secrets.token_hex(16))"
```

**Example Output:**
```
Realm 1: a1b2c3d4e5f6789012345678901234ab
Realm 2: b2c3d4e5f6789012345678901234abc1  
Realm 3: c3d4e5f6789012345678901234abc12b
```

### **Step 3: Update Terraform Configuration**

**Edit `terraform.tfvars`:**
```hcl
# Update for us-east-2 production
realm_id = "a1b2c3d4e5f6789012345678901234ab"  # Use first generated ID
tenant_secrets = {
  "demo" = {
    "1" = "5077a1fd9dfbd60ed0c765ca114f67508e65a1850d3900199efc8a5f3de62c15"
  }
}
```

**Edit `variables.tf`:**
```hcl
# Add default region
variable "aws_region" {
  description = "AWS region for deployment"
  type        = string
  default     = "us-east-2"  # Change from us-east-1
}
```

**Edit `main.tf`:**
```hcl
# Update provider region
provider "aws" {
  region = var.aws_region  # Make sure this references the variable
}

# Update environment names to avoid conflicts
resource "aws_elastic_beanstalk_environment" "realm_1" {
  name                = "jb-sw-realm-prod-1"  # Add 'prod' prefix
  application         = aws_elastic_beanstalk_application.juicebox_realm.name
  solution_stack_name = "64bit Amazon Linux 2023 v4.3.0 running Go 1.21"
  # ... rest of configuration
}

resource "aws_elastic_beanstalk_environment" "realm_2" {
  name                = "jb-sw-realm-prod-2"  # Add 'prod' prefix
  # ... configuration with realm ID: b2c3d4e5f6789012345678901234abc1
}

resource "aws_elastic_beanstalk_environment" "realm_3" {
  name                = "jb-sw-realm-prod-3"  # Add 'prod' prefix  
  # ... configuration with realm ID: c3d4e5f6789012345678901234abc12b
}
```

### **Step 4: Update CloudFront Distribution Configuration**

**Edit `distribution-config.json`:**
```json
{
  "CallerReference": "juicebox-production-2025",
  "Comment": "Juicebox Software Realm Production Distribution",
  "DefaultRootObject": "",
  "Origins": {
    "Quantity": 1,
    "Items": [
      {
        "Id": "EB-jb-sw-realm-prod-1",
        "DomainName": "jb-sw-realm-prod-1.us-east-2.elasticbeanstalk.com",
        "CustomOriginConfig": {
          "HTTPPort": 80,
          "HTTPSPort": 443,
          "OriginProtocolPolicy": "http-only"
        }
      }
    ]
  }
}
```

### **Step 5: Deploy Infrastructure**
```bash
# Set AWS profile and region
export AWS_PROFILE=jelli-wallet
export AWS_DEFAULT_REGION=us-east-2

# Initialize Terraform for new region
terraform init

# Review planned changes
terraform plan

# Deploy infrastructure
terraform apply
# Type 'yes' when prompted
```

### **Step 6: Verify Deployment**
```bash
# Test each realm (URLs will be different)
curl https://[new-cloudfront-url-1].cloudfront.net/
curl https://[new-cloudfront-url-2].cloudfront.net/  
curl https://[new-cloudfront-url-3].cloudfront.net/

# Expected responses with NEW realm IDs:
# {"realmID":"a1b2c3d4e5f6789012345678901234ab"}
# {"realmID":"b2c3d4e5f6789012345678901234abc1"}
# {"realmID":"c3d4e5f6789012345678901234abc12b"}
```

### **Step 7: Update SDK Configuration for Production**

**Create Production Configuration in Jelli Wallet SDK:**
```javascript
// In JelliConfig.js, add environment-based configuration
getInternalJuiceboxConfig() {
  const environment = this.appInfo?.environment || 'development';
  
  if (environment === 'production') {
    return {
      realms: [
        {
          address: 'https://[new-cloudfront-url-1].cloudfront.net',
          id: 'a1b2c3d4e5f6789012345678901234ab',
        },
        {
          address: 'https://[new-cloudfront-url-2].cloudfront.net', 
          id: 'b2c3d4e5f6789012345678901234abc1',
        },
        {
          address: 'https://[new-cloudfront-url-3].cloudfront.net',
          id: 'c3d4e5f6789012345678901234abc12b',
        }
      ],
      register_threshold: 2,
      recover_threshold: 2,
      pin_hashing_mode: 'Standard2019',
      backendUrl: 'https://jelli-juicebox-backend-prod-[hash].herokuapp.com'
    };
  }
  
  // Return development configuration for non-production
  return {
    realms: [
      // ... existing development URLs
    ]
  };
}
```

---

## 🔧 **CONFIGURATION DIFFERENCES**

### **Development (us-east-1)**
- **Purpose**: Development and testing
- **Environment Names**: `jb-sw-realm-1`, `jb-sw-realm-2`, `jb-sw-realm-3`
- **Realm IDs**: Current development IDs
- **SDK Environment**: `development` or `live`

### **Production (us-east-2)**  
- **Purpose**: Production workloads
- **Environment Names**: `jb-sw-realm-prod-1`, `jb-sw-realm-prod-2`, `jb-sw-realm-prod-3`
- **Realm IDs**: New production-specific IDs
- **SDK Environment**: `production`

---

## 🛡️ **SECURITY CONSIDERATIONS**

### **Realm ID Isolation**
- **Critical**: Production and development MUST use different realm IDs
- **Why**: Prevents accidental cross-environment data access
- **Validation**: Each realm ID must be unique across all environments

### **Secrets Management**
- **Development**: Can use demo/test tenant secrets
- **Production**: MUST use production tenant secrets
- **Storage**: Both stored in respective region's Secrets Manager

### **Network Isolation**
- **VPCs**: Each region gets its own VPC
- **Security Groups**: Isolated per environment
- **IAM Roles**: Region-specific permissions

---

## 📊 **MONITORING & MANAGEMENT**

### **CloudWatch Dashboards**
Create separate dashboards for each environment:
```bash
# Development dashboard
aws cloudwatch put-dashboard --dashboard-name "Juicebox-Development-us-east-1"

# Production dashboard  
aws cloudwatch put-dashboard --dashboard-name "Juicebox-Production-us-east-2"
```

### **Cost Management**
```bash
# Tag resources for cost tracking
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list arn:aws:elasticbeanstalk:us-east-2:*:environment/* \
  --tags Environment=Production,Project=Juicebox
```

---

## 🚧 **DEPLOYMENT CHECKLIST**

### **Pre-Deployment**
- [ ] Generate 3 new realm IDs for production
- [ ] Update terraform.tfvars with new realm IDs
- [ ] Update main.tf with prod environment names
- [ ] Update distribution-config.json
- [ ] Set AWS_PROFILE=jelli-wallet
- [ ] Set AWS_DEFAULT_REGION=us-east-2

### **Deployment**
- [ ] Run `terraform init` in aws-production/
- [ ] Run `terraform plan` and review changes
- [ ] Run `terraform apply` and confirm
- [ ] Wait for all resources to be created (~10-15 minutes)

### **Post-Deployment Validation**
- [ ] All 3 CloudFront distributions show "Deployed" status
- [ ] All 3 EB environments show "Green" health
- [ ] All 3 realms respond with correct realm IDs via HTTPS
- [ ] DynamoDB tables created in us-east-2
- [ ] Secrets Manager secrets accessible

### **SDK Integration**
- [ ] Update Jelli Wallet SDK with production URLs
- [ ] Test registration flow in production
- [ ] Test recovery flow in production
- [ ] Verify multi-chain account generation

---

## 💡 **QUICK DEPLOYMENT SUMMARY**

**For someone familiar with the setup, the deployment to us-east-2 can be done in ~30 minutes:**

1. **Copy & Configure** (5 min): Copy aws/ to aws-production/, update realm IDs and region
2. **Deploy Infrastructure** (15 min): `terraform apply` and wait for completion  
3. **Verify & Test** (10 min): Test all endpoints and update SDK configuration

**The existing documentation in AWS_PRODUCTION_DEPLOYMENT.md covers all the technical details. This guide focuses specifically on the multi-region replication process.**

---

## 🔄 **MAINTENANCE ACROSS REGIONS**

### **Updates and Changes**
- **Code Updates**: Deploy to development first, then production
- **Infrastructure Changes**: Test in us-east-1, then replicate to us-east-2
- **Configuration Updates**: Maintain separate terraform.tfvars for each region

### **Monitoring Both Regions**
```bash
# Check all environments across regions
aws elasticbeanstalk describe-environments --region us-east-1
aws elasticbeanstalk describe-environments --region us-east-2
```

---

**🎯 Bottom Line: You now have a complete blueprint to replicate your successful us-east-1 development deployment to us-east-2 for production use, with proper environment isolation and security.**

---

*Multi-Region Deployment Guide*  
*Created: September 22, 2025*  
*Compatible with: Current AWS production infrastructure*
