# Design: AWS, Azure & HashiCorp Vault Secrets Providers

> **Status:** Draft for review — no code until sign-off  
> **Prereq:** [REQUIREMENTS.md](./REQUIREMENTS.md)  
> **Date:** 2026-02-15

---

## 1. Architecture Overview

```
openclaw.json
  └─ secrets.providers.{aws,azure,vault,gcp}
       │
       ▼
┌─────────────────────────────────────────┐
│          buildSecretProviders()          │  ← factory, creates providers from config
│  (secret-resolution.ts — existing)      │
└─────────┬───────────────────────────────┘
          │ Map<string, SecretProvider>
          ▼
┌─────────────────────────────────────────┐
│     resolveConfigSecrets()              │  ← walks config tree, resolves ${p:name}
│     fetchWithCache()                    │  ← shared TTL + stale-while-revalidate
└─────────┬───────────────────────────────┘
          │ dispatches by provider prefix
          ▼
┌──────────┐ ┌───────────┐ ┌─────────────┐ ┌──────────────┐
│ GcpSecret│ │ AwsSecret │ │ AzureSecret │ │ VaultSecret  │
│ Provider │ │ Provider  │ │ Provider    │ │ Provider     │
│(existing)│ │  (new)    │ │   (new)     │ │   (new)      │
└──────────┘ └───────────┘ └─────────────┘ └──────────────┘
     │              │             │               │
     ▼              ▼             ▼               ▼
  GCP SDK      AWS SDK      Azure SDK       Vault HTTP API
```

Key principle: **The shared layer (`fetchWithCache`, cache, config walking, ref parsing) is unchanged.** New providers only implement the `SecretProvider` interface.

---

## 2. SecretProvider Interface

The existing interface (from `secret-resolution.ts`) aligns well with akoscz's `SecretsProvider` from PR #11539. We keep our version as-is:

```typescript
export interface SecretProvider {
  name: string;                                          // "aws" | "azure" | "vault" | "gcp"
  getSecret(name: string, version?: string): Promise<string>;
  setSecret(name: string, value: string): Promise<void>;
  listSecrets(): Promise<string[]>;
  testConnection(): Promise<{ ok: boolean; error?: string }>;
}
```

**Compatibility with akoscz's interface:**  
akoscz's `SecretsProvider` has `get(key)`, `set(key, value)`, `delete(key)`, `list()`. Our interface is a superset (adds `testConnection`, version support). If akoscz's PR merges first, we adapt with a thin adapter or merge the interfaces. The method signatures are compatible — only naming differs (`getSecret` vs `get`).

---

## 3. File Structure

```
src/config/
├── secret-resolution.ts              # EXISTING — shared cache, resolution, SecretProvider interface
├── secret-resolution.test.ts         # EXISTING
├── providers/
│   ├── gcp-secret-provider.ts        # EXTRACTED from secret-resolution.ts (refactor)
│   ├── gcp-secret-provider.test.ts   # EXTRACTED from secret-resolution.test.ts
│   ├── aws-secret-provider.ts        # NEW
│   ├── aws-secret-provider.test.ts   # NEW
│   ├── azure-secret-provider.ts      # NEW
│   ├── azure-secret-provider.test.ts # NEW
│   ├── vault-secret-provider.ts      # NEW
│   ├── vault-secret-provider.test.ts # NEW
│   └── index.ts                      # Re-exports all providers
src/commands/
├── secrets.ts                        # EXISTING — extended with provider routing
├── secrets.test.ts                   # EXISTING
├── secrets-setup-aws.ts              # NEW — AWS setup logic
├── secrets-setup-azure.ts            # NEW — Azure setup logic
├── secrets-setup-vault.ts            # NEW — Vault setup logic
```

**Refactor step:** Extract `GcpSecretProvider` from `secret-resolution.ts` into `providers/gcp-secret-provider.ts`. The parent file keeps the interface, cache, resolution logic, and `buildSecretProviders` factory.

---

## 4. Provider Implementations

### 4.1 AWS Secrets Manager (`AwsSecretProvider`)

```typescript
class AwsSecretProvider implements SecretProvider {
  name = "aws";
  
  constructor(config: {
    region: string;
    cacheTtlSeconds?: number;
    profile?: string;
    credentialsFile?: string;
    roleArn?: string;
    externalId?: string;
  })
}
```

