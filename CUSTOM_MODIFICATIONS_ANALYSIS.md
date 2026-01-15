# Custom Keycloak Modifications Analysis

**Date:** January 14, 2026  
**Branch:** release/26.2  
**Analysis:** Review of source code modifications and alternative implementation strategies

---

## Overview

This document analyzes modifications made to the Keycloak source code and provides recommendations for achieving the same functionality without modifying core Keycloak files. This approach ensures easier upgrades and better maintainability.

---

## Modified Files

### 1. Backend Changes
- `services/src/main/java/org/keycloak/services/resources/admin/AdminRoot.java`

### 2. Frontend Changes
- `js/apps/admin-ui/src/index.ts`
- `js/apps/admin-ui/src/realm-settings/RealmSettingsTabs.tsx`
- `js/apps/admin-ui/src/realm-settings/realm-settings-section.css`
- `js/apps/admin-ui/src/realm-settings/routes/RealmSettings.tsx`
- `js/apps/admin-ui/src/realm-settings/UserAccount.tsx` (new file)
- `js/pnpm-lock.yaml`

---

## Change 1: IP-Based Admin Access Control

### What Was Changed

**File:** `AdminRoot.java`

Added `isAdminCloaked()` method that:
- Checks the client's remote IP address
- Whitelists specific IPs: `51.136.31.9`, `20.4.5.153`, `10.0.0.4`, `10.0.2.4`
- Blocks all other IPs from accessing admin console and API
- Returns HTTP 404 (Not Found) instead of revealing admin endpoints

**Integration Points:**
- `masterRealmAdminConsoleRedirect()` - Line 100
- `shouldRedirect()` - Line 110
- `masterRealmAdminConsoleRedirectHtml()` - Line 137
- `getAdminConsole()` - Line 173
- `getRealmsAdmin()` - Line 235
- `preFlight()` - Line 259
- `getServerInfo()` - Line 275

### Security Implications

**Pros:**
- ✅ Obscures admin interface from unauthorized networks
- ✅ Additional layer of defense-in-depth
- ✅ Simple IP-based access control

**Cons:**
- ❌ Hardcoded IP addresses in source code
- ❌ Requires recompilation to update whitelist
- ❌ Cannot be configured via environment variables or config files
- ❌ Makes Keycloak upgrades difficult
- ❌ IP spoofing possible if not behind proper proxy
- ❌ No audit logging of blocked attempts (only INFO log)

---

## Change 2: User Account Management Tab

### What Was Changed

**New Component:** `UserAccount.tsx`
- Added new "User Account" tab to Realm Settings
- Currently contains only placeholder UI (empty FormPanel)
- Intended for user account linking functionality

**Modified Files:**
- `RealmSettingsTabs.tsx` - Added tab navigation
- `index.ts` - Exported new component
- Route configuration updated

### Current State

The component is a **placeholder** with no actual functionality:
```tsx
<FormPanel title={t("UserAccountLinking")} className="kc-user-account-template">
  {/* Empty - no implementation */}
</FormPanel>
```

---

## Recommended Alternatives (No Source Modification)

### Option 1: Reverse Proxy IP Filtering (RECOMMENDED)

**Best for:** Production deployments, multi-environment setups

#### nginx Configuration

```nginx
# /etc/nginx/conf.d/keycloak.conf

upstream keycloak {
    server keycloak:8080;
}

server {
    listen 443 ssl http2;
    server_name keycloak.example.com;

    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # IP whitelist
    geo $admin_allowed {
        default 0;
        51.136.31.9 1;
        20.4.5.153 1;
        10.0.0.4 1;
        10.0.2.4 1;
    }

    # Block admin endpoints for unauthorized IPs
    location ~ ^/(admin|realms/master) {
        if ($admin_allowed = 0) {
            return 404;  # Match Keycloak's behavior
        }

        proxy_pass http://keycloak;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # Allow all other traffic
    location / {
        proxy_pass http://keycloak;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

**Benefits:**
- ✅ No source code modification
- ✅ Easy to update IP list (just reload nginx)
- ✅ Works across Keycloak versions
- ✅ Can add rate limiting, WAF, etc.
- ✅ Centralized security policy
- ✅ Better logging and monitoring

#### Docker Compose Example

```yaml
version: '3.8'

