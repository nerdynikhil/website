---
reviewers:
- liggitt
- jpbetz
- cici37
title: Validating Admission Policy
content_type: concept
---

<!-- overview -->

{{< feature-state state="alpha" for_k8s_version="v1.26" >}}

This page provides an overview of Validating Admission Policy.


<!-- body -->

## What is Validating Admission Policy?

Validating admission policies offer a declarative, in-process alternative to validating admission webhooks.

Validating admission policies use the Common Expression Language (CEL) to declare the validation
rules of a policy. 
Validation admission policies are highly configurable, enabling policy authors to define policies
that can be parameterized and scoped to resources as needed by cluster administrators.

## What Resources Make a Policy

A policy is generally made up of three resources:

- The `ValidatingAdmissionPolicy` describes the abstract logic of a policy
  (think: "this policy makes sure a particular label is set to a particular value").

- A `ValidatingAdmissionPolicyBinding` links the above resources together and provides scoping.
  If you only want to require an `owner` label to be set for `Pods`, the binding is where you would
  specify this restriction.

- A parameter resource provides information to a ValidatingAdmissionPolicy to make it a concrete
  statement (think "the `owner` label must be set to something that ends in `.company.com`").
  A native type such as ConfigMap or a CRD defines the schema of a parameter resource.
  `ValidatingAdmissionPolicy` objects specify what Kind they are expecting for their parameter resource.

At least a `ValidatingAdmissionPolicy` and a corresponding  `ValidatingAdmissionPolicyBinding`
must be defined for a policy to have an effect.

If a `ValidatingAdmissionPolicy` does not need to be configured via parameters, simply leave
`spec.paramKind` in  `ValidatingAdmissionPolicy` unset.

## {{% heading "prerequisites" %}}

- Ensure the `ValidatingAdmissionPolicy` [feature gate](/docs/reference/command-line-tools-reference/feature-gates/) is enabled.
- Ensure that the `admissionregistration.k8s.io/v1alpha1` API is enabled.

## Getting Started with Validating Admission Policy

Validating Admission Policy is part of the cluster control-plane. You should write and deploy them
with great caution. The following describes how to quickly experiment with Validating Admission Policy.

### Creating a ValidatingAdmissionPolicy

The following is an example of a ValidatingAdmissionPolicy.

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "demo-policy.example.com"
spec:
  failurePolicy: Fail
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= 5"
```

`spec.validations` contains CEL expressions which use the [Common Expression Language (CEL)](https://github.com/google/cel-spec)
to validate the request. If an expression evaluates to false, the validation check is enforced
according to the `spec.failurePolicy` field.

To configure a validating admission policy for use in a cluster, a binding is required.
The following is an example of a ValidatingAdmissionPolicyBinding.:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "demo-binding-test.example.com"
spec:
  policyName: "demo-policy.example.com"
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: test
```

When trying to create a deployment with replicas set not satisfying the validation expression, an
error will return containing message:

```none
ValidatingAdmissionPolicy 'demo-policy.example.com' with binding 'demo-binding-test.example.com' denied request: failed expression: object.spec.replicas <= 5
```

The above provides a simple example of using ValidatingAdmissionPolicy without a parameter configured.

#### Parameter resources

Parameter resources allow a policy configuration to be separate from its definition. 
A policy can define paramKind, which outlines GVK of the parameter resource, 
and then a policy binding ties a policy by name (via policyName) to a particular parameter resource via paramRef.

If parameter configuration is needed, the following is an example of a ValidatingAdmissionPolicy
with parameter configuration.

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
metadata:
  name: "replicalimit-policy.example.com"
spec:
  failurePolicy: Fail
  paramKind:
    apiVersion: rules.example.com/v1
    kind: ReplicaLimit
  matchConstraints:
    resourceRules:
    - apiGroups:   ["apps"]
      apiVersions: ["v1"]
      operations:  ["CREATE", "UPDATE"]
      resources:   ["deployments"]
  validations:
    - expression: "object.spec.replicas <= params.maxReplicas"
      reason: Invalid
```

The `spec.paramKind` field of the ValidatingAdmissionPolicy specifies the kind of resources used
to parameterize this policy. For this example, it is configured by ReplicaLimit custom resources. 
Note in this example how the CEL expression references the parameters via the CEL params variable,
e.g. `params.maxReplicas`. `spec.matchConstraints` specifies what resources this policy is
designed to validate. Note that the native types such like `ConfigMap` could also be used as
parameter reference.

The `spec.validations` fields contain CEL expressions. If an expression evaluates to false, the
validation check is enforced according to the `spec.failurePolicy` field.

The validating admission policy author is responsible for providing the ReplicaLimit parameter CRD.

To configure an validating admission policy for use in a cluster, a binding and parameter resource
are created. The following is an example of a ValidatingAdmissionPolicyBinding.

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "replicalimit-binding-test.example.com"
spec:
  policyName: "replicalimit-policy.example.com"
  paramRef:
    name: "replica-limit-test.example.com"
  matchResources:
    namespaceSelector:
      matchLabels:
        environment: test
```

