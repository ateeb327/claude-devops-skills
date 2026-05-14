# Validation Status

This file tracks what has and has not been independently verified for this skill. Items here are honest about the evidence behind each claim so readers can trust what they read and re-check what's flagged as approximate.

## Verified against `pulumi/pulumi` and `pulumi/pulumi-dotnet` git history

- All version numbers in the `references/version-reference.md` quick-reference table (cross-referenced against git tags, commit history, and changelog entries) **except** the `JsonDeserialize` introduction version, which is approximate (see "Approximate" below)
- `Output.Tuple` has been in the .NET SDK since the initial preview (v1.x); 8-overload form added in v1.13.x
- `Output.Format` has been in the .NET SDK since the initial preview (v1.4.1)
- `Output.Unsecret<T>(Output<T>)` and `Output.IsSecretAsync<T>(Output<T>)` (note: `IsSecretAsync`, not `IsSecret`) added in v2.17.1
- `InputMap`/`InputList` flexible initialization landed in v3.21.0
- `Output.CreateSecret<T>(Output<T>)` overload added in v3.39.0. The plain-value overload `Output.CreateSecret<T>(T value)` has been available across shipped versions
- `Output.JsonSerialize<T>(Output<T>)` added in v3.50.0 (verified via commit `9484c15d2`)
- `Alias` is a `sealed class` with `Name`, `Type`, `Stack`, `Project`, `Urn`, `Parent`, `ParentUrn`, and `NoParent` properties
- `ResourceArgs` is `abstract`; the canonical "no args" sentinel is `ResourceArgs.Empty` (or use the 3-argument `ComponentResource(type, name, options)` constructor overload which supplies it internally)
- The C# SDK has `CustomResourceOptions.Merge(opts1, opts2)` and `ComponentResourceOptions.Merge(opts1, opts2)` as static methods. There is no `ResourceOptions.MergeOptions()` instance method that name appears nowhere in the SDK source
- There is no public `Output.Concat(...)` for string concatenation in the .NET SDK. The internal `Output.Concat<T>(Output<ImmutableArray<T>>, Output<ImmutableArray<T>>)` exists for combining array outputs but is not the same operation; for string building, use `Output.Format(...)`
- The .NET SDK was extracted from `pulumi/pulumi` to `pulumi/pulumi-dotnet` around the time of core release v3.50.0 (December 2022); the two version spaces have been independent since
- Parent option inheritance (per `Deployment_Prepare.cs` and the official `parent` option docs): `provider`/`providers`, `protect`, `aliases`, `transforms`/`transformations`, and `deletedWith` flow down from parent to child. `dependsOn` is **not** inherited

## Verified by reading the SDK source directly

- All Pulumi C# code patterns shown in the practice files use methods, properties, and resource options that exist in the current .NET SDK (verified by inspection of `sdk/Pulumi/` in `pulumi/pulumi-dotnet`)
- The `NoParent = true` alias pattern is documented in the SDK source (`Alias.cs`) itself

## Approximate / depends on user environment

- `Output.JsonDeserialize<T>(Output<string>)` is present in the current SDK but its exact introduction version was not separately tagged in the changelog. Treat the v3.50.0 entry as authoritative for `JsonSerialize` only; for `JsonDeserialize`, verify against `pulumi-dotnet` history if the cutoff matters for your use case
- The exact extraction commit / version where the .NET SDK left `pulumi/pulumi` is approximate ("around v3.50.0, late 2022"). The two-version-spaces consequence is the substantive point; the precise pivot version is a footnote
- Pulumi CLI behaviour examples (`pulumi preview`, `pulumi up` output formatting) are based on stable CLI behaviour; minor cosmetic differences in glyphs (`+- replace` vs. `++ create-replacement` etc.) appear across CLI versions. The default replacement order (create-new-then-delete-old) is per the official docs but can be flipped with `DeleteBeforeReplace = true` or by resources with name-uniqueness constraints
- Helm chart value examples in the kubernetes-helm reference are illustrative; actual chart values depend on the specific chart and its version
- Provider-specific (AWS, Azure, Kubernetes) resource args examples should be checked against the relevant provider's docs for the user's provider version
- GitHub Actions and Azure DevOps pipeline snippets reflect the action versions current at publication (`pulumi/actions@v6`); future major versions may change

## Outside the scope of this skill (not verified, not claimed)

- Full coverage of every `Output<T>` helper method, only the most commonly-used ones are documented
- Multi-language Pulumi component authoring (covered by `pulumi-component-deep-dive` once published)
- Pulumi Automation API in C# (covered by `pulumi-automation-api-csharp` once published)

## How this skill was validated

1. Read `pulumi/pulumi` CHANGELOG.md and git history for all version claims up to the December 2022 SDK extraction
2. Read `pulumi/pulumi-dotnet` CHANGELOG.md and git history for version claims after the extraction
3. Inspected `sdk/Pulumi/` source directly for type definitions (`ResourceArgs`, `Alias`, `ComponentResource`, `Output`) and method signatures
4. Cross-referenced the `parent` option inheritance behaviour against `Deployment_Prepare.cs` to confirm which options actually propagate
5. Code examples were composed against the verified API surface rather than copied from secondary documentation

Future contributors changing technical claims in this skill should add the corresponding verification step to this file or flag the claim as "Approximate" if it can't be directly verified.
