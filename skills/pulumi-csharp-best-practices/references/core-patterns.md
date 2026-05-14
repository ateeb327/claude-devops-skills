# Core Patterns — Pulumi C#

The fundamentals that apply to every Pulumi C# program regardless of provider: `Output<T>` handling, `ComponentResource` design, parent/child relationships, refactoring with aliases, and the preview-before-deploy workflow.

---

## A1. Never Create Resources Inside `Apply()`

**Why**: Resources created inside `Apply()` are not registered with the resource graph at program startup; the engine only sees them once the apply lambda runs, and the lambda only runs when the input output's value is known. In `pulumi preview` (where the value is often unknown), the resource is not recorded and an update plan that didn't record the resource may then fail or behave unexpectedly when the resource shows up at deploy time. The reliable pattern is to keep resource creation at program startup so the graph is complete before preview runs.

**Detection signals**:
- `new Aws.S3.Bucket(...)` or any resource constructor inside a `.Apply(...)` lambda
- Resource creation inside `Output.All(...).Apply(...)`
- Dynamic resource counts determined at runtime inside apply

**Wrong**:
```csharp
var bucket = new Aws.S3.Bucket("bucket");

// WRONG: pulumi preview may not record the BucketObject if bucket.Id is unknown,
// and any update plan derived from such a preview can fail or behave unexpectedly
bucket.Id.Apply(bucketId =>
{
    var obj = new Aws.S3.BucketObject("object", new Aws.S3.BucketObjectArgs
    {
        Bucket = bucketId,
        Content = "hello",
    });
    return bucketId;
});
```

**Right**:
```csharp
var bucket = new Aws.S3.Bucket("bucket");

// RIGHT: Pass the Output<T> directly — Pulumi tracks the dependency
var obj = new Aws.S3.BucketObject("object", new Aws.S3.BucketObjectArgs
{
    Bucket = bucket.Id,   // Output<string> is accepted as Input<string>
    Content = "hello",
});
```

**When `Apply()` IS appropriate**:
- Transforming an output value for use in a tag, a name, or a computed string
- Logging/debugging (never resource creation)
- Conditional logic that affects a resource *property* value, not resource *existence*

**Reference**: https://www.pulumi.com/docs/iac/concepts/inputs-outputs/apply/

---

## A2. Pass `Output<T>` Directly as `Input<T>`

**Why**: Pulumi builds a Directed Acyclic Graph (DAG) from input/output relationships. Passing outputs directly ensures correct creation order and tracks dependencies automatically. Manually unwrapping values (extracting via `Apply()` into a local variable) severs the dependency chain, the resource may deploy before its dependency is ready, or reference a value that does not yet exist.

**Detection signals**:
- A local `string` variable extracted from `.Apply()` used later as a resource input
- String concatenation using `+` with an `Output<string>` instead of `Output.Format()`
- `await` used on an `Output<T>` outside of an async `Apply()` context

**Wrong**:
```csharp
var vpc = new Aws.Ec2.Vpc("vpc", new Aws.Ec2.VpcArgs
{
    CidrBlock = "10.0.0.0/16",
});

// WRONG: Extracting the value breaks the dependency chain
string vpcId = "";
vpc.Id.Apply(id =>
{
    vpcId = id;  // This may never run before subnet is created
    return id;
});

var subnet = new Aws.Ec2.Subnet("subnet", new Aws.Ec2.SubnetArgs
{
    VpcId = vpcId,          // Empty string — no tracked dependency
    CidrBlock = "10.0.1.0/24",
});
```

**Right**:
```csharp
var vpc = new Aws.Ec2.Vpc("vpc", new Aws.Ec2.VpcArgs
{
    CidrBlock = "10.0.0.0/16",
});

// RIGHT: Pass Output<string> directly — implicit conversion to Input<string>
var subnet = new Aws.Ec2.Subnet("subnet", new Aws.Ec2.SubnetArgs
{
    VpcId = vpc.Id,         // Output<string> is accepted as Input<string>
    CidrBlock = "10.0.1.0/24",
});
```

**String interpolation — use `Output.Format()`**:
```csharp
// WRONG: + operator calls ToString() on the Output. The result is a
// diagnostic string (its exact form depends on SDK version), not the
// actual value — concatenation never waits for the Output to resolve.
var name = "prefix-" + bucket.Id + "-suffix";

// WRONG: Apply for simple interpolation is verbose and unnecessary
var name = bucket.Id.Apply(id => $"prefix-{id}-suffix");

// RIGHT: Output.Format() is the idiomatic C# equivalent of pulumi.interpolate
var name = Output.Format($"prefix-{bucket.Id}-suffix");
```

