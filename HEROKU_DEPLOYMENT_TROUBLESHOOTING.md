# Heroku Juicebox Realm Deployment Troubleshooting Guide

## Overview

This document provides a comprehensive technical specification of all issues encountered during the deployment and configuration of Juicebox Software Realms on Heroku, along with their detailed solutions. This guide serves as both a troubleshooting reference and a deployment checklist for future Juicebox realm deployments.

## Architecture Context

The Jelli wallet application uses a distributed Juicebox setup consisting of:
- **3 Heroku-deployed software realms** (jelli-realm-1, jelli-realm-2, jelli-realm-3)
- **1 Heroku backend service** (jelli-juicebox-backend-dev) for JWT creation
- **React Native mobile app** that connects to realms for wallet backup/recovery
- **2-of-3 threshold configuration** (requires 2 realms for registration/recovery)

## Issues Encountered and Solutions

### Issue #1: Procfile Binary Path Mismatch

#### Problem Description
**Error**: `Process exited with status 127` and `/bin/bash: line 1: ./jb-sw-realm: No such file or directory`

**Root Cause**: The Procfile was referencing the wrong binary path. Heroku's Go buildpack compiles binaries to `./bin/` directory, but the Procfile was pointing to the root directory.

#### Original Configuration
```
# Procfile (INCORRECT)
web: ./jb-sw-realm -id $REALM_ID -provider memory
```

#### Solution
```
# Procfile (CORRECT)
web: ./bin/jb-sw-realm -id $REALM_ID -provider memory
```

#### Technical Details
- Heroku Go buildpack automatically detects Go modules and compiles binaries
- Build output shows: `Installed the following binaries: ./bin/jb-sw-realm`
- The Procfile must match the actual binary location post-build

#### Verification
```bash
heroku ps -a jelli-realm-1
# Should show: web.1: up (not crashed)
```

---

### Issue #2: Tenant Secrets Format Inconsistency

#### Problem Description
**Error**: `tenant names must be alphanumeric, exiting...`

**Root Cause**: Mismatch between tenant secret formats across different environments:
- Local development used: `"juiceboxdemo"`
- Heroku realms initially configured with: `"jb-sw-tenant-demo"`
- Backend JWT creation used: `"demo"`

#### Investigation Process
1. **Local Setup Analysis**:
   ```bash
   # start-local-realms.sh
   TENANT_SECRETS='{"juiceboxdemo":{"1":"{\"data\":\"'$TENANT_KEY'\",\"encoding\":\"Hex\",\"algorithm\":\"HmacSha256\"}"}}'
   ```

2. **Backend JWT Creation**:
   ```javascript
   // jelli-juicebox-backend/index.js
   const payload = {
     iss: "demo",           // issuer (tenant name)
     // ...
   };
   ```

3. **Realm Source Code Analysis**:
   ```go
   // secrets/secretsManager.go
   const JuiceboxTenantSecretPrefix string = "jb-sw-tenant-"
   ```

#### Solution
Standardized all environments to use `"demo"` as the tenant name:

```bash
# Heroku realm configuration (CORRECT)
heroku config:set TENANT_SECRETS='{"demo":{"1":"5077a1fd9dfbd60ed0c765ca114f67508e65a1850d3900199efc8a5f3de62c15"}}' -a jelli-realm-1
heroku config:set TENANT_SECRETS='{"demo":{"1":"5077a1fd9dfbd60ed0c765ca114f67508e65a1850d3900199efc8a5f3de62c15"}}' -a jelli-realm-2  
heroku config:set TENANT_SECRETS='{"demo":{"1":"5077a1fd9dfbd60ed0c765ca114f67508e65a1850d3900199efc8a5f3de62c15"}}' -a jelli-realm-3
```

#### Key Insights
- The `jb-sw-tenant-` prefix is added internally by the realm software
- Environment variables should contain the raw tenant name without prefix
- All components (backend, realms, mobile app) must use consistent tenant naming

---

### Issue #3: JWT Header Field Specification Error

#### Problem Description
**Error**: `jwt missing kid`

**Root Cause**: Confusion between JWT specification documentation and actual realm implementation requirements.

#### Investigation Process
1. **Documentation Review**: Official Juicebox SDK docs showed `keyid` field
2. **Source Code Analysis**: Realm implementation actually expects `kid` field
   ```go
   // secrets/secretsManager.go:97
   kid, ok := token.Header["kid"]
   if !ok {
       return nil, nil, errors.New("jwt missing kid")
   }
   ```

#### Initial Incorrect Fix
```javascript
// INCORRECT - Based on documentation
header: {
  keyid: 'demo:1',
  typ: 'JWT',
  alg: 'HS256'
}
```