**Auth flow:**
1. Dynamically import `@aws-sdk/client-secrets-manager`
2. Build credential provider chain:
   - If `credentialsFile` → `fromIni({ filepath })` or parse JSON
   - If `profile` → `fromIni({ profile })`
   - If `roleArn` → `fromTemporaryCredentials({ params: { RoleArn, ExternalId } })`
   - Else → default chain (env → shared credentials → IMDS)
3. Create `SecretsManagerClient({ region, credentials })`

**API mapping:**
| Method | AWS API Call |
|--------|-------------|
| `getSecret(name, version?)` | `GetSecretValueCommand({ SecretId: name, VersionId: version, VersionStage: version })` |
| `setSecret(name, value)` | `CreateSecretCommand` (catch `ResourceExistsException`) → `PutSecretValueCommand` |
| `listSecrets()` | `ListSecretsCommand` with pagination |
| `testConnection()` | `ListSecretsCommand({ MaxResults: 1 })` |

**Error mapping:**
- `ResourceNotFoundException` → "Secret '{name}' not found in region '{region}'"
- `AccessDeniedException` → "Permission denied for secret '{name}'. Check IAM policy."
- `DecryptionFailureException` → "Cannot decrypt secret '{name}'. Check KMS permissions."
- Import failure → "Please install @aws-sdk/client-secrets-manager: pnpm add @aws-sdk/client-secrets-manager"

### 4.2 Azure Key Vault (`AzureSecretProvider`)

```typescript
class AzureSecretProvider implements SecretProvider {
  name = "azure";
  
  constructor(config: {
    vaultUrl: string;
    cacheTtlSeconds?: number;
    credentialsFile?: string;
    tenantId?: string;
    clientId?: string;
  })
}
```

**Auth flow:**
1. Dynamically import `@azure/keyvault-secrets` and `@azure/identity`
2. Build credential:
   - If `credentialsFile` → parse JSON, use `ClientSecretCredential(tenantId, clientId, clientSecret)`
   - If `tenantId` + `clientId` + env `AZURE_CLIENT_SECRET` → `ClientSecretCredential`
   - Else → `DefaultAzureCredential()` (env → managed identity → CLI → VS Code)
3. Create `SecretClient(vaultUrl, credential)`

**API mapping:**
| Method | Azure SDK Call |
|--------|---------------|
| `getSecret(name, version?)` | `client.getSecret(name, { version })` |
| `setSecret(name, value)` | `client.setSecret(name, value)` |
| `listSecrets()` | `client.listPropertiesOfSecrets()` (async iterator) |
| `testConnection()` | `listPropertiesOfSecrets().next()` |

**Error mapping:**
- `RestError` with `statusCode: 404` → "Secret '{name}' not found in vault '{vaultUrl}'"
- `RestError` with `statusCode: 403` → "Permission denied. Check Key Vault access policy or RBAC."
- `CredentialUnavailableError` → "Azure credentials not found. Run `az login` or set env vars."
- Import failure → "Please install @azure/keyvault-secrets @azure/identity"

**Azure naming constraint:** Azure Key Vault secret names only allow `[a-zA-Z0-9-]` (no underscores, dots, or slashes). The provider must validate names and give clear errors. Document this limitation.

### 4.3 HashiCorp Vault (`VaultSecretProvider`)

```typescript
class VaultSecretProvider implements SecretProvider {
  name = "vault";
  
  constructor(config: {
    address: string;
    cacheTtlSeconds?: number;
    namespace?: string;
    mountPath?: string;       // default: "secret"
    authMethod?: string;      // "token" | "approle" | "kubernetes"
    token?: string;
    tokenFile?: string;
    roleId?: string;
    secretId?: string;
  })
}
```

**Auth flow:**
1. No SDK dependency — uses native `fetch()` against Vault HTTP API
2. Token acquisition:
   - If `token` → use directly
   - If `tokenFile` → read file contents
   - If `VAULT_TOKEN` env var → use it
   - If `authMethod: "approle"` → `POST /v1/auth/approle/login` with `role_id` + `secret_id` → extract `client_token`
   - If `authMethod: "kubernetes"` → read `/var/run/secrets/kubernetes.io/serviceaccount/token`, `POST /v1/auth/kubernetes/login`
3. Token is cached in-memory; re-auth on 403 for renewable auth methods

**API mapping (KV v2):**
| Method | Vault HTTP Endpoint |
|--------|-------------------|
| `getSecret(name, version?)` | `GET /v1/{mount}/data/{name}?version={version}` → `.data.data` (KV v2 wrapping) |
| `setSecret(name, value)` | `POST /v1/{mount}/data/{name}` body: `{ data: { value } }` |
| `listSecrets()` | `LIST /v1/{mount}/metadata/` → `.data.keys` |
| `testConnection()` | `GET /v1/sys/health` |

