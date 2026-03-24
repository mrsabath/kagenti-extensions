# GitHub Issue Agent Demo — KubeCon EU 2026

Simplified walkthrough for the **GitHub Issue Agent** with **AuthBridge** demo.
Infrastructure setup (webhook, Keycloak, ConfigMaps) is done via CLI; the agent
and tool are imported through the Kagenti UI.

For the full guide with prerequisites, architecture, troubleshooting, and cleanup,
see [demo-ui.md](demo-ui.md).

---

## Step 1: Configure Keycloak

```bash
cd AuthBridge

# Create virtual environment (if not already done)
python -m venv venv
source venv/bin/activate
pip install --upgrade pip
pip install -r requirements.txt

# Run the Keycloak setup for this demo
python demos/github-issue/setup_keycloak.py
```

---

## Step 2: Apply Demo ConfigMaps

The Kagenti installer creates default ConfigMaps (`authbridge-config`,
`spiffe-helper-config`, `envoy-config`) and the `keycloak-admin-secret` Secret
in the target namespace. No manual secret creation is needed for this demo.

Apply the demo-specific ConfigMaps — `authproxy-routes` configures per-route
token exchange, and `authbridge-config` sets the agent's SPIFFE ID for inbound
audience validation:

```bash
cd AuthBridge

kubectl apply -f demos/github-issue/k8s/configmaps.yaml
```

---

## Step 3: Create the GitHub Tool Secrets

```bash
export PRIVILEGED_ACCESS_PAT=<your-privileged-pat>
export PUBLIC_ACCESS_PAT=<your-public-pat>
```

```bash
kubectl create secret generic github-tool-secrets -n team1 \
  --from-literal=INIT_AUTH_HEADER="Bearer $PRIVILEGED_ACCESS_PAT" \
  --from-literal=UPSTREAM_HEADER_TO_USE_IF_IN_AUDIENCE="Bearer $PRIVILEGED_ACCESS_PAT" \
  --from-literal=UPSTREAM_HEADER_TO_USE_IF_NOT_IN_AUDIENCE="Bearer $PUBLIC_ACCESS_PAT"
```

---

## Step 4: Import the GitHub Tool via Kagenti UI