#### Final Solution
```javascript
// CORRECT - Based on actual implementation
header: {
  kid: 'demo:1',       // Realm expects 'kid' not 'keyid'
  typ: 'JWT',
  alg: 'HS256'
}
```

#### Lesson Learned
Always verify implementation details against source code, not just documentation. The realm software uses `kid` despite some documentation suggesting `keyid`.

---

### Issue #4: JWT Signing Key Encoding Mismatch

#### Problem Description
**Error**: `token signature is invalid: signature is invalid`

**Root Cause**: Incorrect conversion of hex string signing key to bytes for JWT signing.

#### Investigation Process
1. **Key Verification**: Confirmed both backend and realms use same hex key:
   ```
   5077a1fd9dfbd60ed0c765ca114f67508e65a1850d3900199efc8a5f3de62c15
   ```

2. **Encoding Analysis**: Backend was converting hex to buffer, but realm expected raw hex string

#### Original Incorrect Implementation
```javascript
// INCORRECT - Converting hex to buffer
const token = jwt.sign(
  payload,
  Buffer.from(JWT_SECRET, 'hex'), // This conversion was wrong
  options
);
```

#### Final Solution
```javascript
// CORRECT - Use raw hex string
const token = jwt.sign(
  payload,
  JWT_SECRET, // Raw hex string as realm expects
  options
);
```

#### Technical Details
- The realm's memory provider stores tenant secrets as raw hex strings
- JWT signing must use the same format the realm uses for verification
- Different providers (GCP, AWS, MongoDB) may handle key encoding differently

---

### Issue #5: Team vs Personal Heroku App Access

#### Problem Description
**Error**: Apps not found or permission denied when accessing realm apps

**Root Cause**: Realm apps were deployed under the `jelli-wallet` team, not personal account.

#### Solution
```bash
# Correct way to access team apps
heroku config -a jelli-realm-1  # Works for team apps
heroku ps -a jelli-realm-1      # Works for team apps

# List team apps
heroku apps --team jelli-wallet
```

#### Git Remote Configuration
```bash
# Add team app remotes
git remote add jelli-realm-1-team https://git.heroku.com/jelli-realm-1.git
git remote add jelli-realm-2-team https://git.heroku.com/jelli-realm-2.git  
git remote add jelli-realm-3-team https://git.heroku.com/jelli-realm-3.git
```

---

## Deployment Checklist

### Pre-Deployment Verification

1. **Binary Path Verification**:
   ```bash
   # Ensure Procfile uses correct path
   cat Procfile
   # Should show: web: ./bin/jb-sw-realm -id $REALM_ID -provider memory
   ```

2. **Tenant Secret Consistency**:
   ```bash
   # Verify all realms use same tenant name and key
   heroku config -a jelli-realm-1 | grep TENANT_SECRETS
   heroku config -a jelli-realm-2 | grep TENANT_SECRETS  
   heroku config -a jelli-realm-3 | grep TENANT_SECRETS
   ```

3. **Backend JWT Configuration**:
   ```bash
   # Verify backend uses same key and tenant name
   heroku config -a jelli-juicebox-backend-dev | grep JWT_SECRET
   ```

### Deployment Process

1. **Commit Changes**:
   ```bash
   git add Procfile
   git commit -m "Fix Procfile binary path for Heroku deployment"
   ```

2. **Deploy to All Realms**:
   ```bash
   git push jelli-realm-1-team HEAD:main
   git push jelli-realm-2-team HEAD:main
   git push jelli-realm-3-team HEAD:main
   ```

3. **Verify Deployment**:
   ```bash
   heroku ps -a jelli-realm-1
   heroku ps -a jelli-realm-2
   heroku ps -a jelli-realm-3
   # All should show: web.1: up
   ```

### Post-Deployment Testing

1. **Basic Connectivity**:
   ```bash
   curl https://jelli-realm-1-e26248460bc4.herokuapp.com/
   curl https://jelli-realm-2-0d60fb8d3661.herokuapp.com/
   curl https://jelli-realm-3-4fb8ec753624.herokuapp.com/
   # Should return: {"realmID":"<realm-id>"}
   ```

2. **JWT Authentication**:
   ```bash
   # Create JWT
   JWT_TOKEN=$(curl -s -X POST https://jelli-juicebox-backend-dev-b0b281f49955.herokuapp.com/create-jwt \
     -H "Content-Type: application/json" \
     -d '{"email":"test@example.com","realmId":"29237d86b521e338686006682ddc4531"}')
   
   # Test authentication
   curl -X POST https://jelli-realm-1-e26248460bc4.herokuapp.com/req \
     -H "Authorization: Bearer $JWT_TOKEN" \
     -d "test"
   # Should return: SDK upgrade required (not authentication error)
   ```

