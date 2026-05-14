# Code Quality for Pulumi

C# code quality rules that come up specifically in Pulumi programs. Generic SonarQube/Roslyn rules (unused locals, repeated string literals, etc.) are not duplicated here.

---

## D1. Member Ordering in `ComponentResource` Classes

`ComponentResource` classes get read top-down by anyone trying to understand a component's contract. A consistent ordering makes that scan fast:

1. Constants (`private const`)
2. Fields (`private readonly`)
3. Constructor
4. Public properties (the component's outputs)
5. Private helper methods

The constructor is where the resources get created and the outputs get assigned. Putting properties above the constructor forces a reader to scroll past them to see how they're populated. Putting helpers between the constructor and the properties hides the contract.

```csharp
public class MyComponent : ComponentResource
{
    // 1. Constants
    private const string ComponentType = "myorg:components:MyComponent";

    // 2. Fields — type must match what gets assigned. If args.Namespace is
    // Input<string>, the field must also be Input<string> (not string),
    // otherwise the assignment won't compile.
    private readonly Input<string> _namespace;

    // 3. Constructor
    public MyComponent(string name, MyComponentArgs args, ComponentResourceOptions? opts = null)
        : base(ComponentType, name, args, opts)
    {
        _namespace = args.Namespace;
        // ... resource creation, output assignment ...
        this.RegisterOutputs();
    }

    // 4. Public outputs
    public Output<string> ServiceUrl { get; private set; } = null!;

    // 5. Private helpers
    private static InputMap<object>[] CreateDefaultTolerations() => /* ... */;
}
```

A common mistake is declaring `private readonly string _namespace;` and then assigning `args.Namespace` (which is `Input<string>`) to it. The implicit conversion goes from `string` → `Input<string>`, not the other way around, the assignment fails. Match the field type to the args property type, or if you genuinely need a `string` you have to extract it through `Apply()` (and then you're back in the situation A2 warns against). Almost always: use `Input<string>` for the field.

---

## D2. Public `Output<T>` Properties: XML Doc Starts with "Gets"

Roslyn analyser rule SA1623 enforces this for any public property with documentation. For `ComponentResource` outputs, the docs are the contract, they tell consumers what each output means:

```csharp
// WRONG
/// <summary>The Kubernetes namespace where the component is installed.</summary>
public Output<string> Namespace { get; private set; }

// RIGHT
/// <summary>Gets the Kubernetes namespace where the component is installed.</summary>
public Output<string> Namespace { get; private set; }
```

If the property has a public setter (rare for component outputs, but possible for arg types):
```csharp
/// <summary>Gets or sets the namespace name.</summary>
public Input<string> Namespace { get; set; } = null!;
```

---

## D3. Don't Assign `null` to `InputList<Resource>`

`InputList<T>` is a non-nullable collection type. Assigning `null` to a `DependsOn` field (or any other `InputList`) trips CS8601 in nullable-aware code and can cause runtime null-ref errors in older code. Use an empty collection literal instead.

```csharp
// WRONG — triggers CS8601, may throw at runtime in older SDKs
var opts = new CustomResourceOptions
{
    DependsOn = condition ? new InputList<Resource> { someResource } : null,
};

// RIGHT — empty list when there are no dependencies
var opts = new CustomResourceOptions
{
    DependsOn = condition ? new InputList<Resource> { someResource } : new InputList<Resource>(),
};
```

---

## D4. Comment the *Why*, Not the *What*

`// Create namespace` above `new Namespace()` adds zero information. Save comments for context the code can't carry on its own:

```csharp
// USELESS
// Create the bucket
var bucket = new Aws.S3.Bucket("data");

// USEFUL
// Bucket lives in eu-central-1 specifically because the data residency policy
// requires EU-only storage. Don't move this without checking with legal.
var bucket = new Aws.S3.Bucket("data", new Aws.S3.BucketArgs
{
    /* ... */
});
```
