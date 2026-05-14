---
name: pulumi-csharp-best-practices
version: 2.2.0
description: Load when the user is writing, reviewing, or debugging Pulumi C# programs; asks about Output(T) or Apply() usage; wants to create ComponentResource classes; needs to refactor resources without destroying them (aliases); is setting up secrets or config; or is configuring a pulumi preview/up CI workflow. Also load for questions about resource dependency order, parent/child resource relationships, Output.Format, Output.All, or Output.Concat in C#.
---

# Pulumi C# Best Practices

## About This Skill

These practices come from building and operating a production identity platform on Pulumi C#, covering EKS, Helm-deployed components, DaemonSet workloads, secrets pipelines, and CI/CD on Azure DevOps. Each item corresponds to a failure mode that hit me rather than a theoretical concern lifted from documentation.

The skill is organised so readers can skip what does not apply to their stack:

- **Section A: Core**: applies to every Pulumi C# program, regardless of provider
- **Section B: Secrets**: applies whenever the program handles credentials
- **Section C: Kubernetes & Helm**: applies when Pulumi orchestrates Helm releases or K8s resources directly; safe to skip for AWS/Azure-only stacks
- **Section D: Code Quality for Pulumi**: Pulumi-specific traps that surface through C# style and Roslyn/SonarQube analysis
- **Section E: Verify-First Mindset**: habits for generating Pulumi code without inventing provider-specific values, and knowing when to ask

## When to Use This Skill

Invoke this skill when:

- Writing new Pulumi C# programs or components
- Reviewing Pulumi C# code for correctness
- Refactoring existing Pulumi infrastructure written in C#
- Debugging resource dependency issues in C# stacks
- Setting up configuration and secrets in C#
- Asking about `Output<T>`, `Apply()`, `Output.Format()`, or `Output.All()`
- Integrating Helm releases or Kubernetes resources via Pulumi C#

## Files in This Skill

This SKILL.md is the entry point with brief rules and pointers. Load the reference files below when you need full examples for a specific practice, they are kept separate so common-case queries don't waste context.

- `references/core-patterns.md`: Section A (A1–A6): full examples for `Output<T>`/`Apply`, `ComponentResource`, parent/child, aliases, preview workflow
- `references/secrets.md`: Section B (B1–B2): full examples for secrets and the three secret-source patterns
- `references/kubernetes-helm.md`: Section C (C1–C3): full examples for Helm value selection, DaemonSet tolerations, mutable input collections
- `references/code-quality.md`: Section D (D1–D4): full examples for member ordering, output docs, nullable `InputList`, comment style
- `references/version-reference.md`: full .NET SDK version table and version-detection workflow
- `references/validation-status.md`: what's been verified vs. approximate; how to extend or correct the skill

## Pre-Flight: Version Awareness Protocol

Do **not** ask the user for their version upfront every time. Follow this decision flow instead.

### Step 1: Try to infer the version without asking

Before doing anything else, check what is already available: pasted code, project files (`*.csproj`, `dotnet list package`, `pulumi version`), or anything the user already mentioned. See `references/version-reference.md` for the full list of inference signals.

### Step 2: Decide if the version even matters for this query

Many questions do not require knowing the version at all. Skip version resolution entirely for:

- General best-practice questions (e.g., "should I use Apply to create resources?")
- Code structure questions (e.g., "how do I organise my ComponentResource?")
- Questions where the answer is identical across all supported SDK versions
- Questions about `pulumi preview` / `pulumi up` CLI behaviour
- Secret / config patterns that have been stable since 2.x

**Only resolve version when the question touches a version-sensitive API**, such as:
- `Output.JsonSerialize` (.NET SDK 3.50.0+); `Output.JsonDeserialize` exists in current SDKs but its exact cutoff version is not separately marked — verify against `pulumi-dotnet` history if it matters
- `Output.Unsecret` / `Output.IsSecretAsync` (.NET SDK 2.17.1+)
- `Output.CreateSecret<T>(Output<T>)` overload (.NET SDK 3.39.0+); the plain-value overload `Output.CreateSecret<T>(T)` is available across shipped versions
- `InputMap` / `InputList` initialization with implicitly-convertible values (.NET SDK 3.21.0+)
- Provider resource args or property renames (common across major provider bumps, e.g., `Pulumi.Aws` v5 → v6)
- `Alias` semantics changes, the engine-side handling shifted at .NET SDK 3.35.0; behaviour against very old engines may differ
- Any method the user says "doesn't exist" or "can't be found" is likely a version mismatch

