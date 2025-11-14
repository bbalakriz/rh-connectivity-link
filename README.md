# MEPS Connectivity Link - API Security with Kuadrant

This project demonstrates how to secure API endpoints on OpenShift using Red Hat Connectivity Link (based on Kuadrant), progressing from basic static API key authentication to enterprise-grade Keycloak-based authentication and role-based authorization (RBAC).

## Overview

The project uses a sample "toystore" API application to demonstrate three progressive security configurations:

1. **Basic Setup (Files 00-02)**: Deploy the API with static API key authentication
2. **Keycloak Authentication (File 03)**: Upgrade to JWT-based authentication via Keycloak
3. **RBAC Authorization (File 04)**: Add role-based access control using OPA policies

## Architecture Components

- **Service Mesh**: OpenShift Service Mesh v3 (Istio)
- **API Gateway**: Kubernetes Gateway API with Istio implementation
- **Security/Policy Management**: Red Hat Connectivity Link (Kuadrant)
- **Identity Provider**: Red Hat Build of Keycloak (RHBK)
- **Sample Application**: Toystore API (Kuadrant example talker-api)

## Prerequisites

- OpenShift 4.x cluster with cluster-admin access
- `oc` CLI tool configured and logged in
- Access to Red Hat Operator Hub
- A wildcard domain for your OpenShift cluster (e.g., `*.apps.cluster-xxxxx.opentlc.com`)

## Important: Secrets Management

**⚠️ This repository contains placeholder values for secrets.** Before deploying:

1. **Never commit real secrets to Git**
2. Generate secure, random values for:
   - API keys in `02-connectivity-link.yaml`
   - Keycloak client secrets (if modifying the realm export)
3. Consider using external secret management:
   - OpenShift Secrets
   - External Secrets Operator
   - HashiCorp Vault
   - Sealed Secrets

**For this demo:**
- All secret values have been replaced with placeholders (`YOUR_API_KEY_HERE`, etc.)
- Follow the instructions in each phase to generate and configure secrets
- Store your generated secrets securely (password manager, vault, etc.)

**Recommended Workflow for Local Development:**
1. Copy configuration files with placeholders: `cp 02-connectivity-link.yaml 02-connectivity-link-local.yaml`
2. Update the `-local.yaml` file with your real secrets
3. Apply the local file: `oc apply -f 02-connectivity-link-local.yaml`
4. The `.gitignore` file prevents committing `*-local.yaml` files

## Project Structure

```
.
├── 00-toystore.yaml                    # Sample application deployment
├── 01-gateway-crds.yaml                # Gateway API CRDs
├── 02-connectivity-link.yaml           # Core setup with API key auth (contains placeholders)
├── 03-keycloak-authentication.yaml     # Keycloak-based JWT auth
├── 04-keycloak-rbac-authorization.yaml # Role-based authorization
├── kuadrant-realm-export.json          # Keycloak realm configuration (placeholders for secrets)
└── .gitignore                          # Prevents committing local secret files
```

---

## Installation Guide

**📝 Note**: All YAML files include the necessary namespace/project creation resources. You don't need to manually create namespaces with `oc new-project` commands - they will be created automatically when you apply the files.

### Phase 1: Basic Setup with API Key Authentication

This phase installs all core operators and secures the toystore API with a static API key.

#### Step 1: Deploy the Sample Application

```bash
oc apply -f 00-toystore.yaml
```

This creates:
- **Project/Namespace**: `toystore`
- **Deployment**: Running the toystore application (talker-api)
- **Service**: Exposing the application on port 80

**Note**: The YAML file includes the namespace/project creation, so you don't need to create it manually.

#### Step 2: Install Gateway API CRDs

```bash
oc apply -f 01-gateway-crds.yaml
```

This installs the Kubernetes Gateway API Custom Resource Definitions:
- **GatewayClass**: Defines classes of Gateways
- **Gateway**: Defines how traffic should be exposed
- **HTTPRoute**: Defines HTTP routing rules
- **ReferenceGrant**: Allows cross-namespace references

#### Step 3: Deploy Core Infrastructure and API Security

**⚠️ IMPORTANT**: Before applying this file, you need to update two things in `02-connectivity-link.yaml`:

1. **Update the hostname** to match your OpenShift cluster's domain:
   - Find and replace all instances of `api.apps.cluster-nnnnn.nnnnn.sandboxBBB.nnnnnn.com` with your cluster's wildcard domain
   - Lines to update: 108, 191, 202

```bash
# Get your cluster's default domain
oc get ingresses.config.openshift.io cluster -o jsonpath='{.spec.domain}'
```