services:
  nginx:
    image: nginx:alpine
    ports:
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/conf.d/default.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - keycloak

  keycloak:
    image: quay.io/keycloak/keycloak:26.2.5
    environment:
      KC_PROXY: edge
      KC_PROXY_HEADERS: xforwarded
      KC_HTTP_ENABLED: "true"
      KC_HOSTNAME_STRICT: "false"
    command: start
```

---

### Option 2: Custom Keycloak SPI Extension

**Best for:** Keycloak-native solution, complex logic, dynamic configuration

#### Project Structure

```
admin-access-filter/
├── build.gradle
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── org/
│   │   │       └── example/
│   │   │           ├── AdminAccessFilter.java
│   │   │           └── AdminAccessFilterFactory.java
│   │   └── resources/
│   │       └── META-INF/
│   │           └── services/
│   │               └── org.keycloak.services.resource.RealmResourceProviderFactory
```

#### Implementation

**File:** `AdminAccessFilterFactory.java`

```java
package org.example;

import org.keycloak.Config;
import org.keycloak.models.KeycloakSession;
import org.keycloak.models.KeycloakSessionFactory;
import org.keycloak.services.resource.RealmResourceProvider;
import org.keycloak.services.resource.RealmResourceProviderFactory;

import java.util.Arrays;
import java.util.HashSet;
import java.util.Set;

public class AdminAccessFilterFactory implements RealmResourceProviderFactory {
    
    private static final String PROVIDER_ID = "admin-ip-filter";
    private Set<String> allowedIps = new HashSet<>();

    @Override
    public RealmResourceProvider create(KeycloakSession session) {
        return new AdminAccessFilter(session, allowedIps);
    }

    @Override
    public void init(Config.Scope config) {
        // Load from keycloak.conf: spi-admin-ip-filter-allowed-ips
        String ips = config.get("allowed-ips", 
                     System.getenv("KC_ADMIN_ALLOWED_IPS"));
        
        if (ips != null && !ips.isEmpty()) {
            allowedIps.addAll(Arrays.asList(ips.split(",")));
        } else {
            // Default whitelist
            allowedIps.add("51.136.31.9");
            allowedIps.add("20.4.5.153");
            allowedIps.add("10.0.0.4");
            allowedIps.add("10.0.2.4");
        }
    }

    @Override
    public void postInit(KeycloakSessionFactory factory) {
        // No-op
    }

    @Override
    public void close() {
        // No-op
    }

    @Override
    public String getId() {
        return PROVIDER_ID;
    }
}
```

**File:** `AdminAccessFilter.java`

```java
package org.example;

import jakarta.ws.rs.GET;
import jakarta.ws.rs.Path;
import jakarta.ws.rs.Produces;
import jakarta.ws.rs.core.MediaType;
import jakarta.ws.rs.core.Response;
import org.jboss.logging.Logger;
import org.keycloak.models.KeycloakSession;
import org.keycloak.services.resource.RealmResourceProvider;

import java.util.Set;

public class AdminAccessFilter implements RealmResourceProvider {
    
    private static final Logger logger = Logger.getLogger(AdminAccessFilter.class);
    private final KeycloakSession session;
    private final Set<String> allowedIps;

    public AdminAccessFilter(KeycloakSession session, Set<String> allowedIps) {
        this.session = session;
        this.allowedIps = allowedIps;
    }

    @Override
    public Object getResource() {
        return this;
    }

    @Override
    public void close() {
        // No-op
    }

    @GET
    @Path("check-access")
    @Produces(MediaType.APPLICATION_JSON)
    public Response checkAccess() {
        String remoteAddr = session.getContext().getConnection().getRemoteAddr();
        
        if (!allowedIps.contains(remoteAddr)) {
            logger.warnf("Unauthorized admin access attempt from IP: %s", remoteAddr);
            return Response.status(Response.Status.FORBIDDEN)
                    .entity("{\"allowed\": false}")
                    .build();
        }
        
        return Response.ok("{\"allowed\": true}").build();
    }
}
```

**File:** `build.gradle`

```gradle
plugins {
    id 'java'
}