### Step 3: Resolve the version when needed

If you still don't know the version and it matters, in order: (a) search project files via shell if available, (b) infer from pasted code, (c) ask the user with a specific named method whose availability you're checking. Never ask for all versions at once unless you genuinely need them all.

### Step 4: Search docs against the resolved version

Search Pulumi documentation for the specific method or class against the user's version. Flag any deprecations, renames, or breaking changes.

Full inference signals, search patterns, and the version timeline are in `references/version-reference.md`.

---

## Section A: Pulumi C# Core

Full examples and rationale: `references/core-patterns.md`

### A1. Never Create Resources Inside `Apply()`

**Rule**: Create resources at program startup. The graph must be complete before preview runs. Pass `Output<T>` values directly to dependent resource args; use `Apply()` only for transforming a value, not for creating resources.

**When `Apply()` IS appropriate**: transforming an output for a tag/name/computed string, logging, or conditional logic that affects a property value (not resource existence).

### A2. Pass `Output<T>` Directly as `Input<T>`

**Rule**: Don't unwrap `Output<T>` into a local `string` and then use the local in a resource arg that severs the dependency chain. Pulumi accepts `Output<T>` directly wherever `Input<T>` is expected.

**For string interpolation**, use `Output.Format($"prefix-{output}-suffix")`. There is no public `Output.Concat(...)` for strings in the .NET SDK; TS/Python's `pulumi.concat` / `Output.concat` doesn't have a C# equivalent — `Output.Format` is the idiomatic option.

**For combining multiple outputs**, use `Output.All(...)` (returns `Output<ImmutableArray<T>>`) or `Output.Tuple(...)` (strongly typed, preferred when types differ).

### A3. Use `ComponentResource` for Related Resources

**Rule**: Group related resources into a `ComponentResource` subclass. The `args` class **must derive from `ResourceArgs`** (which is `abstract` — use `ResourceArgs.Empty` or the 3-argument `base(type, name, options)` overload when the component takes no args). Always call `this.RegisterOutputs()` at the end of the constructor. Expose outputs as public `Output<T>` properties.

**Optional**: add `[Input("name")]` attributes on args properties if the component might be packaged for multi-language use; for single-language C# components they're not required.

### A4. Always Set `Parent = this` Inside Components

**Rule**: Every child resource inside a `ComponentResource` constructor needs `new CustomResourceOptions { Parent = this }`. Without it, children appear at stack root and component-level options don't reach them.

**What `Parent` inherits** (per the official `parent` option docs): `provider` / `providers`, `protect`, `aliases`, `transforms` / `transformations`, `deletedWith`. **What it does NOT inherit**: `dependsOn` is per-resource. `Parent` is also not a provider-level cascade-delete mechanism, destruction is engine-driven, not destructor-driven.

### A5. Use `Aliases` When Refactoring

**Rule**: Renaming a resource, moving it into a component, or changing its parent causes Pulumi to destroy-and-recreate by default. Add `new Alias { Name = "old-name" }` (or `NoParent = true` when moving from stack root into a component) to preserve identity.

**Alias shape**: `Alias` is a `sealed class` with `Name`, `Type`, `Stack`, `Project`, `Urn`, `Parent`, `ParentUrn`, and `NoParent` properties. `Parent`, `ParentUrn`, and `NoParent` are mutually exclusive — set exactly one of them. For programmatic option merging inside a component, use the static `CustomResourceOptions.Merge(opts1, opts2)` and `ComponentResourceOptions.Merge(opts1, opts2)`. There is no `ResourceOptions.MergeOptions()` instance method.

### A6. Preview Before Every Deployment

**Rule**: Always run `pulumi preview` before `pulumi up`. Review for any `replace` operations. By default, Pulumi creates the new resource first and deletes the old one afterwards, but with `DeleteBeforeReplace = true` (or for resources with name-uniqueness constraints) the order flips to delete-first. Either way, replace means a transition; investigate the immutable property that triggered it.

**CI examples** for GitHub Actions (`pulumi/actions@v6`) and Azure DevOps are in `references/core-patterns.md`.

---

## Section B: Secrets

Full examples and the SOPS / ESC / native trade-off discussion: `references/secrets.md`

### B1. Encrypt Secrets from Day One

**Rule**: Use `pulumi config set --secret` for sensitive values from the start. In code, `config.RequireSecret("key")` returns `Output<string>` that stays encrypted through transformations. `Output.Format` preserves secrecy. Use `Output.CreateSecret(value)` to explicitly mark a computed value as secret.