2. **Update the API key secret** with your own secure value:
   - Generate a secure API key: `openssl rand -base64 32`
   - Base64 encode it: `echo -n "your-api-key" | base64`
   - Replace the placeholder on line 155 with your base64-encoded key

```bash
# Example: Generate and encode a secure API key
API_KEY=$(openssl rand -base64 32)
API_KEY_BASE64=$(echo -n "${API_KEY}" | base64)
echo "Your API Key: ${API_KEY}"
echo "Base64 Encoded: ${API_KEY_BASE64}"

# Update line 144 in 02-connectivity-link.yaml with the base64 value
# Save the API_KEY value - you'll need it for testing!
```

Then apply:

```bash
oc apply -f 02-connectivity-link.yaml
```

This comprehensive file deploys:

**Namespaces/Projects:**
- `istio-system` - Istio control plane
- `istio-cni` - Istio CNI plugin
- `kuadrant-system` - Kuadrant/Connectivity Link
- `api-gateway` - API Gateway resources

**Operators:**
- OpenShift Service Mesh Operator v3
- Red Hat Connectivity Link Operator

**Service Mesh:**
- Istio control plane with InPlace update strategy
- Istio CNI for transparent pod networking

**Kuadrant:**
- Kuadrant instance with mTLS disabled

**API Gateway:**
- Gateway resource with HTTP listener
- HTTPRoute for toystore API (`/toy` endpoint)
- OpenShift Route for external access (TLS edge termination)
- DestinationRule to disable mTLS for toystore namespace

**Security:**
- AuthPolicy with API key authentication (selector-based)
- API key secret (placeholder - needs to be updated)

#### Step 4: Verify the Installation

Wait for all operators to be ready:

```bash
# Check operator status
oc get csv -n openshift-operators
oc get csv -n kuadrant-system

# Check Istio installation
oc get istio -n istio-system
oc get istiocni -n istio-cni

# Check Kuadrant
oc get kuadrant -n kuadrant-system

# Check Gateway
oc get gateway -n api-gateway
oc get httproute -n toystore
```

#### Step 5: Test API Key Authentication

Get the API endpoint:

```bash
GATEWAY_URL=$(oc get route api-gateway-route -n api-gateway -o jsonpath='{.spec.host}')
echo "API URL: https://${GATEWAY_URL}/toy"
```

Test without authentication (should fail):

```bash
curl -k https://${GATEWAY_URL}/toy
```

Test with API key (should succeed):

```bash
# First, create your own API key secret (replace YOUR_API_KEY_HERE with your desired key)
API_KEY="YOUR_API_KEY_HERE"
API_KEY_BASE64=$(echo -n "${API_KEY}" | base64)

# Update the secret with your API key
oc patch secret api-key-regular-user -n kuadrant-system \
  -p "{\"data\":{\"api_key\":\"${API_KEY_BASE64}\"}}"

# Test with your API key
curl -k -H "Authorization: APIKEY ${API_KEY}" https://${GATEWAY_URL}/toy
```

---

### Phase 2: Upgrade to Keycloak Authentication

This phase replaces static API keys with JWT-based authentication using Keycloak.

#### Step 1: Update Hostnames

**⚠️ IMPORTANT**: Update the Keycloak hostname in `03-keycloak-authentication.yaml`:

Find and replace `sso.apps.cluster-nnnnn.nnnnn.sandboxBBB.nnnnnn.com` with your cluster's domain (lines 37, 64, 84).

#### Step 2: Deploy Keycloak

```bash
oc apply -f 03-keycloak-authentication.yaml
```

This deploys:
- **Project/Namespace**: `keycloak`
- **Operator**: Red Hat Build of Keycloak Operator (RHBK)
- **Keycloak Instance**: Single instance with HTTP enabled
- **OpenShift Route**: External access to Keycloak with edge TLS termination
- **Updated AuthPolicy**: Replaces API key auth with JWT validation from Keycloak

**Note**: The YAML file includes the namespace/project creation automatically.

#### Step 3: Wait for Keycloak to be Ready

```bash
# Watch Keycloak status
oc get keycloak -n keycloak -w

# Once ready, get admin credentials
oc get secret keycloak-initial-admin -n keycloak -o jsonpath='{.data.username}' | base64 -d && echo
oc get secret keycloak-initial-admin -n keycloak -o jsonpath='{.data.password}' | base64 -d && echo
```

#### Step 4: Import Keycloak Realm Configuration

Access Keycloak admin console:

```bash
SSO_URL=$(oc get route keycloak-ingress -n keycloak -o jsonpath='{.spec.host}')
echo "Keycloak Admin Console: https://${SSO_URL}"
```

