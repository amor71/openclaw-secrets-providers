# Requirements: AWS, Azure & HashiCorp Vault Secrets Providers

> **Status:** Draft for review — no code until sign-off  
> **Parent PR:** #16663 (GCP Secret Manager)  
> **Related:** #11539 (akoscz `SecretsProvider` interface)  
> **Foundation:** [REQUIREMENTS-gcp-secrets.md](https://github.com/amor71/openclaw-contrib/blob/main/REQUIREMENTS-gcp-secrets.md) — the original GCP Secret Manager requirements  
> **Date:** 2026-02-15

---

## 1. Overview

This document extends the original requirements established in [REQUIREMENTS-gcp-secrets.md](https://github.com/amor71/openclaw-contrib/blob/main/REQUIREMENTS-gcp-secrets.md) for the GCP Secret Manager provider. That document defines the core problem statement (§1), goals (§2), functional requirements for storage/retrieval, per-agent isolation, bootstrapping, migration, CLI, error handling, and backward compatibility (§4), user stories (§5), and access model (§6). **All of those requirements remain in force and are not repeated here.** This document covers only what is NEW for the multi-provider expansion.

Extend OpenClaw's secrets infrastructure (established by the GCP Secret Manager PR) with three additional providers:

| Provider | Backend | SDK |
|----------|---------|-----|
| `aws` | AWS Secrets Manager | `@aws-sdk/client-secrets-manager` |
| `azure` | Azure Key Vault | `@azure/keyvault-secrets` + `@azure/identity` |
| `vault` | HashiCorp Vault (KV v2) | `node-vault` or plain HTTP (Vault HTTP API) |

Each provider MUST implement the existing `SecretProvider` interface and integrate seamlessly with the `${provider:name}` reference syntax, caching layer, CLI commands, and config resolution already shipping with GCP.

---

## 2. Secret Reference Syntax

References follow the established pattern — provider name is the discriminator:

```
${aws:my-secret}              # latest version
${aws:my-secret#versionId}    # version-pinned (AWS versionId or versionStage)
${azure:my-secret}            # latest version
${azure:my-secret#version}    # version-pinned (Azure version id)
${vault:my-secret}            # latest (KV v2 latest version)
${vault:path/to/secret}       # path-based (Vault mount/path)
${vault:my-secret#3}          # version-pinned (KV v2 version number)
```

All providers use lowercase names matching `[a-z]+` per the existing regex.

---

## 3. Authentication

### 3.1 AWS Secrets Manager

| Method | Description |
|--------|-------------|
| **Default credential chain** | SDK auto-discovers: env vars (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`), shared credentials file (`~/.aws/credentials`), EC2/ECS instance metadata, SSO |
| **Named profile** | Config option `profile` → uses `~/.aws/credentials [profile]` |
| **Explicit credentials file** | Config option `credentialsFile` → path to JSON with `accessKeyId`, `secretAccessKey`, `region` |
| **Assume Role** | Config option `roleArn` → STS AssumeRole for cross-account access |

**Required config fields:** `region` (e.g. `us-east-1`). Optional: `profile`, `credentialsFile`, `roleArn`, `externalId`.

### 3.2 Azure Key Vault

| Method | Description |
|--------|-------------|
| **DefaultAzureCredential** | Auto-discovers: env vars, managed identity, Azure CLI, VS Code, PowerShell |
| **Service Principal (client secret)** | `AZURE_TENANT_ID` + `AZURE_CLIENT_ID` + `AZURE_CLIENT_SECRET` |
| **Service Principal (certificate)** | `AZURE_TENANT_ID` + `AZURE_CLIENT_ID` + `AZURE_CLIENT_CERTIFICATE_PATH` |
| **Managed Identity** | Automatic on Azure VMs/App Service/Functions |
| **Explicit credentials file** | Config option `credentialsFile` → JSON with `tenantId`, `clientId`, `clientSecret` |

**Required config fields:** `vaultUrl` (e.g. `https://my-vault.vault.azure.net`). Optional: `credentialsFile`, `tenantId`, `clientId`.

### 3.3 HashiCorp Vault

| Method | Description |
|--------|-------------|
| **Token** | `VAULT_TOKEN` env var or config `token` field |
| **AppRole** | Config `roleId` + `secretId` → login to get token |
| **Kubernetes** | Auto JWT from service account → Vault k8s auth |
| **OIDC/JWT** | Config `jwt` or auto-discover from platform |
| **Token file** | Config `tokenFile` → reads token from file path (supports k8s secret mounts) |

**Required config fields:** `address` (e.g. `http://127.0.0.1:8200`). Optional: `token`, `tokenFile`, `namespace`, `mountPath` (default `secret`), `roleId`, `secretId`, `authMethod`.

---

## 4. Per-Agent Isolation

Each provider must support scoping secrets to individual agents, matching the GCP pattern of per-agent service accounts.

### 4.1 AWS

- **IAM Users or Roles per agent:** `openclaw-<agent>` IAM user/role with policy restricting `secretsmanager:GetSecretValue` to secrets prefixed with `openclaw-<agent>-*` or tagged `openclaw-agent=<agent>`.
- **Resource-based policies** on individual secrets can further restrict access.
- Setup command creates IAM policies with `Resource` conditions scoped to the agent's secret prefix.

### 4.2 Azure

- **Service Principals per agent:** Each agent gets an Azure AD app registration / service principal.
- **Key Vault Access Policies** or **RBAC** (Key Vault Secrets User role) scoped per principal.
- Setup command assigns `Key Vault Secrets User` role to each agent's principal, optionally with condition on secret name prefix.

### 4.3 HashiCorp Vault

- **Per-agent policies:** Vault policy `openclaw-<agent>` with `read` capability on `secret/data/openclaw/<agent>/*`.
- **AppRole per agent:** Each agent gets its own `role_id` / `secret_id` pair bound to its policy.
- Setup command creates policies and AppRole entities via Vault CLI/API.

---

## 5. CLI Commands

All commands follow the existing `openclaw secrets <subcommand>` pattern. The `--provider` flag selects the backend.

### 5.1 `openclaw secrets setup --provider <aws|azure|vault> [options]`

| Provider | Key Options |
|----------|-------------|
| `aws` | `--region`, `--profile`, `--role-arn`, `--agents` |
| `azure` | `--vault-url`, `--subscription`, `--resource-group`, `--agents` |
| `vault` | `--address`, `--auth-method`, `--mount-path`, `--agents` |

**Behavior:** Validates prerequisites (CLI tools, connectivity), creates per-agent identities/policies, writes `secrets.providers.<name>` to `openclaw.json`. Idempotent.

### 5.2 `openclaw secrets migrate --provider <name>`

Scans config for plaintext secrets (existing heuristic: `sk-`, `xoxb-`, etc.), uploads to the chosen provider, replaces with `${provider:name}` references. Same upload → verify → purge flow as GCP.

### 5.3 `openclaw secrets test --provider <name>`

Verifies connectivity and that all referenced secrets resolve.

### 5.4 `openclaw secrets list --provider <name>`

Lists secrets stored in the provider.

### 5.5 `openclaw secrets set --provider <name> --name <n> --value <v>`

Stores/updates a single secret.

---

## 6. Caching Behavior

All providers use the **same shared cache** already implemented in `secret-resolution.ts`:

- **TTL:** Configurable per provider via `cacheTtlSeconds` (default: 300s / 5 min)
- **Stale-while-revalidate:** On fetch failure, return expired cached value if available
- **Cache key format:** `<provider>:<name>#<version|latest>`
- **Cache scope:** In-memory, per-process (same `Map<string, CacheEntry>`)
- `clearSecretCache()` clears all providers

No provider-specific cache logic needed — the shared `fetchWithCache` wrapper handles it.

---

## 7. Error Handling

Each provider MUST map platform-specific errors to consistent behavior:

| Scenario | AWS | Azure | Vault | Behavior |
|----------|-----|-------|-------|----------|
| Secret not found | `ResourceNotFoundException` | 404 `SecretNotFound` | 404 | Throw with clear message including secret name + project/vault |
| Permission denied | `AccessDeniedException` | 403 `Forbidden` | 403 | Throw with IAM/RBAC guidance |
| Network timeout | SDK timeout | SDK timeout | `ECONNREFUSED` / timeout | Retry once, then stale-while-revalidate, then throw |
| Auth expired | `ExpiredTokenException` | `CredentialUnavailableError` | 403 token expired | Re-authenticate (AppRole/STS) if possible, else throw |
| SDK not installed | Import fails | Import fails | Import fails | Throw with `pnpm add <package>` instructions |
| Vault sealed | N/A | N/A | 503 sealed | Throw with "Vault is sealed" message |

All errors flow through `SecretResolutionError` with `provider`, `secretName`, `configPath`, and `cause`.

---

## 8. Configuration Schema

Added to `openclaw.json` under `secrets.providers`:

```jsonc
{
  "secrets": {
    "providers": {
      // Existing
      "gcp": { "project": "my-project", "cacheTtlSeconds": 300 },
      
      // New
      "aws": {
        "region": "us-east-1",
        "cacheTtlSeconds": 300,
        // Optional:
        "profile": "openclaw",
        "credentialsFile": "/path/to/creds.json",
        "roleArn": "arn:aws:iam::123456789012:role/openclaw"
      },
      "azure": {
        "vaultUrl": "https://my-vault.vault.azure.net",
        "cacheTtlSeconds": 300,
        // Optional:
        "credentialsFile": "/path/to/creds.json",
        "tenantId": "...",
        "clientId": "..."
      },
      "vault": {
        "address": "http://127.0.0.1:8200",
        "cacheTtlSeconds": 300,
        // Optional:
        "namespace": "admin",
        "mountPath": "secret",
        "authMethod": "token",  // "token" | "approle" | "kubernetes"
        "tokenFile": "/var/run/secrets/vault-token",
        "roleId": "...",
        "secretId": "..."
      }
    }
  }
}
```

Multiple providers can be active simultaneously — refs are dispatched by prefix (`${aws:x}` vs `${vault:y}`).

---

## 9. Dependencies

| Provider | Required Packages | Peer / Optional |
|----------|------------------|-----------------|
| `aws` | `@aws-sdk/client-secrets-manager` | `@aws-sdk/client-sts` (for AssumeRole) |
| `azure` | `@azure/keyvault-secrets`, `@azure/identity` | — |
| `vault` | None (uses Vault HTTP API via `fetch`) | `node-vault` (optional convenience wrapper) |

All SDKs are **optional peer dependencies** — dynamically imported at runtime, with clear install instructions on missing module error (matching GCP pattern).

---

## 10. Testing Strategy

### 10.1 Unit Tests (CI, no credentials needed)

- **Mock SDK clients** — same pattern as GCP tests (`vi.doMock(...)`)
- **Interface compliance:** Each provider passes the same test suite verifying `SecretProvider` contract
- **Error mapping:** Each platform error code → expected OpenClaw error type
- **Cache interaction:** Verify cache hits, misses, stale-while-revalidate per provider
- **Config validation:** Zod schema rejects invalid config, accepts valid

### 10.2 CLI Tests

- **Mock injection** via `_mock*` options (established pattern from `secrets.test.ts`)
- **Setup idempotency:** Running setup twice doesn't error
- **Migrate flow:** scan → upload → verify → purge (with mock provider)

### 10.3 Integration Tests (optional, credential-gated)

- Guarded by `OPENCLAW_TEST_AWS=1`, `OPENCLAW_TEST_AZURE=1`, `OPENCLAW_TEST_VAULT=1` env vars
- Use real SDKs against test accounts / local Vault dev server
- `vault server -dev` for HashiCorp Vault integration tests
- Skipped in CI by default

### 10.4 Test File Structure

```
src/config/providers/aws-secret-provider.test.ts
src/config/providers/azure-secret-provider.test.ts
src/config/providers/vault-secret-provider.test.ts
src/commands/secrets.aws.test.ts     (CLI tests)
src/commands/secrets.azure.test.ts
src/commands/secrets.vault.test.ts
```

---

## 11. Secret Rotation

> The original GCP requirements (§3) listed automatic secret rotation as out of scope. This section brings it in scope for the multi-provider expansion.

### 11.1 Goals

- Secrets should be rotatable without manual intervention or agent downtime
- When a secret rotates, all agents using it must transparently pick up the new value
- The caching layer must invalidate stale values on rotation events
- Rotation must be provider-native — OpenClaw orchestrates, not reimplements

### 11.2 Provider-Specific Rotation Requirements

#### AWS Secrets Manager
- Support AWS-native rotation via Lambda functions
- Allow configuring rotation schedule and window via `openclaw secrets rotation` CLI
- Accept rotation Lambda ARN (built-in templates for RDS/Redshift/DocumentDB or custom)
- Listen for `SecretRotationSucceeded` / `SecretRotationFailed` CloudWatch events to trigger cache invalidation

#### Azure Key Vault
- Support event-driven rotation via Event Grid (`SecretNearExpiry`, `SecretExpired` events)
- Provide guidance and optional CLI setup for wiring events to Azure Functions
- Less turnkey than AWS — document the required user-side setup clearly
- On receiving rotation events, invalidate cached value and re-fetch

#### HashiCorp Vault
- **Dynamic secrets (primary model):** Support Vault's lease-based dynamic secret generation — credentials are short-lived and ephemeral, not rotated
- Manage lease lifecycle: request → use → renew or let expire → request new
- Track lease IDs and TTLs; renew leases before expiry when possible
- **Static secret rotation:** Support Vault's database static role rotation on a configurable schedule
- On lease expiry or rotation, invalidate cache and obtain fresh credentials

#### GCP Secret Manager
- GCP does not have built-in auto-rotation like AWS. Rotation is user-orchestrated via Cloud Scheduler + Cloud Functions (or equivalent)
- **Pub/Sub integration:** Support `SECRET_VERSION_ADD` event topic notifications — when a new secret version is created (by rotation or manual update), GCP publishes to a Pub/Sub topic. OpenClaw should subscribe to these events to trigger immediate cache invalidation
- **Polling fallback:** If Pub/Sub is not configured, fall back to TTL-based cache expiry (existing behavior) — rotated values are picked up after cache TTL expires
- `openclaw secrets rotation setup --provider gcp` should: create the Pub/Sub topic + subscription, configure Secret Manager notification policies, and document the Cloud Scheduler + Cloud Function pattern for the actual rotation logic
- Provide a sample Cloud Function template for common rotation patterns (API key regeneration)
- Our existing TTL + stale-while-revalidate caching already handles the passive case; Pub/Sub adds real-time awareness

### 11.3 Cache Invalidation on Rotation

- When a rotation event is detected (via provider-specific mechanism), the corresponding cache entry MUST be invalidated immediately
- The next access to the rotated secret fetches the new value from the provider
- If event-based invalidation is not configured, the existing TTL-based cache expiry serves as the fallback
- Agents must NOT cache rotated-out values beyond one cache TTL cycle

### 11.4 Rotation Event Notifications

- When a secret rotation is detected, emit an internal `secret:rotated` event with `{ provider, secretName, timestamp }`
- Agents and plugins can subscribe to this event to take action (e.g., reconnect a database, refresh a token)
- Events are best-effort — missed events fall back to TTL-based refresh

### 11.5 CLI Commands for Rotation

- `openclaw secrets rotation status --provider <name>` — show rotation config for all secrets
- `openclaw secrets rotation enable --provider <name> --secret <s> [--schedule <cron>] [--lambda-arn <arn>]` — configure rotation (AWS)
- `openclaw secrets rotation disable --provider <name> --secret <s>` — disable rotation
- Vault: `openclaw secrets lease list` / `openclaw secrets lease renew --lease-id <id>`

### 11.6 Non-Rotatable Secrets

Many API keys are issued by third parties (Alpaca, Anthropic, OpenAI, Brave, etc.) and cannot be programmatically rotated — the provider gives you a static key that lives until you manually regenerate it in their dashboard.

For these secrets:

- **Store & protect** — still keep them in the secrets manager (encrypted, access-controlled, audited). The value of centralized secrets management applies regardless of rotation capability.
- **`rotation: "manual"` classification** — secrets must be configurable as `rotation: "manual"` so the system does not attempt auto-rotation. This is the default for any secret without explicit rotation config.
- **Periodic review reminders** — optionally configure a review interval (e.g., `reviewEveryDays: 90`). OpenClaw emits a `secret:review-due` event when the interval elapses, reminding the admin to check if the key should be regenerated.
- **Expiry tracking** — if the provider supports expiration dates (some API keys have TTLs), track them and emit `secret:expiring-soon` before expiry.
- **Failure detection** — if an agent receives a 401/403 from an API using a managed secret, emit a `secret:auth-failed` event with the secret name. This doesn't auto-rotate (can't), but alerts the admin that the key may be revoked or expired.
- **CLI:** `openclaw secrets list --rotation-status` shows each secret's rotation classification (`auto`, `manual`, `dynamic`) and last review/rotation date.

### 11.7 Non-Requirements (Rotation)

- OpenClaw does NOT implement the rotation logic itself — it delegates to provider-native mechanisms
- OpenClaw does NOT generate new secret values — that is the rotation function's job
- Rotation setup for Azure requires user-side Azure Function/Logic App creation — OpenClaw provides templates and docs, not a fully automated setup
- OpenClaw does NOT auto-rotate third-party API keys — it can only remind, detect failures, and alert

---

## 12. Non-Functional Requirements

- **No breaking changes:** Existing GCP users are unaffected
- **Lazy loading:** SDKs loaded only when provider is referenced — zero cost for unused providers
- **Startup impact:** Secret resolution happens at config load, before agent boot. Must not add >500ms to cold start (cached) or >3s (uncached, network)
- **Secret size:** Support secrets up to 64KB (AWS limit: 64KB, Azure: 25KB for secrets / 200KB for certs, Vault: configurable). Document limits per provider.
- **Binary secrets:** Support UTF-8 string secrets only (matching GCP). Binary payloads out of scope.
- **Audit logging:** Each provider should log (at debug level) when a secret is fetched vs served from cache, without logging the secret value itself.