**KV v2 data structure:** Vault KV v2 stores arbitrary JSON. We store `{ value: "<secret>" }` and extract the `value` field on read. This keeps it simple and consistent. If the secret has multiple fields, the user references the top-level key name and gets the `value` field.

**Error mapping:**
- HTTP 404 → "Secret '{name}' not found at path '{mount}/data/{name}'"
- HTTP 403 → "Permission denied. Check Vault policy for path '{mount}/data/{name}'."
- HTTP 503 → "Vault is sealed. Unseal before use."
- `ECONNREFUSED` → "Cannot connect to Vault at '{address}'"
- No token available → "No Vault token. Set VAULT_TOKEN, use --token, or configure AppRole."

**Namespace support:** If `namespace` is set, include `X-Vault-Namespace` header in all requests.

---

## 5. Config Schema (Zod)

Extend the existing secrets config schema:

```typescript
import { z } from "zod";

const GcpProviderSchema = z.object({
  project: z.string(),
  cacheTtlSeconds: z.number().int().positive().optional(),
  credentialsFile: z.string().optional(),
});

const AwsProviderSchema = z.object({
  region: z.string(),
  cacheTtlSeconds: z.number().int().positive().optional(),
  profile: z.string().optional(),
  credentialsFile: z.string().optional(),
  roleArn: z.string().startsWith("arn:aws:iam::").optional(),
  externalId: z.string().optional(),
});

const AzureProviderSchema = z.object({
  vaultUrl: z.string().url(),
  cacheTtlSeconds: z.number().int().positive().optional(),
  credentialsFile: z.string().optional(),
  tenantId: z.string().uuid().optional(),
  clientId: z.string().uuid().optional(),
});

const VaultProviderSchema = z.object({
  address: z.string().url(),
  cacheTtlSeconds: z.number().int().positive().optional(),
  namespace: z.string().optional(),
  mountPath: z.string().default("secret"),
  authMethod: z.enum(["token", "approle", "kubernetes"]).default("token"),
  token: z.string().optional(),
  tokenFile: z.string().optional(),
  roleId: z.string().optional(),
  secretId: z.string().optional(),
});

const SecretsConfigSchema = z.object({
  providers: z.object({
    gcp: GcpProviderSchema.optional(),
    aws: AwsProviderSchema.optional(),
    azure: AzureProviderSchema.optional(),
    vault: VaultProviderSchema.optional(),
  }).optional(),
});
```

---

## 6. Factory Extension (`buildSecretProviders`)

The existing factory in `secret-resolution.ts` grows to handle new providers:

```typescript
export function buildSecretProviders(config: SecretsConfig | undefined): Map<string, SecretProvider> {
  const providers = new Map<string, SecretProvider>();
  if (!config?.providers) return providers;

  const { gcp, aws, azure, vault } = config.providers;
  if (gcp)   providers.set("gcp",   new GcpSecretProvider(gcp));
  if (aws)   providers.set("aws",   new AwsSecretProvider(aws));
  if (azure) providers.set("azure", new AzureSecretProvider(azure));
  if (vault) providers.set("vault", new VaultSecretProvider(vault));

  return providers;
}
```

No changes to `resolveConfigSecrets`, `fetchWithCache`, `extractSecretReferences`, or the cache. The dispatch is already provider-agnostic.

---

## 7. CLI Setup Flows

### 7.1 AWS Setup (`openclaw secrets setup --provider aws`)

1. Check `aws` CLI is installed (`aws --version`)
2. Validate region and credentials (`aws sts get-caller-identity`)
3. For each agent:
   - Create IAM policy `openclaw-<agent>-secrets` restricting `secretsmanager:GetSecretValue` to `arn:aws:secretsmanager:<region>:<account>:secret:openclaw-<agent>-*`
   - Create IAM user `openclaw-<agent>` or role, attach policy
   - Generate access keys (if user-based) and store path in config
4. Write `secrets.providers.aws` to `openclaw.json`

### 7.2 Azure Setup (`openclaw secrets setup --provider azure`)

1. Check `az` CLI is installed (`az --version`)
2. Validate login (`az account show`)
3. Create Key Vault (or use existing `--vault-url`)
4. For each agent:
   - Create Azure AD app registration `openclaw-<agent>`
   - Create service principal
   - Assign `Key Vault Secrets User` role scoped to the vault
5. Write `secrets.providers.azure` to `openclaw.json`