1. Log in to Keycloak admin console
2. Create a new realm or select existing
3. Go to **Realm Settings** → **Action** → **Partial Import**
4. Upload `kuadrant-realm-export.json`
5. Select all components to import and click **Import**

This imports:
- Realm: `kuadrant`
- Client: `toystore` (with client secret - note: you may want to regenerate this in production)
- Service Account: `service-account-toystore` with `admin` role
- Roles: `admin` role for authorization

**⚠️ IMPORTANT - Get/Generate the Client Secret:**

The realm export file contains a placeholder for the client secret. After importing, you MUST:

1. Log in to Keycloak admin console
2. Select the **kuadrant** realm
3. Navigate to **Clients** → **toystore** → **Credentials** tab
4. Click **Regenerate** to create a new secure client secret
5. Copy the generated secret value (you'll need this for testing)
6. Store it securely - you won't be able to see it again without regenerating

**Note**: The secret is only displayed once. If you lose it, regenerate a new one.

#### Step 5: Test JWT Authentication

Get an access token using client credentials:

```bash
SSO_URL=$(oc get route keycloak-ingress -n keycloak -o jsonpath='{.spec.host}')
GATEWAY_URL=$(oc get route api-gateway-route -n api-gateway -o jsonpath='{.spec.host}')

# Get the client secret from Keycloak UI
# Navigate to: Clients → toystore → Credentials tab → Copy the Client Secret value
CLIENT_SECRET="YOUR_CLIENT_SECRET_HERE"

# Get JWT token
TOKEN=$(curl -k -X POST "https://${SSO_URL}/realms/kuadrant/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=toystore" \
  -d "client_secret=${CLIENT_SECRET}" | jq -r '.access_token')

# Verify token was obtained
echo "Token obtained: ${TOKEN:0:50}..."

# Test with JWT token
curl -k -H "Authorization: Bearer ${TOKEN}" https://${GATEWAY_URL}/toy
```

---

### Phase 3: Add Role-Based Authorization (RBAC)

This phase adds fine-grained authorization using OPA (Open Policy Agent) policies.

#### Step 1: Update Hostnames

**⚠️ IMPORTANT**: Update the hostname in `04-keycloak-rbac-authorization.yaml`:

Find and replace the API hostname (line 35, 63) with your cluster's domain.

#### Step 2: Apply RBAC Configuration

```bash
oc apply -f 04-keycloak-rbac-authorization.yaml
```

This configuration:
- Adds a second AuthPolicy (`toystore-admins`) with OPA-based authorization
- Requires users to have the `admin` role in their JWT token
- Updates the HTTPRoute with multiple rules (one for `/cars`, one for `/toy`)
- Adds response filters to inject user identity information into response headers

#### Step 3: Understand the Authorization Policy

The OPA policy in the configuration checks for the `admin` role:

```rego
roles := input.auth.identity.realm_access.roles
allow { roles[_] == "admin" }
```

This policy:
- Extracts roles from the JWT token's `realm_access.roles` claim
- Allows access only if the `admin` role is present

#### Step 4: Test Authorization

The imported realm configuration includes a service account with the `admin` role, so the previous test should continue to work:

```bash
# Get the client secret from Keycloak (Clients → toystore → Credentials tab)
CLIENT_SECRET="YOUR_CLIENT_SECRET_HERE"

# Get token (service account has admin role)
TOKEN=$(curl -k -X POST "https://${SSO_URL}/realms/kuadrant/protocol/openid-connect/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "grant_type=client_credentials" \
  -d "client_id=toystore" \
  -d "client_secret=${CLIENT_SECRET}" | jq -r '.access_token')

# Access /toy endpoint (requires admin role on rule-2)
curl -k -H "Authorization: Bearer ${TOKEN}" https://${GATEWAY_URL}/toy

# Check response headers for user identity
curl -k -i -H "Authorization: Bearer ${TOKEN}" https://${GATEWAY_URL}/toy | grep -i "x-"
```

To test with a user without admin role, you would need to:
1. Create a new user in Keycloak without the `admin` role
2. Configure the client for password grant or authorization code flow
3. Obtain a token for that user
4. Attempt to access the API (should be denied)

---

## Configuration Details

### API Key Structure

The API key in `02-connectivity-link.yaml`:
- **Secret name**: `api-key-regular-user`
- **Namespace**: `kuadrant-system`
- **Label**: `app: toystore` (used by AuthPolicy selector)
- **Key value**: `YOUR_API_KEY_HERE` (placeholder - replace with your own secret value)
- **Header format**: `Authorization: APIKEY <value>`

**⚠️ Security Note**: The default API key in the YAML file is a placeholder. You should:
1. Generate a secure random API key (e.g., `openssl rand -base64 32`)
2. Update the secret before deploying or patch it after deployment

### Keycloak Client Configuration

From `kuadrant-realm-export.json`:
- **Client ID**: `toystore`
- **Client Secret**: Contains placeholder - MUST be regenerated after import
- **Service Account**: Enabled
- **Service Account User**: `service-account-toystore`
- **Service Account Roles**: `admin`, `default-roles-kuadrant`

**Required Step**: After importing the realm, you MUST generate a client secret:
- Navigate to: **Clients** → **toystore** → **Credentials** tab
- Click **Regenerate** to create a new client secret
- Copy and store the secret securely (it's only shown once)
- Use this secret in all API calls requiring client authentication

### Gateway and Route Configuration

- **Gateway Name**: `external`
- **Gateway Namespace**: `api-gateway`
- **Listener Port**: 80 (HTTP)
- **TLS Termination**: Edge (handled by OpenShift Route)
- **HTTPRoute Namespace**: `toystore`
- **Backend Service**: `toystore:80`

---

## Troubleshooting

### Common Issues

1. **"no matches for kind" error when applying files**
   - Wait for CRDs to be fully installed
   - Check operator installation status

2. **Gateway not ready**
   ```bash
   oc describe gateway external -n api-gateway
   oc logs -n istio-system -l app=istiod
   ```

3. **AuthPolicy not working**
   ```bash
   oc describe authpolicy -n toystore
   oc logs -n kuadrant-system -l app.kubernetes.io/name=authorino
   ```

4. **Keycloak authentication failing**
   - Verify Keycloak is accessible
   - Check realm and client configuration
   - Validate JWT token: https://jwt.io
   - Check issuer URL matches exactly (including https://)

5. **Authorization failing**
   - Verify the user/service account has the required roles
   - Check OPA policy evaluation in Kuadrant logs
   - Ensure the JWT includes the `realm_access.roles` claim

### Useful Commands

```bash
# Check all Kuadrant resources
oc get authpolicy,ratelimitpolicy -A

# View Authorino logs
oc logs -n kuadrant-system -l app.kubernetes.io/name=authorino -f

# View Gateway status
oc get gateway -A -o wide

# Check HTTPRoute status
oc get httproute -A -o yaml

# Describe Keycloak instance
oc describe keycloak keycloak -n keycloak

# Check Istio Gateway pods
oc get pods -n api-gateway
```

---

## Cleanup

To remove all resources in reverse order:

```bash
# Remove RBAC configuration (Phase 3)
oc delete -f 04-keycloak-rbac-authorization.yaml

# Remove Keycloak (Phase 2)
oc delete -f 03-keycloak-authentication.yaml

# Remove core infrastructure (Phase 1)
oc delete -f 02-connectivity-link.yaml

# Remove Gateway CRDs
oc delete -f 01-gateway-crds.yaml

# Remove application
oc delete -f 00-toystore.yaml

# Wait for operators to clean up their resources, then delete projects
# Note: Some operators may take time to finalize resource deletion
oc delete project keycloak toystore api-gateway kuadrant-system istio-system istio-cni
```

**Note**: Deleting projects will also delete all resources within them. The `oc delete -f` commands above clean up the resources gracefully before deleting the projects.

---

## Next Steps

After completing this setup, you can:

1. **Add Rate Limiting**: Use RateLimitPolicy CRs to protect your APIs from abuse
2. **Add TLS**: Configure TLS certificates for the Gateway
3. **Create Custom Roles**: Define more granular RBAC policies in Keycloak
4. **Multi-tenancy**: Use different realms or clients for different applications
5. **Observability**: Integrate with OpenShift Service Mesh observability features
6. **Production Hardening**: 
   - Use external databases for Keycloak
   - Enable mTLS in the service mesh
   - Implement proper secret management
   - Configure high availability for all components

---

## Additional Resources

- [Red Hat Connectivity Link Documentation](https://docs.redhat.com/en/documentation/red_hat_connectivity_link/)
- [Kuadrant Documentation](https://docs.kuadrant.io/)
- [Red Hat Build of Keycloak Documentation](https://access.redhat.com/documentation/en-us/red_hat_build_of_keycloak)
- [Kubernetes Gateway API](https://gateway-api.sigs.k8s.io/)
- [OpenShift Service Mesh Documentation](https://docs.openshift.com/container-platform/latest/service_mesh/v2x/ossm-about.html)

---

## Questions or Issues?

If you encounter any issues or have questions about specific configurations, please refer to:
- The operator logs for detailed error messages
- The official documentation links above
- Red Hat Support (if you have a subscription)

## License

This is a demonstration project. Refer to individual component licenses for production use.

