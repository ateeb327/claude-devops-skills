# Secrets Pulumi C#

Practices that apply whenever a Pulumi C# program handles credentials, API keys, or other sensitive values.

---

## B1. Encrypt Secrets from Day One

**Why**: Secrets set with `--secret` are encrypted in state files, masked in CLI output, and remain encrypted through `Apply()` transformations. Converting plaintext config to secrets after the fact requires credential rotation, audit of leaked values in historical state and logs, and updates to all references.

**Detection signals**:
- Passwords, API keys, tokens set as plain `pulumi config set` (no `--secret`)
- Connection strings with embedded credentials stored as non-secret config
- Private keys or certificates stored in plaintext config

**Wrong**:
```bash
# Plaintext — will appear unencrypted in state file and CLI output
pulumi config set databasePassword hunter2
pulumi config set apiKey sk-1234567890
```

**Right**:
```bash
# Encrypted from the start
pulumi config set --secret databasePassword hunter2
pulumi config set --secret apiKey sk-1234567890
```

**In C# code**:
```csharp
var config = new Config();

// RequireSecret returns Output<string> — value stays encrypted through transformations
var dbPassword = config.RequireSecret("databasePassword");

// Output.Format preserves secrecy — connectionString is also a secret Output
var connectionString = Output.Format($"postgres://user:{dbPassword}@host/db");

// Explicitly mark a computed value as secret
var computed = Output.CreateSecret(someComputedValue);
```

**What qualifies as a secret**:
- Passwords and passphrases
- API keys and access tokens
- Private keys and TLS certificates
- Connection strings with embedded credentials
- OAuth client secrets
- Encryption keys and key material

**Version note**: `config.RequireSecret()` is available across all shipped .NET SDK versions. `Output.CreateSecret<T>` has two overloads: the **plain-value overload** `Output.CreateSecret<T>(T value)` has been available across shipped versions, and the **Output overload** `Output.CreateSecret<T>(Output<T> value)` was added in .NET SDK 3.39.0. Both are generic; the distinction is whether the argument is a plain `T` or an `Output<T>`. On older SDKs you can still create a secret Output, but you need to wrap a plain value rather than passing an existing `Output<T>` directly.

**Reference**: https://www.pulumi.com/docs/iac/concepts/secrets/

---

## B2. Choose One Secret Source per Secret, Don't Mix

**Why**: There are three viable secret-management patterns for Pulumi C#, and each works well on its own. Mixing them for the *same* secret causes drift: the value in `pulumi config` no longer matches the value in ESC, which no longer matches what's in the external secret store. When something breaks, you spend hours debugging "which copy is the real one?"

**The three patterns**:

1. **Native Pulumi secrets**: `pulumi config set --secret` + `Config.RequireSecret()` in code.
   - **Best for**: small projects, secrets that vary per stack, getting started fast.
   - **Trade-off**: stack-scoped, hard to share across projects.

2. **Pulumi ESC** (Environments, Secrets, Configuration): referenced from the stack settings file (`Pulumi.<stack>.yaml`, not `Pulumi.yaml`).
   - **Best for**: multi-stack programs sharing credentials, teams needing audit trails, organisations standardising on Pulumi for governance.
   - **Trade-off**: another moving part to set up; vendor lock-in to Pulumi Cloud (or self-hosted ESC).

3. **External secret stores**: SOPS+Age, AWS Secrets Manager, Azure Key Vault, HashiCorp Vault, etc., with values fetched from Pulumi code at deploy time.
   - **Best for**: secrets already managed by another team or tool the org doesn't want to migrate; setups where IaC is not the source of truth for secrets.
   - **Trade-off**: extra IAM/access wiring; secrets are not visible in `pulumi config`.

**Detection signals for mixing**:
- The same logical secret (e.g., `db-password`) referenced from both `config.RequireSecret(...)` and an external lookup
- A secret rotated in one system but not the other
- "Why does production work but staging doesn't?" turning out to be ESC vs. native config drift

**Rule**: Pick one pattern per secret. Document the choice in the project README. If you migrate from one to another, decommission the old source. don't leave both populated.

**Pulumi ESC reference**:

ESC environments are referenced from the **stack settings file** (`Pulumi.<stack>.yaml`), not the project file (`Pulumi.yaml`). For a stack named `production`, this is `Pulumi.production.yaml`:

```yaml
# Pulumi.production.yaml — stack settings file
environment:
  - production-secrets    # Pull from ESC environment at deploy time
```

```bash
esc env set production-secrets db.password --secret "hunter2"
```

**External-store sketch (pattern, not verified code)**:
The general pattern is to fetch the secret in Pulumi program, either by shelling out (`sops -d`) or using the relevant SDK (e.g., `AmazonSecretsManagerClient`), and wrap the resulting value in `Output.CreateSecret(...)` so Pulumi treats it as encrypted in its own state. Confirm the exact API for the secret store and the `Output.CreateSecret` overload against your SDK version before relying on it.

**References**:
- https://www.pulumi.com/docs/iac/concepts/secrets/
- https://www.pulumi.com/docs/esc/guides/integrate-with-pulumi-iac/
- https://www.pulumi.com/docs/iac/concepts/projects/stack-settings-file/