**`CreateSecret` overloads**: the **plain-value** overload `Output.CreateSecret<T>(T value)` is available across shipped versions; the **Output overload** `Output.CreateSecret<T>(Output<T> value)` was added in .NET SDK 3.39.0. Both are generic, the distinction is plain `T` vs. `Output<T>` argument.

### B2. Choose One Secret Source per Secret — Don't Mix

**Rule**: Pick one of (a) native Pulumi secrets via `pulumi config set --secret`, (b) Pulumi ESC referenced from the **stack settings file** (`Pulumi.<stack>.yaml`, not `Pulumi.yaml`), or (c) an external secret store (SOPS, AWS Secrets Manager, Vault) per secret. Mixing causes drift between sources.

---

## Section C: Pulumi C# + Kubernetes / Helm

Skip this section if your stack is not K8s-related. Full examples: `references/kubernetes-helm.md`

### C1. Don't Ship Helm Values Meant for a Different Deployment Model

**Rule**: For every Helm value, check (1) does it apply to `helm install` (Pulumi's mode) vs. GitOps server-side apply, (2) does it apply to your cluster's actual setup (autoscaler choice, CNI, secret tooling), (3) will the chart silently ignore it if the answer is no? If you can't answer all three, don't add the flag.

### C2. DaemonSet Toleration Coverage

**Rule**: For DaemonSets specifically, enumerate every unique `(key, value, effect)` taint tuple across every node group (managed, self-managed, Karpenter NodePools, Fargate) and add a matching toleration for each. Non-DaemonSet workloads only need tolerations for the node groups they target. When a new node group is added to the cluster, every DaemonSet needs review — put this on the new-node-group runbook.

### C3. `InputMap<object>` Is Mutable — Extract Helpers, Don't Const

**Rule**: Don't share a `static readonly InputMap<object>` across multiple resources. `InputMap`/`InputList` are mutable reference types — sharing one instance means mutation in one place affects every consumer. Extract a `static` helper method that returns a fresh instance on each call. `const` doesn't compile for these types at all.

---

## Section D: Code Quality for Pulumi

Full examples: `references/code-quality.md`

### D1. Member Ordering in `ComponentResource` Classes

**Rule**: Constants → fields → constructor → public properties → private helpers. Field types must match what gets assigned: if `args.Namespace` is `Input<string>`, the field must also be `Input<string>` (not `string`) or the assignment won't compile.

### D2. Public `Output<T>` Properties: XML Doc Starts with "Gets"

**Rule**: Roslyn SA1623. `<summary>Gets the X.</summary>` not `<summary>The X.</summary>`.

### D3. Don't Assign `null` to `InputList<Resource>`

**Rule**: Use `new InputList<Resource>()` (empty) instead of `null` for `DependsOn` and other `InputList` fields. `null` trips CS8601 and may throw at runtime on older SDKs.

### D4. Comment the *Why*, Not the *What*

**Rule**: `// Create namespace` above `new Namespace()` is useless. Comments should capture provider quirks, deliberate ordering choices, residency requirements, or cluster constraints, context the code can't carry on its own.

---

## Section E: Verify-First Mindset

These two practices govern how you use everything above: verify what you can before assuming, and ask when verification isn't possible.

### E1. Don't Invent Provider-Specific Values

**Why**: Generated infrastructure code that contains invented values such as labels, taints, ARNs, AMI IDs, security group IDs, node-group names, k8s annotations, fails silently in confusing ways. A pod scheduled with a non-existent toleration sits Pending. An IAM policy referencing a non-existent role gets created but does nothing. The error surfaces hours after the deploy, far from the cause.

**The fix**: Before writing any provider-specific value, do one of:
1. **Read the actual codebase or cluster state**:
   - `kubectl get nodes -o jsonpath='{.items[*].spec.taints}'` for node taints
   - `kubectl get nodepools.karpenter.sh -o yaml` for Karpenter setups
   - `aws iam get-role --role-name <name>` for ARNs
   - Pulumi state itself: `pulumi stack output` for outputs of a previous deploy
2. **Ask the user**: "What are the taint keys on your system node group?" is a 5-second question that prevents 5-hour debugging
3. **Use a placeholder and call it out**: `// TODO: replace <ROLE_ARN> with actual ARN — verify with team` and *don't ship the code* without resolving the placeholder

**Common invention failures**:
- **EKS node-group labels**: `eks.amazonaws.com/nodegroup=<name>` exists for managed node groups, but the actual selector your cluster uses may be `karpenter.sh/nodepool=<name>` or a team-defined label. Don't assume
- **"Standard" taints**: `CriticalAddonsOnly=true:NoSchedule` is a common convention but is *not* guaranteed to exist in any given cluster
- **AMI IDs**: Hardcoded IDs are region-specific and rotate over time; resolve via SSM parameter or data source
- **IAM role names**: AWS managed-policy ARNs are stable, but custom role names are project-specific

This rule is the single biggest reason generated IaC code "looks right but doesn't work."

### E2. Stop and Ask When Something Is Unclear

**Why**: The cost of asking a quick question is one round-trip. The cost of guessing wrong on Pulumi code can be a deployed resource pointing at the wrong account, an alias that matches nothing, a security group rule on the wrong port, or a refactor that destroys state. The asymmetry is steep, "ask when uncertain" is almost always the right call.

**When you should ask**:
- A request mentions a resource the user assumes is known to you ("the prod cluster", "our standard VPC") but its identity isn't established in the conversation
- Multiple plausible interpretations exist for an instruction
- The user's intent is clear but a specific value is missing, such as region, account, naming convention, taint key — and inventing it would silently break things
- An SDK or provider version matters and the version isn't given (try inference first; ask only when inference fails)
- The "correct" answer depends on a constraint the user is more likely to know than to derive (compliance, internal naming standard, existing IAM topology)

**When NOT to ask**:
- The answer is in code, files, or context the user already provided, re-read first
- The choice is yours to make (style, ordering, idiomatic phrasing), make it
- Multiple reasonable defaults exist and any would work, pick one and explicitly call out the choice so the user can override

**How to ask**: be specific ("Which cluster, staging or production?" beats "More details?"), bound the question with the candidates you've considered ("I see two plausible meanings, A or B, which?"), and don't stack three questions when one will do.

**High-payoff questions for Pulumi C# specifically**: region / account / AWS profile, cluster identity, existing resource names when adding `Aliases` (precision matters), Helm chart version, provider major version mid-upgrade, whether a refactor should preserve state or replace.

**The bar**: if a wrong assumption would be hard for the user to detect later (silent scheduling failure, alias miss, IAM scope creep), prefer to ask. If a wrong assumption is loud (compile error, immediate exception), make a reasonable choice and proceed.

---

## C# ↔ TypeScript / Python Quick Mapping

When reading Pulumi docs that show TypeScript or Python examples, use this mapping:

| Concept                  | TypeScript                          | Python                              | C#                                              |
|--------------------------|-------------------------------------|-------------------------------------|-------------------------------------------------|
| Apply a transform        | `output.apply(v => ...)`            | `output.apply(lambda v: ...)`       | `output.Apply(v => ...)`                        |
| String interpolation     | `pulumi.interpolate\`...\``         | `Output.concat(...)`                | `Output.Format($"...")`                         |
| Concatenation            | `pulumi.concat(...)`                | `Output.concat(...)`                | `Output.Format($"...")` (no public `Output.Concat` for strings — see A2) |
| Combine multiple outputs | `pulumi.all([a, b]).apply(...)`     | `Output.all(a, b).apply(...)`       | `Output.All(a, b).Apply(...)` or `Output.Tuple(a, b).Apply(...)` |
| Serialize JSON           | `pulumi.jsonStringify(obj)`         | `Output.json_dumps(obj)`            | `Output.JsonSerialize(Output.Create(obj))` (.NET SDK 3.50+; takes `Output<T>`) |
| Mark as secret           | `pulumi.secret(v)`                  | `Output.secret(v)`                  | `Output.CreateSecret(v)`                        |
| Require secret config    | `config.requireSecret("k")`         | `config.require_secret("k")`        | `config.RequireSecret("k")`                     |
| Component base class     | `extends pulumi.ComponentResource`  | `pulumi.ComponentResource`          | `class Foo : ComponentResource`                 |
| Register outputs         | `this.registerOutputs({...})`       | `self.register_outputs({...})`      | `this.RegisterOutputs(new Dictionary<...>())`   |
| Resource options         | `{ parent: this, aliases: [...] }`  | `ResourceOptions(parent=self, ...)` | `new CustomResourceOptions { Parent = this, ... }` |

---

## Quick Reference

| Practice              | Key Signal in C#                                    | Fix                                                    |
|-----------------------|-----------------------------------------------------|--------------------------------------------------------|
| No resources in Apply | `new Aws.X.Y(...)` inside `.Apply(v => ...)` lambda | Move resource outside; pass `Output<T>` directly       |
| Pass outputs directly | Local `string` extracted from Apply used as input   | Use `Output<T>` directly; use `Output.Format()` for strings |
| Use components        | Flat `Program.cs` with repeated patterns            | Create `ComponentResource` subclass with `args : ResourceArgs` |
| Set Parent = this     | Component children missing `Parent = this`          | Add `new CustomResourceOptions { Parent = this }`      |
| Aliases on refactor   | Rename causes `- delete` + `+ create` in preview    | Add `new Alias { Name = "old-name" }` to options       |
| Preview before deploy | `pulumi up --yes` without prior preview             | Always run `pulumi preview` first                      |
| Secrets from day one  | Plaintext passwords in `pulumi config set`          | Use `--secret` flag; `config.RequireSecret()` in code  |
| One secret source     | Same secret in `config` AND ESC AND external store  | Pick one; decommission the others                      |
| ESC config location   | ESC `environment:` in `Pulumi.yaml`                 | Move to `Pulumi.<stack>.yaml` (stack settings file)    |
| Helm values fit model | `validationFailurePolicy=Fail` with Pulumi Helm SDK | Drop GitOps-only flags; check chart README per value   |
| DaemonSet tolerations | DS tolerations < unique cluster taints              | Tolerate every (key, value, effect) across all groups  |
| InputMap helpers      | `static readonly InputMap<object>` shared across releases | Extract a `static` method that returns a fresh instance |
| Verify-first          | Hardcoded ARN, label, or taint with no source       | Read actual state; ask the user; or block on TODO      |
| Ask when unclear      | Ambiguous request, missing identity, missing version | Ask one bounded, specific question before generating   |

---

## Validation Checklist

When reviewing Pulumi C# code, verify:

**Core (Section A)**
- [ ] No resource constructors inside `.Apply()` callbacks
- [ ] `Output<T>` passed directly to dependent resource args
- [ ] `Output.Format($"...")` used for string interpolation
- [ ] Related resources grouped in `ComponentResource` subclasses with args deriving from `ResourceArgs`
- [ ] Every child inside a component has `new CustomResourceOptions { Parent = this }`
- [ ] `this.RegisterOutputs()` called at end of every `ComponentResource` constructor
- [ ] Refactored resources have `Aliases` preserving prior identity
- [ ] Deployment process includes a `pulumi preview` step

**Secrets (Section B)**
- [ ] Sensitive values use `config.RequireSecret()` or `--secret` flag from day one
- [ ] Each secret has exactly one source of truth (native, ESC, or external — not mixed)
- [ ] ESC references live in `Pulumi.<stack>.yaml`, not `Pulumi.yaml`

**Kubernetes / Helm (Section C, if applicable)**
- [ ] Every Helm value justified for the deployment model and the actual cluster setup
- [ ] DaemonSet tolerations cover every unique taint across every node group
- [ ] No `static readonly InputMap<object>` shared across multiple resources

**Code Quality (Section D)**
- [ ] `ComponentResource` classes follow constants → fields → constructor → properties → methods order
- [ ] Field types match assigned values (e.g., `Input<string>`, not `string`, when storing `Input<string>` args)
- [ ] Public output properties have XML doc comments starting with "Gets"
- [ ] No `null` assigned to `InputList<Resource>`, empty collection used instead

**Mindset (Section E)**
- [ ] No invented provider-specific values; every taint, label, ARN, and ID has a verified source (E1)
- [ ] When the request was ambiguous, a clarifying question was asked before generating code (E2)
- [ ] Pulumi SDK and provider versions confirmed; docs searched for deprecations on any version-sensitive API used

---

## Validation Status (summary)

The technical claims in this skill are verified against `pulumi/pulumi` and `pulumi/pulumi-dotnet` git history and against direct inspection of the SDK source. A small number of items are explicitly flagged as approximate. The full breakdown, what was verified, what's approximate, and the methodology, is in `references/validation-status.md`.

## Related Skills

The following skills are planned for the same repository but not yet published:

- `pulumi-component-deep-dive` authoring `ComponentResource` classes for distribution, args interface design, multi-language packaging
- `pulumi-automation-api-csharp` programmatic stack orchestration with the Automation API in C#
- `pulumi-esc-patterns` centralised secrets and configuration management with Pulumi ESC

See https://github.com/ateeb327/claude-devops-skills for the current set.
