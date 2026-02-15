# Requirements: AWS, Azure & HashiCorp Vault Secrets Providers

> **Status:** Draft for review — no code until sign-off  
> **Parent PR:** #16663 (GCP Secret Manager)  
> **Related:** #11539 (akoscz `SecretsProvider` interface)  
> **Date:** 2026-02-15

---

## 1. Overview

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

## 11. Non-Functional Requirements

- **No breaking changes:** Existing GCP users are unaffected
- **Lazy loading:** SDKs loaded only when provider is referenced — zero cost for unused providers
- **Startup impact:** Secret resolution happens at config load, before agent boot. Must not add >500ms to cold start (cached) or >3s (uncached, network)
- **Secret size:** Support secrets up to 64KB (AWS limit: 64KB, Azure: 25KB for secrets / 200KB for certs, Vault: configurable). Document limits per provider.
- **Binary secrets:** Support UTF-8 string secrets only (matching GCP). Binary payloads out of scope.
- **Audit logging:** Each provider should log (at debug level) when a secret is fetched vs served from cache, without logging the secret value itself.