> **Note**: There is no public `Output.Concat(...)` for string concatenation in the .NET SDK. The TypeScript and Python SDKs expose `pulumi.concat` / `Output.concat`; in C# the idiomatic equivalent is `Output.Format(...)`. (An internal `Output.Concat` exists for concatenating `Output<ImmutableArray<T>>` values, that's a different operation and not relevant for string building.)

**Combining multiple outputs: use `Output.All()` or `Output.Tuple()`**:
```csharp
// Output.All — returns Output<ImmutableArray<string>>
var combined = Output.All(vpc.Id, subnet.Id).Apply(values =>
    $"vpc={values[0]}, subnet={values[1]}");

// Output.Tuple — strongly typed, preferred when types differ
var tag = Output.Tuple(bucket.Id, bucket.Arn).Apply(t =>
    $"id={t.Item1}, arn={t.Item2}");
```

**Version note**: `Output.Tuple()` has been part of the .NET SDK since the initial preview (v1.4.1) and is stable across every shipped version. The 8-overload variant (`Output.Tuple<T1...T8>`) was added at v1.13.x; if you're on a very old preview build that predates the 8-overload form, use `Output.All()` with index-based access for cases needing more than 7 values.

**Reference**: https://www.pulumi.com/docs/iac/concepts/inputs-outputs/

---

## A3. Use `ComponentResource` for Related Resources

**Why**: `ComponentResource` subclasses group logically related resources into reusable, named units. Without components, the resource graph is flat. it is hard to understand which resources belong together, difficult to reuse patterns across stacks, and the Pulumi Console shows an unstructured list of resources with no grouping.

**Detection signals**:
- Multiple related resources created at the top level of `Program.cs` or `Stack.cs` without grouping
- Repeated resource creation patterns copy-pasted across stacks
- Hard to navigate resource list in Pulumi Console

**Wrong**:
```csharp
// Flat — no logical grouping, hard to reuse
var bucket = new Aws.S3.Bucket("app-bucket");
var bucketPolicy = new Aws.S3.BucketPolicy("app-bucket-policy", new Aws.S3.BucketPolicyArgs
{
    Bucket = bucket.Id,
    Policy = policyJson,
});
var oai = new Aws.CloudFront.OriginAccessIdentity("app-oai");
var distribution = new Aws.CloudFront.Distribution("app-cdn", new Aws.CloudFront.DistributionArgs
{
    // ...
});
```

**Right**:
```csharp
public class StaticSiteArgs : ResourceArgs
{
    // The [Input] attribute is OPTIONAL for plain C# components. It is REQUIRED
    // if the component will be packaged as a multi-language component or have
    // its inputs surfaced in the Pulumi component schema. For a single-language
    // C# component, leaving it off works fine.
    [Input("domain")]
    public Input<string> Domain { get; set; } = null!;
}

public class StaticSite : ComponentResource
{
    public Output<string> Url { get; private set; }

    public StaticSite(string name, StaticSiteArgs args, ComponentResourceOptions? opts = null)
        : base("myorg:components:StaticSite", name, args, opts)
    {
        // See A4 for why Parent = this is required here
        var bucket = new Aws.S3.Bucket($"{name}-bucket", new Aws.S3.BucketArgs(), new CustomResourceOptions
        {
            Parent = this,
        });

        var distribution = new Aws.CloudFront.Distribution($"{name}-cdn",
            new Aws.CloudFront.DistributionArgs { /* ... */ },
            new CustomResourceOptions { Parent = this });

        this.Url = distribution.DomainName;

        // Always call RegisterOutputs at the end of the constructor
        this.RegisterOutputs(new Dictionary<string, object?>
        {
            { "url", this.Url },
        });
    }
}

// Reusable across stacks
var site = new StaticSite("marketing", new StaticSiteArgs
{
    Domain = "marketing.example.com",
});
```

**Component best practices**:
- Use a consistent type URN format: `"organization:module:ComponentName"` (three colon-separated segments)
- The `args` class **must derive from `ResourceArgs`** so it can be passed to the `ComponentResource(string type, string name, ResourceArgs? args, ComponentResourceOptions? options)` base constructor. `ResourceArgs` itself is `abstract`, you can't instantiate it directly. If your component takes no args, use the 3-argument base constructor `base(type, name, options)` (it internally supplies `ResourceArgs.Empty`)
- `[Input("name")]` attributes on args properties are optional for single-language C# components but required if you want the component to participate in Pulumi's multi-language component schema. Adding them costs nothing and future-proofs the component for packaging
- Always call `this.RegisterOutputs()` at the end of the constructor. this tells Pulumi the component is done registering children
- Expose outputs as public `Output<T>` properties so callers can pass them to other resources
- Accept `ComponentResourceOptions?` so callers can set providers, aliases, etc.

**Reference**: https://www.pulumi.com/docs/iac/concepts/resources/components/

---

## A4. Always Set `Parent = this` Inside Components

