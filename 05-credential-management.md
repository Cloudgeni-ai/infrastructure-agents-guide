# Chapter 5: Credential Management

> Short-lived tokens, vault patterns, and blast radius control.

---

## The Cardinal Rule

> **Agents never hold long-lived credentials. Ever.**

No API keys in environment variables. No service principal secrets in config files. No SSH keys on disk. The agent requests credentials just-in-time from a broker, gets a short-lived token scoped to the minimum required permissions, and the token expires automatically.

This is the single most important security pattern for infrastructure agents.

---

## Why This Matters

The report on autonomous agents documents multiple real-world incidents:
- **Infostealer malware** targeting agent config directories to extract API keys and auth tokens
- **Prompt injection** causing agents to exfiltrate credentials via tool calls
- **Framework CVEs** enabling secret extraction from serialized agent state

If credentials are short-lived and narrowly scoped, all of these attacks have limited blast radius.

---

## Credential Flow Architecture

```mermaid
sequenceDiagram
    participant Agent as Agent (Sandbox)
    participant Internal as Internal Server (Worker)
    participant API as API Server
    participant Vault as Secret Store
    participant Cloud as Cloud Provider

    Agent->>Internal: Request credentials<br/>(integrationId, scope)
    Internal->>Internal: Validate request against<br/>task context & policy

    Internal->>API: RPC: generateCloudToken<br/>(integrationId, orgId, scope)
    API->>Vault: Fetch service principal<br/>or refresh token
    Vault-->>API: Long-lived credential<br/>(never leaves API server)

    API->>Cloud: Exchange for short-lived token<br/>(STS AssumeRole / OAuth token exchange)
    Cloud-->>API: Short-lived token<br/>(1h expiry, scoped permissions)

    API-->>Internal: Short-lived token
    Internal-->>Agent: Token (expires in 1h)

    Agent->>Cloud: Use token for API calls
    Note over Agent,Cloud: Token auto-expires<br/>No revocation needed
```

---

## Implementation by Cloud Provider

### AWS: STS AssumeRole

```typescript
import { STSClient, AssumeRoleCommand } from '@aws-sdk/client-sts';

async function generateAwsToken(
  integration: AwsIntegration,
  sessionName: string
): Promise<AwsCredentials> {
  const sts = new STSClient({ region: integration.defaultRegion });

  const response = await sts.send(new AssumeRoleCommand({
    RoleArn: integration.roleArn,        // Cross-account role
    RoleSessionName: `agent-${sessionName}`,
    DurationSeconds: 3600,               // 1 hour
    // Optional: further restrict permissions
    Policy: JSON.stringify({
      Version: '2012-10-17',
      Statement: [{
        Effect: 'Allow',
        Action: [
          's3:GetObject', 's3:PutObject',  // Only what's needed
          'ec2:Describe*',
        ],
        Resource: '*',
      }],
    }),
  }));

  return {
    accessKeyId: response.Credentials.AccessKeyId,
    secretAccessKey: response.Credentials.SecretAccessKey,
    sessionToken: response.Credentials.SessionToken,
    expiration: response.Credentials.Expiration,
  };
}
```

### Azure: OAuth Token Exchange

```typescript
import { ConfidentialClientApplication } from '@azure/msal-node';

async function generateAzureToken(
  integration: AzureIntegration,
  scope: string = 'https://management.azure.com/.default'
): Promise<AzureToken> {
  // For OAuth integrations: use refresh token
  if (integration.authMethod === 'OAuth') {
    const msalClient = new ConfidentialClientApplication({
      auth: {
        clientId: AZURE_APP_CLIENT_ID,
        clientSecret: AZURE_APP_CLIENT_SECRET,
        authority: `https://login.microsoftonline.com/${integration.tenantId}`,
      },
    });

    const result = await msalClient.acquireTokenByRefreshToken({
      refreshToken: integration.refreshToken,  // Stored encrypted
      scopes: [scope],
    });

    return {
      accessToken: result.accessToken,
      expiresOn: result.expiresOn,
      subscriptionId: integration.subscriptionId,
    };
  }

  // For service principal: client credentials flow
  // Still produces a short-lived token
  const result = await msalClient.acquireTokenByClientCredential({
    scopes: [scope],
  });

  return { accessToken: result.accessToken, expiresOn: result.expiresOn };
}
```

### GCP: Service Account Impersonation

```typescript
import { IAMCredentialsClient } from '@google-cloud/iam-credentials';