### 7.3 Vault Setup (`openclaw secrets setup --provider vault`)

1. Check connectivity (`GET /v1/sys/health`)
2. Check auth (token must have policy management capability)
3. Ensure KV v2 mount exists at `mountPath`
4. For each agent:
   - Create policy `openclaw-<agent>` with `read` on `<mount>/data/openclaw/<agent>/*`
   - Create AppRole `openclaw-<agent>` bound to policy
   - Fetch `role_id` and generate `secret_id`
   - Store in config or output for user
5. Write `secrets.providers.vault` to `openclaw.json`

---

## 8. Migration Path

### 8.1 New Users

`openclaw secrets setup --provider <name>` → `openclaw secrets migrate --provider <name>` — same flow as GCP.

### 8.2 Existing GCP Users Adding Another Provider

No migration needed. Users simply add a new provider to `openclaw.json` and start using `${aws:...}` or `${vault:...}` refs alongside `${gcp:...}`. Multiple providers coexist.

### 8.3 Switching Providers

`openclaw secrets migrate --from gcp --to aws` (future enhancement, out of scope for v1). For now, users manually re-upload secrets and update refs.

### 8.4 From Plaintext

Same `openclaw secrets migrate` flow — scan, upload, replace, verify, purge.

---

## 9. Shared Abstractions

What's common across all providers (already exists or can be shared):

| Concern | Location | Shared? |
|---------|----------|---------|
| `SecretProvider` interface | `secret-resolution.ts` | ✅ Already shared |
| TTL cache + stale-while-revalidate | `fetchWithCache()` in `secret-resolution.ts` | ✅ Already shared |
| `${provider:name#version}` parsing | `SECRET_REF_PATTERN` regex | ✅ Already shared |
| Config tree walking | `resolveAny()` | ✅ Already shared |
| Error types | `SecretResolutionError`, `UnknownSecretProviderError` | ✅ Already shared |
| Secret name validation | — | ❌ **New:** helper to validate name per provider rules |
| Dynamic SDK import + error | — | ❌ **New:** shared `lazyImport(pkg, installHint)` helper |
| CLI mock injection pattern | `_mock*` options | ✅ Already established |

**New shared helpers to create:**

```typescript
// Lazy import with friendly error
async function lazyImport<T>(pkg: string): Promise<T> {
  try {
    return await import(pkg);
  } catch {
    throw new Error(`Please install ${pkg}: pnpm add ${pkg}`);
  }
}

// Provider-specific name validation
function validateSecretName(provider: string, name: string): void {
  if (provider === "azure" && /[^a-zA-Z0-9-]/.test(name)) {
    throw new Error(`Azure Key Vault secret names only allow alphanumeric and hyphens. Got: "${name}"`);
  }
  // AWS and Vault are permissive — no extra validation needed
}
```

---

## 10. Security Considerations

- **No secrets in logs:** All providers must avoid logging secret values. Log secret names and operations at debug level only.
- **Memory:** Secrets are held in the in-memory cache Map. No disk persistence.  
- **Credential files:** If `credentialsFile` is used, warn if file permissions are too open (>0600).
- **Vault token rotation:** For AppRole auth, tokens are renewable. Provider should handle token expiry transparently.
- **AWS STS tokens:** When using `roleArn`, temporary credentials expire (default 1h). Provider should catch `ExpiredTokenException` and re-assume role.

---

## 11. Implementation Order

1. **Refactor:** Extract `GcpSecretProvider` into `providers/gcp-secret-provider.ts` + shared `lazyImport` helper
2. **AWS provider** — most users, straightforward SDK
3. **Vault provider** — no SDK dependency (plain fetch), good for self-hosted
4. **Azure provider** — two SDK packages, more complex auth
5. **CLI setup commands** — one per provider
6. **Integration tests** — optional, credential-gated

Each provider is a standalone PR after the refactor PR. They can be reviewed and merged independently.

---

## 12. Open Questions

1. **Vault KV v1 support?** — v1 has no versioning. Proposal: v2 only for now, document.
2. **AWS SSM Parameter Store?** — Popular alternative to Secrets Manager, cheaper. Defer to separate provider `ssm`?
3. **Secret rotation hooks?** — AWS has built-in rotation lambdas. Out of scope for v1.
4. **`delete` method?** — akoscz's interface has `delete`. Add to our interface? Proposal: yes, add in the refactor PR.
5. **Multi-field Vault secrets?** — Vault KV stores JSON objects. We extract `.value`. Support `${vault:name.field}` syntax later?
