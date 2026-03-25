# Cross-Compatibility

Use Poof's MCP tools from any language or framework — not just the TypeScript SDK.

## Contents
- [Authentication from Any Language](#authentication-from-any-language)
- [curl Examples](#curl-examples)
- [Python Helper](#python-helper)
- [OpenAI Function Calling Format](#openai-function-calling-format)
- [Direct REST Usage](#direct-rest-usage)

## Authentication from Any Language

The `@pooflabs/server` package handles Cognito authentication. You can call it from any language via a one-liner:

```bash
# Get a Cognito JWT token (requires Node.js installed)
# For staging, change appId and authApiUrl to staging values (see SKILL.md)
TOKEN=$(TAROBASE_SOLANA_KEYPAIR="<base58-private-key>" node -e "
  const { init, getIdToken } = require('@pooflabs/server');
  init({ appId: '697d5189a1e3dd2cc1a82d2b', authApiUrl: 'https://auth.tarobase.com' });
  getIdToken().then(t => process.stdout.write(t));
")
```

Install the package first: `npm install @pooflabs/server`

## curl Examples

### Discovery (no auth)

```bash
curl https://poof.new/api/mcp
```

### List projects

```bash
curl -X POST https://poof.new/api/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Wallet-Address: $WALLET_ADDRESS" \
  -d '{
    "jsonrpc": "2.0",
    "id": 1,
    "method": "tools/call",
    "params": {
      "name": "list_projects",
      "arguments": { "limit": 10 }
    }
  }'
```

### Create a project

```bash
curl -X POST https://poof.new/api/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Wallet-Address: $WALLET_ADDRESS" \
  -d '{
    "jsonrpc": "2.0",
    "id": 2,
    "method": "tools/call",
    "params": {
      "name": "create_project",
      "arguments": {
        "projectId": "'$(uuidgen | tr A-Z a-z)'",
        "firstMessage": "Build a todo app with Solana wallet auth",
        "tarobaseToken": "'"$TOKEN"'"
      }
    }
  }'
```

### Check AI status

```bash
curl -X POST https://poof.new/api/mcp \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $TOKEN" \
  -H "X-Wallet-Address: $WALLET_ADDRESS" \
  -d '{
    "jsonrpc": "2.0",
    "id": 3,
    "method": "tools/call",
    "params": {
      "name": "check_ai_active",
      "arguments": { "projectId": "'"$PROJECT_ID"'" }
    }
  }'
```

## Python Helper

```python
import requests
import subprocess
import json
import uuid
import os

POOF_ENVIRONMENTS = {
    "production": {
        "app_id": "697d5189a1e3dd2cc1a82d2b",
        "base_url": "https://poof.new",
        "auth_url": "https://auth.tarobase.com",
    },
    "staging": {
        "app_id": "6993d4b0b2b6ac08cd334dfb",
        "base_url": "https://staging.poof.new",
        "auth_url": "https://auth-staging.tarobase.com",
        "vercel_bypass_token": os.environ.get("VERCEL_BYPASS_TOKEN"),
    },
}

poof_env = POOF_ENVIRONMENTS[os.environ.get("POOF_ENV", "production")]
POOF_BASE_URL = poof_env["base_url"]

def get_token(private_key: str) -> str:
    """Get a Cognito JWT via the @pooflabs/server package."""
    result = subprocess.run(
        ["node", "-e", f"""
const {{ init, getIdToken }} = require('@pooflabs/server');
init({{ appId: '{poof_env["app_id"]}', authApiUrl: '{poof_env["auth_url"]}' }});
getIdToken().then(t => process.stdout.write(t));
"""],
        capture_output=True, text=True,
        env={**os.environ, "TAROBASE_SOLANA_KEYPAIR": private_key}
    )
    if result.returncode != 0:
        raise RuntimeError(f"Auth failed: {result.stderr}")
    return result.stdout

def mcp_call(token: str, wallet_address: str, tool_name: str, arguments: dict = None) -> dict:
    """Call an MCP tool and return the parsed result."""
    headers = {
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}",
        "X-Wallet-Address": wallet_address,
    }
    bypass_token = poof_env.get("vercel_bypass_token")
    if bypass_token:
        headers["x-vercel-protection-bypass"] = bypass_token
    resp = requests.post(
        f"{POOF_BASE_URL}/api/mcp",
        headers=headers,
        json={
            "jsonrpc": "2.0",
            "id": 1,
            "method": "tools/call",
            "params": {
                "name": tool_name,
                "arguments": arguments or {},
            },
        },
    )
    resp.raise_for_status()
    data = resp.json()
    if "error" in data:
        raise RuntimeError(data["error"]["message"])
    result = data.get("result", {})
    if result.get("isError"):
        raise RuntimeError(result["content"][0]["text"])
    content = result.get("content", [{}])[0].get("text", "{}")
    return json.loads(content)

# Usage
token = get_token("your-base58-private-key")
wallet_address = "your-solana-wallet-address"

projects = mcp_call(token, wallet_address, "list_projects", {"limit": 5})
print(projects)

project_id = str(uuid.uuid4())
mcp_call(token, wallet_address, "create_project", {
    "projectId": project_id,
    "firstMessage": "Build a voting app",
    "tarobaseToken": token,
})
```

## OpenAI Function Calling Format

MCP tool schemas map directly to OpenAI's function calling format. Mechanical conversion:

```python
# Fetch MCP tool definitions with full schemas via tools/list
resp = requests.post(
    f"{POOF_BASE_URL}/api/mcp",
    headers={
        "Content-Type": "application/json",
        "Authorization": f"Bearer {token}",
        "X-Wallet-Address": wallet_address,
    },
    json={"jsonrpc": "2.0", "id": 1, "method": "tools/list", "params": {}},
)
mcp_tools = resp.json()["result"]["tools"]

# Convert to OpenAI function calling format
openai_tools = [
    {
        "type": "function",
        "function": {
            "name": tool["name"],
            "description": tool["description"],
            "parameters": tool.get("inputSchema", {"type": "object", "properties": {}}),
        },
    }
    for tool in mcp_tools
]

# Use with OpenAI
from openai import OpenAI
client = OpenAI()

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[{"role": "user", "content": "List my Poof projects"}],
    tools=openai_tools,
)

# When the model calls a tool, forward it to Poof MCP
for call in response.choices[0].message.tool_calls or []:
    result = mcp_call(token, wallet_address, call.function.name, json.loads(call.function.arguments))
```

## Direct REST Usage

Skip MCP entirely and call REST endpoints directly. Same auth headers, standard HTTP:

```python
import requests

headers = {
    "Content-Type": "application/json",
    "Authorization": f"Bearer {token}",
    "X-Wallet-Address": wallet_address,
}

# List projects
projects = requests.get(f"{POOF_BASE_URL}/api/project?limit=10", headers=headers).json()

# Create project
requests.post(f"{POOF_BASE_URL}/api/project/{project_id}", headers=headers, json={
    "firstMessage": "Build a voting app",
    "tarobaseToken": token,
    "isPublic": True,
    "generationMode": "full",
})

# Send chat message
requests.post(f"{POOF_BASE_URL}/api/project/{project_id}/chat", headers=headers, json={
    "messageId": str(uuid.uuid4()),
    "message": "Add a leaderboard",
    "tarobaseToken": token,
})

# Check AI status
status = requests.get(f"{POOF_BASE_URL}/api/project/{project_id}/ai/active", headers=headers).json()

# Get files
files = requests.get(f"{POOF_BASE_URL}/api/project/{project_id}/files", headers=headers).json()

# Deploy to preview
requests.post(f"{POOF_BASE_URL}/api/project/{project_id}/deploy-mainnet-preview", headers=headers, json={
    "authToken": token,
})
```

See [api-reference.md](api-reference.md) for the full REST endpoint table.