async function generateGcpToken(
  integration: GcpIntegration,
  scopes: string[] = ['https://www.googleapis.com/auth/cloud-platform']
): Promise<GcpToken> {
  const client = new IAMCredentialsClient();

  // Impersonate the service account to get a short-lived token
  const [response] = await client.generateAccessToken({
    name: `projects/-/serviceAccounts/${integration.serviceAccountEmail}`,
    scope: scopes,
    lifetime: { seconds: 3600 }, // 1 hour
  });

  return {
    accessToken: response.accessToken,
    expireTime: response.expireTime,
  };
}
```

---

## The Credential Broker (Internal Server)

The credential broker runs inside the worker and validates every credential request:

```typescript
async function handleGetCredentials(req: Request): Promise<Response> {
  const { integrationId, scope } = await req.json();
  const context = getCurrentTaskContext();

  // 1. Validate integration belongs to this organization
  if (!context.authorizedIntegrations.includes(integrationId)) {
    return Response.json(
      { error: 'Integration not authorized for this task' },
      { status: 403 }
    );
  }

  // 2. Validate scope is within allowed limits
  if (scope && !isAllowedScope(scope, context.policy)) {
    return Response.json(
      { error: `Scope '${scope}' exceeds policy limits` },
      { status: 403 }
    );
  }

  // 3. Request short-lived token via RPC to API server
  const token = await rpcToApiServer('generateCloudToken', {
    integrationId,
    organizationId: context.organizationId,
    scope,
    requestedBy: `agent:${context.agentSlug}:${context.sessionId}`,
  });

  // 4. Log the credential issuance for audit
  await emitAuditEvent({
    type: 'credential_issued',
    integrationId,
    scope,
    agentSlug: context.agentSlug,
    sessionId: context.sessionId,
    expiresAt: token.expiresAt,
  });

  return Response.json(token);
}
```

---

## Secret Storage Alternatives

### HashiCorp Vault

```typescript
import Vault from 'node-vault';

const vault = Vault({
  endpoint: process.env.VAULT_ADDR,
  token: process.env.VAULT_TOKEN,
});

// Store integration credentials
await vault.write('secret/data/integrations/aws-prod', {
  data: {
    roleArn: 'arn:aws:iam::123456789:role/agent-role',
    externalId: 'cloudgeni-agent',
  },
});

// Read at token generation time (API server only)
const secret = await vault.read('secret/data/integrations/aws-prod');
```

### AWS Secrets Manager

```typescript
import { SecretsManager } from '@aws-sdk/client-secrets-manager';

const sm = new SecretsManager();
const secret = await sm.getSecretValue({
  SecretId: `integrations/${integrationId}`,
});
const credentials = JSON.parse(secret.SecretString);
```

### Azure Key Vault

```typescript
import { SecretClient } from '@azure/keyvault-secrets';
import { DefaultAzureCredential } from '@azure/identity';

const client = new SecretClient(
  `https://${VAULT_NAME}.vault.azure.net`,
  new DefaultAzureCredential()
);

const secret = await client.getSecret(`integration-${integrationId}`);
```

### 1Password (via Connect Server)

```typescript
const response = await fetch(`${OP_CONNECT_URL}/v1/vaults/${VAULT_ID}/items/${ITEM_ID}`, {
  headers: { 'Authorization': `Bearer ${OP_TOKEN}` },
});
const item = await response.json();
```

---

## Defense-in-Depth Checklist

```
[ ] Long-lived credentials ONLY in API server / vault — never in workers
[ ] Short-lived tokens (1h max) for all agent operations
[ ] Scope tokens to minimum required permissions
[ ] Validate integration ownership before issuing tokens
[ ] Log every credential issuance with correlation ID
[ ] Block cloud metadata endpoints in sandbox (169.254.169.254)
[ ] Encrypt credentials at rest in the database
[ ] Rotate refresh tokens on a schedule
[ ] Credential requests fail closed (deny on error)
[ ] No credentials in agent prompts, logs, or error messages
```

---

## Anti-Patterns to Avoid

| Anti-Pattern | Why It's Dangerous | Better Approach |
|-------------|-------------------|-----------------|
| API keys in env vars | Visible to all processes, logged in crashes | Credential broker with short-lived tokens |
| Service principal secrets in agent config | Stolen by infostealers, leaked in serialization | Managed identity or vault-backed generation |
| Long-lived OAuth tokens | Wide blast radius if compromised | Refresh token in vault, short-lived access tokens |
| Shared credentials across agents | One compromised agent = all agents compromised | Per-agent, per-session token issuance |
| Credentials in LLM prompts | Model can repeat them in output | Inject into tool environment, not prompt text |
| No credential expiry | Compromised tokens live forever | 1h TTL with no renewal option |

---

## Key Takeaways

1. **Short-lived tokens only** — 1 hour max, no renewal
2. **Credential broker** validates every request against task context and policy
3. **API server holds secrets**, workers/agents never do
4. **Audit every issuance** — who, what scope, when, which agent session
5. **Block metadata endpoints** — prevents credential theft via SSRF
6. **Use managed identity** where possible (Azure Container Apps, AWS IAM roles)

---

## Next Chapter

[Chapter 6: Change Control & GitOps →](./06-change-control.md)