group = 'org.example'
version = '1.0.0'
java.sourceCompatibility = JavaVersion.VERSION_21

ext {
    keycloakVersion = '26.2.5'
}

repositories {
    mavenCentral()
}

dependencies {
    compileOnly "org.keycloak:keycloak-core:$keycloakVersion"
    compileOnly "org.keycloak:keycloak-server-spi:$keycloakVersion"
    compileOnly "org.keycloak:keycloak-server-spi-private:$keycloakVersion"
    compileOnly "org.keycloak:keycloak-services:$keycloakVersion"
}

jar {
    enabled = true
}
```

#### Configuration

**File:** `keycloak.conf`

```properties
# Configure allowed IPs
spi-admin-ip-filter-allowed-ips=51.136.31.9,20.4.5.153,10.0.0.4,10.0.2.4
```

**Or via Environment Variable:**

```bash
export KC_ADMIN_ALLOWED_IPS="51.136.31.9,20.4.5.153,10.0.0.4,10.0.2.4"
```

#### Deployment

```bash
# Build
./gradlew build

# Deploy
cp build/libs/admin-access-filter-1.0.0.jar /opt/keycloak/providers/

# Rebuild Keycloak
kc.sh build

# Restart
kc.sh start
```

**Limitations:** This approach provides an API endpoint but doesn't intercept admin console requests. You'd need to combine it with a reverse proxy or use Keycloak's event listener SPI to monitor admin access.

---

### Option 3: Firewall Rules (Infrastructure Level)

**Best for:** Cloud deployments, simple requirements

#### AWS Security Group

```json
{
  "IpPermissions": [
    {
      "IpProtocol": "tcp",
      "FromPort": 8443,
      "ToPort": 8443,
      "IpRanges": [
        {"CidrIp": "51.136.31.9/32", "Description": "Office IP 1"},
        {"CidrIp": "20.4.5.153/32", "Description": "Office IP 2"},
        {"CidrIp": "10.0.0.4/32", "Description": "Internal subnet 1"},
        {"CidrIp": "10.0.2.4/32", "Description": "Internal subnet 2"}
      ]
    }
  ]
}
```

#### Linux iptables

```bash
#!/bin/bash
# admin-firewall.sh

# Flush existing rules
iptables -F INPUT

# Default policy: DROP
iptables -P INPUT DROP

# Allow localhost
iptables -A INPUT -i lo -j ACCEPT

# Allow established connections
iptables -A INPUT -m state --state ESTABLISHED,RELATED -j ACCEPT

# Allow admin IPs to Keycloak admin port
iptables -A INPUT -p tcp --dport 8443 -s 51.136.31.9 -j ACCEPT
iptables -A INPUT -p tcp --dport 8443 -s 20.4.5.153 -j ACCEPT
iptables -A INPUT -p tcp --dport 8443 -s 10.0.0.4 -j ACCEPT
iptables -A INPUT -p tcp --dport 8443 -s 10.0.2.4 -j ACCEPT

# Allow public traffic on different port (8080)
iptables -A INPUT -p tcp --dport 8080 -j ACCEPT

# Log blocked admin attempts
iptables -A INPUT -p tcp --dport 8443 -j LOG --log-prefix "ADMIN_BLOCKED: "
iptables -A INPUT -p tcp --dport 8443 -j DROP

# Save rules
iptables-save > /etc/iptables/rules.v4
```

---

## User Account Linking Feature

### Current Implementation

The `UserAccount.tsx` component is currently empty. Based on the context, this appears to be for **federated identity linking** functionality.

### Alternative: Extend Account Console

Instead of modifying Admin UI, extend the **user-facing Account Console**:

#### Custom Account Console Theme

```
themes/
└── custom-account/
    ├── account/
    │   ├── resources/
    │   │   ├── css/
    │   │   │   └── account.css
    │   │   └── js/
    │   │       └── account-linking.js
    │   └── theme.properties