1. Navigate to [Import Tool](http://kagenti-ui.localtest.me:8080/tools/import).

2. **Namespace**: `team1`

3. Select **Build from Source**.

4. Under **Source Code**:
   - **Git Repository URL**: `https://github.com/kagenti/agent-examples`
   - **Branch or Tag**: `main`
   - **Example Tools**: `GitHub Tool`
   - **Source Subfolder**: `mcp/github_tool`

5. **Workload Type**: `Deployment`

6. **MCP Transport Protocol**: `streamable HTTP`

7. **Enable AuthBridge sidecar injection**: **unchecked**

8. **Enable SPIRE identity (spiffe-helper sidecar)**: **unchecked**

9. Under **Port Configuration**, set **Service Port** to `9090` and **Target Port** to `9090`

10. Under **Environment Variables**, click **Import from File/URL**,
    select **From URL** and provide:
    - **URL**: `https://raw.githubusercontent.com/kagenti/agent-examples/refs/heads/main/mcp/github_tool/.env.authbridge`
    - Click **Fetch & Parse**, then **Import**.

11. Click **Build & Deploy New Tool**. Wait for the Shipwright build to complete.

### Verify the tool is reachable

```bash
kubectl run test-mcp --image=curlimages/curl -n team1 --restart=Never --rm -it -- \
  curl -s -o /dev/null -w "%{http_code}" --max-time 5 http://github-tool-mcp:9090/mcp
# Expected: 200
```

---

## Step 5: Import the GitHub Issue Agent via Kagenti UI

1. Navigate to [Import Agent](http://kagenti-ui.localtest.me:8080/agents/import).

2. **Namespace**: `team1`

3. Select **Build from Source**.

4. Under **Source Repository**:
   - **Git Repository URL**: `https://github.com/kagenti/agent-examples`
   - **Git Branch**: `main`
   - **Select Example**: `Git Issue Agent`
   - **Source Path**: `a2a/git_issue_agent`

5. **Protocol**: `A2A`

6. **Framework**: `LangGraph`

7. **Workload Type**: `Deployment`

8. **Enable AuthBridge sidecar injection**: **checked**

9. **Enable SPIRE identity (spiffe-helper sidecar)**: **checked**

10. Under **Port Configuration**, set **Service Port** to `8080` and **Target Port** to `8000`

11. Under **Environment Variables**, click **Import from File/URL**,
    select **From URL** and provide:
    - For Ollama: `https://raw.githubusercontent.com/kagenti/agent-examples/refs/heads/main/a2a/git_issue_agent/.env.ollama`
    - For OpenAI: `https://raw.githubusercontent.com/kagenti/agent-examples/refs/heads/main/a2a/git_issue_agent/.env.openai`
    - Click **Fetch & Parse**, then **Import**.

    > **OpenAI prerequisite:** Create the secret first:
    > ```bash
    > kubectl create secret generic openai-secret -n team1 \
    >   --from-literal=apikey="<YOUR_OPENAI_API_KEY>"
    > ```

12. Click **Build & Deploy Agent**. Wait for the build to complete.

---

## Step 6: Verify the Deployment

### Verify injected containers

```bash
kubectl get pod -n team1 -l app.kubernetes.io/name=git-issue-agent -o jsonpath='{.items[0].spec.containers[*].name}'
```

Expected:

```
agent kagenti-client-registration envoy-proxy spiffe-helper
```

### Check client registration

```bash
kubectl logs deployment/git-issue-agent -n team1 -c kagenti-client-registration
```

Expected:

```
SPIFFE credentials ready!
Client ID (SPIFFE ID): spiffe://localtest.me/ns/team1/sa/git-issue-agent
Created Keycloak client "spiffe://localtest.me/ns/team1/sa/git-issue-agent"
Client registration complete!
```

### Check agent logs

```bash
kubectl logs deployment/git-issue-agent -n team1 -c agent
```

Expected:

```
INFO: JWKS_URI is set - using JWT Validation middleware
INFO:     Started server process [17]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

---

## Step 7: Chat via Kagenti UI

1. Navigate to the **Agent Catalog** in the Kagenti UI.
2. Select the `team1` namespace.
3. Under **Available Agents**, select `git-issue-agent` and click **View Details**.
4. Verify the **Agent Card** is visible.
5. Use the **Chat** panel to send: "List 10 open issues in kagenti/kagenti repo".
6. The agent should respond with a list of GitHub issues.

---

## Step 8: Test via CLI

Test the AuthBridge flow from the command line to verify inbound validation and
token exchange.

### Setup

```bash
kubectl run test-client --image=nicolaka/netshoot -n team1 --restart=Never -- sleep 3600
kubectl wait --for=condition=ready pod/test-client -n team1 --timeout=30s
```

### 8a. Agent Card — Public Endpoint (No Token Required)

```bash
kubectl exec test-client -n team1 -- curl -s \
  http://git-issue-agent:8080/.well-known/agent.json | jq
```

### 8b. Inbound Rejection — No Token

```bash
kubectl exec test-client -n team1 -- curl -s \
  http://git-issue-agent:8080/
```

### 8c. Inbound Rejection — Invalid Token

```bash
kubectl exec test-client -n team1 -- curl -s \
  -H "Authorization: Bearer invalid-token" \
  http://git-issue-agent:8080/
```

### 8d. End-to-End Test with Valid Token

Open a shell inside the test-client pod:

```bash
kubectl exec -it test-client -n team1 -- sh
```

Inside the pod:

```bash
ADMIN_TOKEN=$(curl -s http://keycloak-service.keycloak.svc:8080/realms/kagenti/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=admin" | jq -r ".access_token")

echo "Admin token length: ${#ADMIN_TOKEN}"

SPIFFE_ID="spiffe://localtest.me/ns/team1/sa/git-issue-agent"
CLIENTS=$(curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "http://keycloak-service.keycloak.svc:8080/admin/realms/kagenti/clients" \
  --data-urlencode "clientId=$SPIFFE_ID" --get)
INTERNAL_ID=$(echo "$CLIENTS" | jq -r ".[0].id")
CLIENT_ID=$(echo "$CLIENTS" | jq -r ".[0].clientId")

echo "Internal ID:   $INTERNAL_ID"
echo "Client ID:     $CLIENT_ID"

CLIENT_SECRET=$(echo "$CLIENTS" | jq -r ".[0].secret")

echo "Secret length: ${#CLIENT_SECRET}"

TOKEN=$(curl -s -X POST \
  "http://keycloak-service.keycloak.svc:8080/realms/kagenti/protocol/openid-connect/token" \
  -d "grant_type=client_credentials" \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$CLIENT_SECRET" | jq -r ".access_token")

echo "Token length:  ${#TOKEN}"

curl -s --max-time 300 \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://git-issue-agent:8080/ \
  -d '{
    "jsonrpc": "2.0",
    "id": "test-1",
    "method": "message/send",
    "params": {
      "message": {
        "role": "user",
        "messageId": "msg-001",
        "parts": [{"type": "text", "text": "List 10 open issues in kagenti/kagenti repo"}]
      }
    }
  }' | jq
```

Exit the pod:

```bash
exit
```

### 8e. Verify AuthProxy Logs

**Inbound validation:**

```bash
kubectl logs deployment/git-issue-agent -n team1 -c envoy-proxy 2>&1 | grep "\[Inbound\]"
```

Expected:

```
[Inbound] Token validated - issuer: http://keycloak.localtest.me:8080/realms/kagenti, audience: [spiffe://localtest.me/ns/team1/sa/git-issue-agent ...]
[Inbound] JWT validation succeeded, forwarding request
```

**Outbound token exchange:**

```bash
kubectl logs deployment/git-issue-agent -n team1 -c envoy-proxy 2>&1 | grep "^2026/" | grep "\[Token Exchange\]"
```

Expected:

```
[Token Exchange] Token URL: http://keycloak-service.keycloak.svc:8080/realms/kagenti/protocol/openid-connect/token
[Token Exchange] Client ID: spiffe://localtest.me/ns/team1/sa/git-issue-agent
[Token Exchange] Audience: github-tool
[Token Exchange] Scopes: openid github-tool-aud github-full-access
[Token Exchange] Successfully exchanged token
[Token Exchange] Successfully exchanged token, replacing Authorization header
```

### Clean up test client

```bash
kubectl delete pod test-client -n team1 --ignore-not-found
```

---

## Step 9: Access Control — Alice vs Bob

> **Known limitation:** Scope forwarding
> ([kagenti-extensions#139](https://github.com/kagenti/kagenti-extensions/issues/139))
> is not yet implemented. Currently all exchanged tokens include `github-full-access`
> from the static `token_scopes` in `authproxy-routes`.

This step demonstrates **scope-based access control**: two users with different
privilege levels get different GitHub API access through the same agent.

| User | Token Scope | Tool PAT Used | Public Repos | Private Repos |
|------|-------------|---------------|:------------:|:-------------:|
| **Alice** | `openid` (no `github-full-access`) | `PUBLIC_ACCESS_PAT` | Yes | No |
| **Bob** | `openid github-full-access` | `PRIVILEGED_ACCESS_PAT` | Yes | Yes |

> **Prerequisite:** You need a **private** GitHub repository that `PRIVILEGED_ACCESS_PAT`
> can access but `PUBLIC_ACCESS_PAT` cannot. Replace `<your-org/your-private-repo>`
> below with your own private repo.

### 9a. Open a shell inside the test-client pod

```bash
kubectl run test-client --image=nicolaka/netshoot -n team1 --restart=Never -- sleep 3600 2>/dev/null
kubectl wait --for=condition=ready pod/test-client -n team1 --timeout=30s
kubectl exec -it test-client -n team1 -- sh
```

### 9b. Get agent credentials

```bash
jwt_payload() {
  local p=$(echo "$1" | cut -d. -f2 | tr '_-' '/+')
  case $((${#p} % 4)) in 2) p="${p}==" ;; 3) p="${p}=" ;; esac
  echo "$p" | base64 -d
}

ADMIN_TOKEN=$(curl -s http://keycloak-service.keycloak.svc:8080/realms/kagenti/protocol/openid-connect/token \
  -d "grant_type=password" \
  -d "client_id=admin-cli" \
  -d "username=admin" \
  -d "password=admin" | jq -r ".access_token")

SPIFFE_ID="spiffe://localtest.me/ns/team1/sa/git-issue-agent"
CLIENTS=$(curl -s -H "Authorization: Bearer $ADMIN_TOKEN" \
  "http://keycloak-service.keycloak.svc:8080/admin/realms/kagenti/clients" \
  --data-urlencode "clientId=$SPIFFE_ID" --get)
INTERNAL_ID=$(echo "$CLIENTS" | jq -r ".[0].id")
CLIENT_ID=$(echo "$CLIENTS" | jq -r ".[0].clientId")
CLIENT_SECRET=$(echo "$CLIENTS" | jq -r ".[0].secret")
echo "Client ID: $CLIENT_ID  Secret length: ${#CLIENT_SECRET}"
```

### 9c. Test as Alice (public access only)

```bash
ALICE_TOKEN=$(curl -s -X POST \
  "http://keycloak-service.keycloak.svc:8080/realms/kagenti/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "username=alice" \
  -d "password=alice123" \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$CLIENT_SECRET" | jq -r ".access_token")

echo "Alice token length: ${#ALICE_TOKEN}"
echo "Alice scopes: $(jwt_payload $ALICE_TOKEN | jq -r '.scope')"
```

**Alice queries a public repo** (should succeed):

```bash
curl -s --max-time 300 \
  -H "Authorization: Bearer $ALICE_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://git-issue-agent:8080/ \
  -d '{
    "jsonrpc": "2.0",
    "id": "alice-public",
    "method": "message/send",
    "params": {
      "message": {
        "role": "user",
        "messageId": "msg-alice-1",
        "parts": [{"type": "text", "text": "List 10 open issues in kagenti/kagenti repo"}]
      }
    }
  }' | jq '.result.artifacts[0].parts[0].text' | head -5
```

**Alice queries a private repo** (should fail):

```bash
curl -s --max-time 300 \
  -H "Authorization: Bearer $ALICE_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://git-issue-agent:8080/ \
  -d '{
    "jsonrpc": "2.0",
    "id": "alice-private",
    "method": "message/send",
    "params": {
      "message": {
        "role": "user",
        "messageId": "msg-alice-2",
        "parts": [{"type": "text", "text": "List issues in <your-org/your-private-repo>"}]
      }
    }
  }' | jq '.result.artifacts[0].parts[0].text' | head -5
```

### 9d. Test as Bob (privileged access)

```bash
BOB_TOKEN=$(curl -s -X POST \
  "http://keycloak-service.keycloak.svc:8080/realms/kagenti/protocol/openid-connect/token" \
  -d "grant_type=password" \
  -d "username=bob" \
  -d "password=bob123" \
  -d "scope=openid github-full-access" \
  --data-urlencode "client_id=$CLIENT_ID" \
  --data-urlencode "client_secret=$CLIENT_SECRET" | jq -r ".access_token")

echo "Bob token length: ${#BOB_TOKEN}"
echo "Bob scopes: $(jwt_payload $BOB_TOKEN | jq -r '.scope')"
```

**Bob queries the same private repo** (should succeed):

```bash
curl -s --max-time 300 \
  -H "Authorization: Bearer $BOB_TOKEN" \
  -H "Content-Type: application/json" \
  -X POST http://git-issue-agent:8080/ \
  -d '{
    "jsonrpc": "2.0",
    "id": "bob-private",
    "method": "message/send",
    "params": {
      "message": {
        "role": "user",
        "messageId": "msg-bob-1",
        "parts": [{"type": "text", "text": "List issues in <your-org/your-private-repo>"}]
      }
    }
  }' | jq '.result.artifacts[0].parts[0].text' | head -5
```

### 9e. Verify scope-based PAT selection in tool logs

```bash
exit
kubectl logs deployment/github-tool -n team1 | grep -E "REQUIRED_SCOPE|scopes"
```

Expected:

```
This OIDC user has scopes "openid email profile"
The REQUIRED_SCOPE "github-full-access" NOT IN scopes [openid email profile]
This OIDC user has scopes "openid email profile github-full-access"
The REQUIRED_SCOPE "github-full-access" in scopes [openid email profile github-full-access]
```

### 9f. Clean up

```bash
kubectl delete pod test-client -n team1 --ignore-not-found
```