**Why**: Without `Parent = this`, child resources appear at the root level of the stack's state, as if they had nothing to do with the component. This breaks the visual hierarchy in the Pulumi Console, breaks alias propagation during refactors, and prevents the component-level options that *do* inherit (provider, protect, transformations, etc.) from reaching the children.

**Detection signals**:
- `ComponentResource` subclass that creates child resources without `new CustomResourceOptions { Parent = this }`
- Child resources appearing at root level in `pulumi stack --show-ids` output
- Unexpected behaviour when adding `Aliases` to a component

**Wrong**:
```csharp
public class MyComponent : ComponentResource
{
    public MyComponent(string name, ComponentResourceOptions? opts = null)
        : base("myorg:components:MyComponent", name, opts)
    {
        // WRONG: No Parent — this bucket appears at root level, unowned
        var bucket = new Aws.S3.Bucket($"{name}-bucket");
    }
}
```

**Right**:
```csharp
public class MyComponent : ComponentResource
{
    public MyComponent(string name, ComponentResourceOptions? opts = null)
        : base("myorg:components:MyComponent", name, opts)
    {
        // RIGHT: Parent = this establishes ownership and hierarchy
        var bucket = new Aws.S3.Bucket($"{name}-bucket", new Aws.S3.BucketArgs(), new CustomResourceOptions
        {
            Parent = this,
        });

        var policy = new Aws.S3.BucketPolicy($"{name}-policy", new Aws.S3.BucketPolicyArgs
        {
            Bucket = bucket.Id,
            Policy = policyJson,
        },
        new CustomResourceOptions
        {
            Parent = this,    // Every direct child resource needs this
        });

        this.RegisterOutputs();
    }
}
```