## Configuration Reference

### Environment Variables

#### Realm Configuration
```bash
# Each realm needs these environment variables
REALM_ID=<16-byte-hex-string>
TENANT_SECRETS={"demo":{"1":"<64-char-hex-key>"}}
```

#### Backend Configuration  
```bash
# Backend needs matching JWT secret
JWT_SECRET=<same-64-char-hex-key-as-realms>
CORS_ORIGIN=*
NODE_ENV=development
```

### Mobile App Configuration

The mobile app (`JuiceboxService.js`) should be configured with:

```javascript
this.configuration = {
  realms: [
    {
      address: 'https://jelli-realm-1-e26248460bc4.herokuapp.com',
      id: '29237d86b521e338686006682ddc4531',
    },
    {
      address: 'https://jelli-realm-2-0d60fb8d3661.herokuapp.com', 
      id: '4bb8c14640f1883a97d887a90a708beb',
    },
    {
      address: 'https://jelli-realm-3-4fb8ec753624.herokuapp.com',
      id: 'cf4b65d4c56872825da6155654423e98',
    },
  ],
  register_threshold: 2,
  recover_threshold: 2,
  pin_hashing_mode: PinHashingMode.Standard2019,
};
```

## Troubleshooting Commands

### Realm Status Check
```bash
# Check if realms are running
heroku ps -a jelli-realm-1
heroku ps -a jelli-realm-2  
heroku ps -a jelli-realm-3
```

### Log Analysis
```bash
# Check recent logs for errors
heroku logs --num=20 -a jelli-realm-1
heroku logs --tail -a jelli-realm-1  # Real-time logs
```

### Configuration Verification
```bash
# Verify environment variables
heroku config -a jelli-realm-1
heroku config -a jelli-juicebox-backend-dev
```

### JWT Testing
```bash
# Test JWT creation and authentication
JWT_TOKEN=$(curl -s -X POST https://jelli-juicebox-backend-dev-b0b281f49955.herokuapp.com/create-jwt \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","realmId":"29237d86b521e338686006682ddc4531"}')

echo "JWT Token: $JWT_TOKEN"

curl -X POST https://jelli-realm-1-e26248460bc4.herokuapp.com/req \
  -H "Authorization: Bearer $JWT_TOKEN" \
  -d "test"
```

## Common Error Messages and Solutions

| Error Message | Root Cause | Solution |
|---------------|------------|----------|
| `Process exited with status 127` | Wrong binary path in Procfile | Update Procfile to use `./bin/jb-sw-realm` |
| `tenant names must be alphanumeric` | Incorrect tenant secret format | Use `{"demo":{"1":"hex_key"}}` format |
| `jwt missing kid` | Wrong JWT header field | Use `kid` not `keyid` in JWT header |
| `token signature is invalid` | Wrong signing key encoding | Use raw hex string, not Buffer.from() |
| `invalid or expired jwt` | Multiple possible causes | Check tenant name, key, and JWT format |
| `SDK upgrade required` | **Success!** JWT auth working | This is expected for test requests |

## Security Considerations

1. **Key Management**: 
   - Use the same 64-character hex key across all realms and backend
   - Store keys securely in Heroku config vars (not in code)

2. **JWT Expiration**:
   - Tokens expire in 10 minutes by default
   - Implement proper token refresh in mobile app

3. **Tenant Isolation**:
   - Each tenant has isolated secrets
   - Tenant names must be alphanumeric for security

4. **HTTPS Only**:
   - All realm communication must use HTTPS
   - Heroku provides SSL termination automatically

## Performance Considerations

1. **Memory Provider**:
   - Current setup uses in-memory storage (not persistent)
   - Consider upgrading to MongoDB/DynamoDB for production

2. **Dyno Management**:
   - Basic dynos sleep after 30 minutes of inactivity
   - Consider upgrading to Standard dynos for production

3. **Geographic Distribution**:
   - All realms currently in same region
   - Consider multi-region deployment for better availability

## Future Improvements

1. **Persistent Storage**: Migrate from memory provider to database-backed storage
2. **Monitoring**: Add application monitoring and alerting
3. **Load Balancing**: Implement proper load balancing for high availability
4. **Backup Strategy**: Implement backup and disaster recovery procedures
5. **CI/CD Pipeline**: Automate deployment process with proper testing

---

**Document Version**: 1.0  
**Last Updated**: August 16, 2025  
**Author**: Technical Troubleshooting Session  
**Status**: Production Ready ✅
