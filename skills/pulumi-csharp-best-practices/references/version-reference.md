# Version Reference: Pulumi .NET SDK

This file holds the full .NET SDK version timeline and the "two version spaces" explainer. The main `SKILL.md` carries the Pre-Flight decision protocol; load this file when a version-sensitive API or compatibility question comes up.

## Two version spacesL read this first

The .NET SDK and the core Pulumi engine have **different version numbers** since late 2022. Before that, they shipped together in `pulumi/pulumi` and shared a version. The .NET SDK was extracted into the separate `pulumi/pulumi-dotnet` repo around the time of core release v3.50.0 (December 2022); after that, the two version spaces evolved independently.

What this means in practice:

- **`pulumi version`** shows the core engine/CLI version (in the 3.230s at the time this skill was published)
- **`dotnet list package` for the `Pulumi` package** shows the .NET SDK NuGet version (in the 3.100s at the time this skill was published)
- These can differ by 100+ minor versions that is normal, not a misconfiguration. A core CLI release-note line like "upgrade dotnet to 3.103.0" is a routine bump of the .NET SDK reference, not a version mismatch

Methods like `Output.Format`, `Output.Tuple`, `Output.JsonSerialize`, etc. live in the .NET SDK, so the version that matters is the .NET SDK package version.

## Version quick-reference table (.NET SDK)

Verified against `pulumi/pulumi` and `pulumi/pulumi-dotnet` git history. Re-verify against the Pulumi changelog before relying on these for compatibility decisions on uncommon edge cases.

| .NET SDK version | Key change |
|------------------|------------|
| v1.4.1           | Initial .NET Core preview support (#3399). `Output.Format` (FormattableString-based interpolation), `Output.All`, `Alias` class, basic `Output.Tuple` all are present from this point |
| v1.13.x          | `Output.Tuple` extended to support up to 8 type parameters (#3471) |
| v2.0.0           | .NET API marked as shipped, first stable GA release |
| v2.17.1          | `Output.Unsecret<T>(Output<T>)` and `Output.IsSecretAsync<T>(Output<T>)` added |
| v3.21.0          | `InputMap` and `InputList` accept any value with implicit conversion to the collection type; full alias set computation when both parent and child are aliased |
| v3.23.0          | `Output<T>.ToString()` returns an informative message instead of `Output\`1[T]` |
| v3.35.0          | .NET SDK sends `Alias` objects (not URNs) to the engine |
| v3.39.0          | `Output.CreateSecret<T>(Output<T> value)` overload added |
| v3.47.0          | `DictionaryInvokeArgs` for dynamically constructing invoke input |
| v3.50.0          | `Output.JsonSerialize<T>(Output<T>)` added (uses `System.Text.Json`). `Output.JsonDeserialize<T>(Output<string>)` is also present in current SDKs but its exact introduction commit is not separately marked, verify against `pulumi-dotnet` history if the version cutoff matters |
| ~v3.50           | Around this time the .NET SDK was extracted to `pulumi/pulumi-dotnet`. From here on the .NET package version is independent of core Pulumi |

For .NET SDK changes after the split, check `pulumi/pulumi-dotnet`'s `CHANGELOG.md` directly: https://github.com/pulumi/pulumi-dotnet/blob/main/CHANGELOG.md

## Inferring the version from pasted code

When the user hasn't stated their version, look for these signals in their code before asking:

- `Output.Format(...)` and `Output.Tuple(...)` are present in any shipped .NET SDK (v2.0.0 onward), neither narrows the version much
- `Output.Unsecret(...)` or `Output.IsSecretAsync(...)` present → .NET SDK 2.17.1 or later
- `Output.CreateSecret<T>(Output<T>)` overload usage (vs. the plain-value overload) → .NET SDK 3.39.0 or later
- `Output.JsonSerialize(...)` present → .NET SDK 3.50.0 or later
- `new Alias { NoParent = true }` syntax → the `Alias` class has been stable since v1.x; this property works on any shipped version
- Provider namespace (e.g., `Pulumi.Aws.` vs `Aws.`) can hint at provider major version, which has its own lifecycle separate from the SDK

## When the version is unknown and matters

Resolve in this order:

1. Try to find the version from the conversation, pasted `.csproj` snippets, or earlier mentions
2. If you have shell or file access: `cat *.csproj | grep PackageReference`, `dotnet list package`, `pulumi version`
3. Infer from the provider namespace or import style in pasted code
4. As a last resort, ask the user for the specific thing you need — name the method whose availability you're checking:
   > "I need your `Pulumi` NuGet package version to confirm whether `Output.JsonSerialize` is available in your setup. You can find it with `dotnet list package` inside your project directory."

Never ask for all versions at once unless you genuinely need them all.

## When the version is known and matters

Search Pulumi documentation for the specific method or class against that version:

- Search pattern: `site:pulumi.com/docs <method name> csharp`
- .NET SDK changelog: https://github.com/pulumi/pulumi-dotnet/blob/main/CHANGELOG.md
- Core Pulumi changelog (for pre-split history): https://github.com/pulumi/pulumi/blob/master/CHANGELOG.md
- Provider-specific changelog: check releases on the provider's GitHub repo for the user's version range

Flag any deprecations, renames, or breaking changes found.