```

**File:** `theme.properties`

```properties
parent=keycloak.v2
styles=css/account.css
scripts=js/account-linking.js
```

#### Custom REST API Endpoint (Similar to Elective Consent Pattern)

Create a custom SPI for account linking management:

```java
// UserAccountLinkingResource.java
public class UserAccountLinkingResource {
    
    @GET
    @Path("linked-accounts")
    @Produces(MediaType.APPLICATION_JSON)
    public Response getLinkedAccounts(@Context KeycloakSession session) {
        UserModel user = session.users().getUserById(
            session.getContext().getRealm(), 
            getUserId(session)
        );
        
        Set<FederatedIdentityModel> federatedIdentities = 
            session.users().getFederatedIdentitiesStream(
                session.getContext().getRealm(), 
                user
            ).collect(Collectors.toSet());
        
        return Response.ok(federatedIdentities).build();
    }
    
    @POST
    @Path("link-account")
    @Consumes(MediaType.APPLICATION_JSON)
    public Response linkAccount(LinkAccountRequest request) {
        // Implementation for account linking
        return Response.ok().build();
    }
}
```

This follows the **same pattern** as your `elective-consent-provider` project.

---

## Migration Strategy

### Phase 1: Immediate (Week 1)
1. ✅ Deploy nginx reverse proxy with IP filtering
2. ✅ Test admin access from whitelisted IPs
3. ✅ Document nginx configuration

### Phase 2: Short-term (Week 2-3)
1. ✅ Build custom SPI for configurable IP management
2. ✅ Add environment variable configuration
3. ✅ Deploy and test SPI extension

### Phase 3: Medium-term (Month 1-2)
1. ✅ Design User Account Linking REST API
2. ✅ Build SPI following elective-consent pattern
3. ✅ Create custom Account Console theme
4. ✅ Test account linking functionality

### Phase 4: Cleanup (Month 2-3)
1. ✅ Revert all source code modifications
2. ✅ Upgrade to latest Keycloak version
3. ✅ Verify all custom extensions still work
4. ✅ Document custom extension architecture

---

## Comparison Matrix

| Approach | Pros | Cons | Maintenance | Upgrade Safety |
|----------|------|------|-------------|----------------|
| **Source Modification** | Simple, direct control | Blocks upgrades, hard to maintain | ❌ High | ❌ Blocks |
| **Reverse Proxy** | No code changes, flexible | External dependency | ✅ Low | ✅ Safe |
| **Custom SPI** | Keycloak-native, configurable | More development effort | ⚠️ Medium | ✅ Safe |
| **Firewall Rules** | Infrastructure-level, simple | Less granular control | ✅ Low | ✅ Safe |

---

## Recommendations

### For IP-Based Admin Access Control

**Primary Solution:** Nginx Reverse Proxy
- Easiest to implement and maintain
- No Keycloak modifications
- Upgrade-safe
- Can be updated without Keycloak restart

**Backup Solution:** Custom SPI + Nginx
- SPI provides API for access checks
- Nginx enforces at edge
- Configurable via environment variables

### For User Account Linking

**Recommended Approach:** Custom SPI Extension
- Follow the pattern from `elective-consent-provider`
- Create REST endpoints for account management
- Extend Account Console (not Admin Console)
- Keep separate from Keycloak source

---

## Implementation Priority

1. **HIGH**: Nginx IP filtering (immediate security improvement)
2. **MEDIUM**: Custom SPI for admin access (better configurability)
3. **MEDIUM**: User Account Linking API (feature development)
4. **LOW**: Source code revert (after alternatives are proven)

---

## Conclusion

The current source modifications achieve the desired functionality but create significant maintenance burden. The recommended alternatives provide:

- ✅ **Upgrade safety** - No source modifications
- ✅ **Flexibility** - Configuration via environment/files
- ✅ **Maintainability** - Standard extension patterns
- ✅ **Security** - Defense-in-depth with multiple layers

**Next Steps:**
1. Implement nginx configuration (1-2 hours)
2. Test admin access restrictions (1 hour)
3. Plan custom SPI development (1 week)
4. Design User Account Linking feature (2 weeks)

---

**Document Version:** 1.0  
**Last Updated:** January 14, 2026  
**Author:** GitHub Copilot Analysis