The parameter resource could be as following:

```yaml
apiVersion: rules.example.com/v1
kind: ReplicaLimit
metadata:
  name: "replica-limit-test.example.com"
maxReplicas: 3
```

This policy parameter resource limits deployments to a max of 3 replicas in all namespaces in the
test environment. An admission policy may have multiple bindings. To bind all other environments
environment to have a maxReplicas limit of 100, create another ValidatingAdmissionPolicyBinding:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "replicalimit-binding-nontest"
spec:
  policyName: "replicalimit-policy.example.com"
  paramRef:
    name: "replica-limit-clusterwide.example.com"
  matchResources:
    namespaceSelector:
      matchExpressions:
      - key: environment
        operator: NotIn
        values:
        - test
```

And have a parameter resource like:

```yaml
apiVersion: rules.example.com/v1
kind: ReplicaLimit
metadata:
  name: "replica-limit-clusterwide.example.com"
maxReplicas: 100
```

Bindings can have overlapping match criteria. The policy is evaluated for each matching binding.
In the above example, the "nontest" policy binding could instead have been defined as a global policy:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicyBinding
metadata:
  name: "replicalimit-binding-global"
spec:
  policyName: "replicalimit-policy.example.com"
  params: "replica-limit-clusterwide.example.com"
  matchResources:
    namespaceSelector:
      matchExpressions:
      - key: environment
        operator: Exists
```

The params object representing a parameter resource will not be set if a parameter resource has
not been bound, so for policies requiring a parameter resource, it can be useful to add a check to
ensure one has been bound.

For the use cases require parameter configuration, we recommend to add a param check in
`spec.validations[0].expression`:

```
- expression: "params != null"
  message: "params missing but required to bind to this policy"
```

It can be convenient to be able to have optional parameters as part of a parameter resource, and
only validate them if present. CEL provides `has()`, which checks if the key passed to it exists.
CEL also implements Boolean short-circuiting. If the first half of a logical OR evaluates to true,
it won’t evaluate the other half (since the result of the entire OR will be true regardless). 

Combining the two, we can provide a way to validate optional parameters:

`!has(params.optionalNumber) || (params.optionalNumber >= 5 && params.optionalNumber <= 10)`

Here, we first check that the optional parameter is present with `!has(params.optionalNumber)`. 

- If `optionalNumber` hasn’t been defined, then the expression short-circuits since
  `!has(params.optionalNumber)` will evaluate to true. 
- If `optionalNumber` has been defined, then the latter half of the CEL expression will be
  evaluated, and optionalNumber will be checked to ensure that it contains a value between 5 and
  10 inclusive.

#### Authorization Check

We introduced the authorization check for parameter resources.
User is expected to have `read` access to the resources referenced by `paramKind` in
`ValidatingAdmissionPolicy` and `paramRef` in `ValidatingAdmissionPolicyBinding`.

Note that if a resource in `paramKind` fails resolving via the restmapper, `read` access to all
resources of groups is required.

### Failure Policy

`failurePolicy` defines how mis-configurations and CEL expressions evaluating to error from the
admission policy are handled. Allowed values are `Ignore` or `Fail`.

- `Ignore` means that an error calling the ValidatingAdmissionPolicy is ignored and the API
  request is allowed to continue.
- `Fail` means that an error calling the ValidatingAdmissionPolicy causes the admission to fail
  and the API request to be rejected.

Note that the `failurePolicy` is defined inside `ValidatingAdmissionPolicy`:

```yaml
apiVersion: admissionregistration.k8s.io/v1alpha1
kind: ValidatingAdmissionPolicy
spec:
...
failurePolicy: Ignore # The default is "Fail"
validations:
- expression: "object.spec.xyz == params.x"  
```

### Validation Expression