**What `Parent = this` actually provides** (per Pulumi's parent option docs):
- Resources appear nested under the component in the Pulumi Console
- A subset of resource options **flows down** from parent to child: `provider` / `providers` (provider map), `protect`, `aliases`, `transforms` / `transformations`, and `deletedWith`
- Refactoring the component (e.g., aliasing the component to a new name) automatically propagates to children
- Clear ownership in state files (`.pulumi/stacks/*.json`) and stable URNs incorporating the component as a path segment

**What `Parent = this` does NOT provide**:
- `DependsOn` is **not** inherited from parent. Each child resource declares its own dependencies
- It is not a provider-level cascade-delete mechanism. When the Pulumi engine destroys a component, the engine walks the URN graph and destroys descendants in dependency order — that's a function of the engine's resource model, not a destructor on the parent. Don't treat `Parent = this` as a "delete this and everything under it dies" guarantee outside of Pulumi-managed teardown

**Reference**: https://www.pulumi.com/docs/iac/concepts/resources/options/parent/

---

## A5. Use `Aliases` When Refactoring

**Why**: Renaming a resource, moving it into a component, or changing its parent causes Pulumi to treat it as a new resource, the old one gets deleted and a new one created. For stateful resources (databases, S3 buckets, DNS records, certificates), this means data loss or downtime. `Aliases` tells Pulumi "this resource was previously known by this other name/URN, do not delete it."

**Detection signals**:
- Resource renamed without an alias (preview shows `- delete` + `+ create` for the same resource)
- Resource moved into or out of a `ComponentResource` without alias pointing to old parent
- `+-replace` in preview for a rename that should have been a no-op

**Wrong**:
```csharp
// Before: resource named "my-bucket"
var bucket = new Aws.S3.Bucket("my-bucket");

// After rename — WRONG: Pulumi will destroy the old bucket and create a new one
var bucket = new Aws.S3.Bucket("application-bucket");
```

**Right: simple rename**:
```csharp
// RIGHT: Alias preserves the existing bucket's state identity
var bucket = new Aws.S3.Bucket("application-bucket", new Aws.S3.BucketArgs(), new CustomResourceOptions
{
    Aliases =
    {
        new Alias { Name = "my-bucket" },
    },
});
```

**Right: moving into a component**:
```csharp
// Before: top-level resource
var bucket = new Aws.S3.Bucket("my-bucket");

// After: inside a component — alias signals the previous name AND that there
// was no previous parent (the resource was at stack root before).
public class MyComponent : ComponentResource
{
    public MyComponent(string name, ComponentResourceOptions? opts = null)
        : base("myorg:components:MyComponent", name, opts)
    {
        var bucket = new Aws.S3.Bucket("bucket", new Aws.S3.BucketArgs(), new CustomResourceOptions
        {
            Parent = this,
            Aliases =
            {
                new Alias
                {
                    Name     = "my-bucket",
                    NoParent = true,   // resource was at stack root before this refactor
                },
            },
        });

        this.RegisterOutputs();
    }
}
```

**Alias shapes in C#**:
```csharp
// Simple name change only
new Alias { Name = "old-name" }

// Name change + previous parent was the stack root (no parent)
new Alias { Name = "old-name", NoParent = true }

// Name + parent change (set Parent to the previous parent's resource reference,
// or ParentUrn if you only have the URN string)
new Alias { Name = "resource-name", Parent = oldParentResource }
new Alias { Name = "resource-name", ParentUrn = "urn:pulumi:..." }

// Full URN (when you know the exact previous URN)
new Alias { Urn = "urn:pulumi:stack::project::aws:s3/bucket:Bucket::old-name" }
```

`Alias` is a `sealed class` (not a record). The `Parent`, `ParentUrn`, and `NoParent` properties are mutually exclusive, set exactly one of them.

**Lifecycle**:
1. Add alias during the refactor
2. Run `pulumi up` on all affected stacks
3. Remove the alias after all stacks are updated (keeps code clean; the resource identity is now tracked under the new name)

**Version note**: The `Alias` class has been part of the .NET SDK since v1.x and the `NoParent`, `Parent`, and `ParentUrn` properties are stable. At .NET SDK 3.35.0 the engine handshake changed, the SDK started sending `Alias` objects directly rather than pre-computed URN strings, so behaviour against very old engines may differ. For merging resource options programmatically (e.g., layering options inside a component), use the static methods `CustomResourceOptions.Merge(opts1, opts2)` and `ComponentResourceOptions.Merge(opts1, opts2)`. There is no `MergeOptions()` instance method on `ResourceOptions` despite what some cross-language docs may suggest.

**Reference**: https://www.pulumi.com/docs/iac/concepts/resources/options/aliases/

---

## A6. Preview Before Every Deployment

**Why**: `pulumi preview` shows exactly what will be created, updated, or destroyed, including `replace` operations that swap out a resource for a new one. A resource showing `replace` when you expected `update` means a property requiring replacement was changed. Surprises in production come from skipping preview.

**Detection signals**:
- Running `pulumi up --yes` in interactive sessions without reviewing changes
- No `pulumi preview` step in the CI/CD pipeline for a given branch
- Preview output not reviewed before deployment approval

**Wrong**:
```bash
# Deploying blind
pulumi up --yes
```

**Right**:
```bash
# Always preview first and review the output
pulumi preview

# Then deploy with a manual confirmation gate
pulumi up
```

**What to look for in preview output**:

| Symbol      | Meaning                                                                          | Action                                |
|-------------|----------------------------------------------------------------------------------|---------------------------------------|
| `+ create`  | New resource will be created                                                     | Expected for new code                 |
| `~ update`  | Existing resource modified in place                                              | Usually safe — review property diffs  |
| `- delete`  | Resource will be destroyed                                                       | Verify this is intentional            |
| `+- replace`| Resource is replaced — by default Pulumi creates the new one first and deletes the old one afterwards. With `DeleteBeforeReplace = true` (or for resources with name-uniqueness constraints that force it), the order flips to delete-first. Either way: expect a transition and look for the property that triggered it | Investigate the immutable property change; ensure the order is what you want |

The exact rendering of replace operations can vary slightly across CLI versions, some versions display `++` for create-replacement and `--` for delete-original separately. The substantive thing to look for is any replace operation, regardless of glyph. Run `pulumi preview --diff` to see the property-level reasons that triggered the replace.

**Warning signs in preview**:
- Unexpected `replace` operations, usually caused by changing an immutable property (e.g., bucket name, AZ, instance type that requires replacement)
- Resources being deleted that are not in your diff
- More changes than expected from your code diff (may indicate a provider upgrade changed defaults)

**GitHub Actions example**:
```yaml
jobs:
  preview:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Pulumi Preview
        uses: pulumi/actions@v6
        with:
          command: preview
          stack-name: production
        env:
          PULUMI_ACCESS_TOKEN: ${{ secrets.PULUMI_ACCESS_TOKEN }}

  deploy:
    needs: preview
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    steps:
      - name: Pulumi Up
        uses: pulumi/actions@v6
        with:
          command: up
          stack-name: production
```

**Azure DevOps pipeline example**:
```yaml
stages:
  - stage: Preview
    jobs:
      - job: PulumiPreview
        steps:
          - script: pulumi preview --stack production --diff
            env:
              PULUMI_ACCESS_TOKEN: $(PULUMI_ACCESS_TOKEN)

  - stage: Deploy
    dependsOn: Preview
    condition: and(succeeded(), eq(variables['Build.SourceBranch'], 'refs/heads/main'))
    jobs:
      - deployment: PulumiUp
        environment: production
        strategy:
          runOnce:
            deploy:
              steps:
                - script: pulumi up --yes --stack production
```

**References**:
- https://www.pulumi.com/docs/iac/cli/commands/pulumi_preview/
- https://www.pulumi.com/docs/iac/packages-and-automation/continuous-delivery/github-actions/
