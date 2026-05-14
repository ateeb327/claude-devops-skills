# Pulumi C# + Kubernetes / Helm

Practices that apply when Pulumi orchestrates Helm releases (`Pulumi.Kubernetes.Helm.V3.Release`) or Kubernetes resources directly. Skip this file if the stack is not K8s-related.

---

## C1. Don't Ship Helm Values Meant for a Different Deployment Model

**Why**: Many Helm charts include values that only make sense for a specific deployment model. Pulumi installs charts via the Helm SDK natively (equivalent to `helm install`), which has different semantics than GitOps tools like ArgoCD or Flux that use `helm template` + server-side apply. Shipping flags meant for the wrong model is at best a no-op, at worst a confusing source of "why isn't my flag taking effect" debugging.

**Detection signals**:
- Chart-specific flags that only apply to a different deployment model for example, some charts ship a `validationFailurePolicy` value intended for ArgoCD/Flux server-side apply, not `helm install`. Verify against the chart's README and `values.yaml` rather than assuming.
- KMS-based secret config in a stack that uses external secret tooling (SOPS, Sealed Secrets)
- Cluster Autoscaler config in a cluster running Karpenter
- VPC CNI config when running a third-party CNI (Cilium, Calico)
- Multiple admission webhook configs that ordering-conflict with each other

**Rule of thumb**: For every Helm value, ask:
1. Does this flag apply to `helm install` (Pulumi's mode), or only to a different deployment method?
2. Does this flag apply to my cluster's actual setup (autoscaler choice, CNI, secret tooling, ingress controller)?
3. Will the chart silently ignore this if the answer to (1) or (2) is "no"?

If you can't answer all three, don't add the flag. Read the chart's `values.yaml` comments and the chart README for each value before including it.

**Reference**: https://www.pulumi.com/registry/packages/kubernetes/api-docs/helm/v3/release/

---

## C2. DaemonSet Toleration Coverage

**Why**: A DaemonSet that doesn't tolerate every taint across every node group leaves nodes uncovered. For workloads like log collectors, monitoring agents, or security/compliance daemons, an uncovered node means you have no visibility on it and you may not know until something breaks on that node and the investigation reveals a 6-month gap in logs.

**Detection signals**:
- DaemonSet tolerations list shorter than the unique-taints list across all node groups
- New node group added without revisiting DaemonSet tolerations
- DaemonSet pods Pending on specific nodes (`kubectl get pods -o wide` shows nodes with no matching DaemonSet pod)

**Mandatory workflow before writing DaemonSet tolerations**:
1. List ALL node groups in the cluster: managed node groups (EKS), self-managed nodes, Karpenter NodePools, virtual nodes (Fargate)
2. Collect ALL taints from each: managed-node-group `Taints` blocks, launch-template taints, Karpenter NodePool `spec.template.spec.taints` arrays
3. Add a `toleration` entry for every unique `(key, value, effect)` tuple
4. If a node group has no taints, no toleration is needed for it

**Generic example**:
```csharp
// Cluster has three node groups with these taints:
//   system-nodes:    workload=system:NoSchedule
//   gpu-nodes:       nvidia.com/gpu=true:NoSchedule, dedicated=gpu:NoSchedule
//   spot-nodes:      lifecycle=spot:NoSchedule

// A log collector DaemonSet must tolerate ALL of them
var values = new Dictionary<string, object>
{
    ["tolerations"] = new[]
    {
        new Dictionary<string, string>
        {
            ["key"] = "workload", ["value"] = "system",
            ["operator"] = "Equal", ["effect"] = "NoSchedule",
        },
        new Dictionary<string, string>
        {
            ["key"] = "nvidia.com/gpu", ["value"] = "true",
            ["operator"] = "Equal", ["effect"] = "NoSchedule",
        },
        new Dictionary<string, string>
        {
            ["key"] = "dedicated", ["value"] = "gpu",
            ["operator"] = "Equal", ["effect"] = "NoSchedule",
        },
        new Dictionary<string, string>
        {
            ["key"] = "lifecycle", ["value"] = "spot",
            ["operator"] = "Equal", ["effect"] = "NoSchedule",
        },
    },
};
```

**Non-DaemonSet workloads** (Deployments, StatefulSets) only need tolerations for the specific node group they target via `nodeSelector` or `affinity`. The "tolerate everything" rule applies *only* to DaemonSets.

**Operational note**: When a new node group is added to the cluster, every DaemonSet in the cluster needs review. Putting this on the new-node-group runbook prevents silent coverage gaps.

---

## C3. `InputMap<object>` Is Mutable: Extract Helpers, Don't Const

**Why**: `InputMap<object>` and `InputList<object>` are mutable reference types. Storing one as a `static readonly` field and reusing it across multiple resources means every resource shares the same instance and any later mutation to that instance affects every consumer in ways that are hard to debug. `const` doesn't compile for these types at all. The safest pattern is to never share mutable input collections, return a fresh instance from a helper method instead.

**Detection signals**:
- `private static readonly InputMap<object> SharedTolerations = ...`
- Attempted `const` on `InputMap` types (compilation error)
- Multiple Helm releases referencing the same shared values map
- Subtle bugs where a values mutation in one release affects another

**Wrong**:
```csharp
public class MyComponent : ComponentResource
{
    // WRONG: shared mutable instance across all consumers
    private static readonly InputMap<object> SharedTolerations = new()
    {
        ["key"] = "workload",
        ["value"] = "system",
        ["operator"] = "Equal",
        ["effect"] = "NoSchedule",
    };

    public MyComponent(string name, ComponentResourceOptions? opts = null)
        : base("myorg:components:MyComponent", name, opts)
    {
        var release1 = new Release("rel1", new ReleaseArgs
        {
            Values = new Dictionary<string, object> { ["tolerations"] = SharedTolerations },
        }, new CustomResourceOptions { Parent = this });

        var release2 = new Release("rel2", new ReleaseArgs
        {
            Values = new Dictionary<string, object> { ["tolerations"] = SharedTolerations },
        }, new CustomResourceOptions { Parent = this });
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
        var release1 = new Release("rel1", new ReleaseArgs
        {
            Values = new Dictionary<string, object> { ["tolerations"] = CreateSharedTolerations() },
        }, new CustomResourceOptions { Parent = this });

        var release2 = new Release("rel2", new ReleaseArgs
        {
            Values = new Dictionary<string, object> { ["tolerations"] = CreateSharedTolerations() },
        }, new CustomResourceOptions { Parent = this });

        this.RegisterOutputs();
    }

    // RIGHT: helper returns a fresh instance per call — no shared state
    private static InputMap<object>[] CreateSharedTolerations() => new InputMap<object>[]
    {
        new()
        {
            ["key"] = "workload", ["value"] = "system",
            ["operator"] = "Equal", ["effect"] = "NoSchedule",
        },
    };
}
```

**Same pattern applies to**:
- Resource block helpers: `CreateResourceBlock(cpu, memory, memLimit)` instead of duplicating the dict literal in 8 places
- Common label/annotation maps: `CreateStandardLabels(componentName)`
- Probe configs: `CreateLivenessProbe(path, port)`

This pattern also reduces SonarQube S3776 (cognitive complexity) warnings when Helm values trees get deeply nested. break each subtree into its own helper.