`spec.validations[i].expression` represents the expression which will be evaluated by CEL.
To learn more, see the [CEL language specification](https://github.com/google/cel-spec)
CEL expressions have access to the contents of the Admission request/response, organized into CEL
variables as well as some other useful variables:

- 'object' - The object from the incoming request. The value is null for DELETE requests.
- 'oldObject' - The existing object. The value is null for CREATE requests.
- 'request' - Attributes of the [admission request](/docs/reference/config-api/apiserver-admission.v1/#admission-k8s-io-v1-AdmissionRequest).
- 'params' - Parameter resource referred to by the policy binding being evaluated. The value is
  null if `ParamKind` is unset.
	
The `apiVersion`, `kind`, `metadata.name` and `metadata.generateName` are always accessible from
the root of the object. No other metadata properties are accessible.
	
Only property names of the form `[a-zA-Z_.-/][a-zA-Z0-9_.-/]*` are accessible.
Accessible property names are escaped according to the following rules when accessed in the
expression:

| escape sequence         | property name equivalent  |
| ----------------------- | -----------------------|
| `__underscores__`       | `__`                  |
| `__dot__`               | `.`                   |
|`__dash__`               | `-`                   |
| `__slash__`             | `/`                   |
| `__{keyword}__`         | [CEL RESERVED keyword](https://github.com/google/cel-spec/blob/v0.6.0/doc/langdef.md#syntax)       |

{{< note >}}
A **CEL reserved** keyword only needs to be escaped if the token is an exact match
for the reserved keyword.
For example, `int` in the word “sprint” would not be escaped.
{{< /note >}}

Examples on escaping:

|property name    | rule with escaped property name     |
| ----------------| -----------------------             |
| namespace       | `self.__namespace__ > 0`            |
| x-prop          | `self.x__dash__prop > 0`            |
| redact__d       | `self.redact__underscores__d > 0`   |
| string          | `self.startsWith('kube')`           |
	
Equality on arrays with list type of 'set' or 'map' ignores element order, i.e. [1, 2] == [2, 1].
Concatenation on arrays with x-kubernetes-list-type use the semantics of the list type:
    - 'set': `X + Y` performs a union where the array positions of all elements in `X` are preserved and
      non-intersecting elements in `Y` are appended, retaining their partial order.
    - 'map': `X + Y` performs a merge where the array positions of all keys in `X` are preserved but the values
      are overwritten by values in `Y` when the key sets of `X` and `Y` intersect. Elements in `Y` with
      non-intersecting keys are appended, retaining their partial order.

#### Validation expression examples

| Expression                                                                                   | Purpose                                                                           |
|----------------------------------------------------------------------------------------------| ------------                                                                      |
| `object.minReplicas <= object.replicas && object.replicas <= object.maxReplicas`             | Validate that the three fields defining replicas are ordered appropriately        |
| `'Available' in object.stateCounts`                                                          | Validate that an entry with the 'Available' key exists in a map                   |
| `(size(object.list1) == 0) != (size(object.list2) == 0)`                                     | Validate that one of two lists is non-empty, but not both                         |
| <code>!('MY_KEY' in object.map1) &#124;&#124; object['MY_KEY'].matches('^[a-zA-Z]*$')</code> | Validate the value of a map for a specific key, if it is in the map               |
| `object.envars.filter(e, e.name == 'MY_ENV').all(e, e.value.matches('^[a-zA-Z]*$')`          | Validate the 'value' field of a listMap entry where key field 'name' is 'MY_ENV'  |
| `has(object.expired) && object.created + object.ttl < object.expired`                        | Validate that 'expired' date is after a 'create' date plus a 'ttl' duration       |
| `object.health.startsWith('ok')`                                                             | Validate a 'health' string field has the prefix 'ok'                              |
| `object.widgets.exists(w, w.key == 'x' && w.foo < 10)`                                       | Validate that the 'foo' property of a listMap item with a key 'x' is less than 10 |
| `type(object) == string ? object == '100%' : object == 1000`                                 | Validate an int-or-string field for both the int and string cases             |
| `object.metadata.name.startsWith(object.prefix)`                                             | Validate that an object's name has the prefix of another field value              |
| `object.set1.all(e, !(e in object.set2))`                                                    | Validate that two listSets are disjoint                                           |
| `size(object.names) == size(object.details) && object.names.all(n, n in object.details)`     | Validate the 'details' map is keyed by the items in the 'names' listSet           |
| `size(object.clusters.filter(c, c.name == object.primary)) == 1`                             | Validate that the 'primary' property has one and only one occurrence in the 'clusters' listMap           |

Read [Supported evaluation on CEL](https://github.com/google/cel-spec/blob/v0.6.0/doc/langdef.md#evaluation)
for more information about CEL rules.

`spec.validation[i].reason` represents a machine-readable description of why this validation failed.
If this is the first validation in the list to fail, this reason, as well as the corresponding
HTTP response code, are used in the HTTP response to the client.
The currently supported reasons are: `Unauthorized`, `Forbidden`, `Invalid`, `RequestEntityTooLarge`.
If not set, `StatusReasonInvalid` is used in the response to the client.

