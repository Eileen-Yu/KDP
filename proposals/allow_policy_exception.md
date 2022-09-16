# Policy Exceptions

- **Authors**: Eileen Yu (@Eileen-Yu), Jim Bugwadia (jim@nirmata.com)
- **Created**: Sep 16th, 2022
- **Updated**: Sep 21st, 2022
- **Abstract**: Allow managing policy exceptions (exclude) independently of policies

## Contents

- [Contents](#contents)
- [Introduction](#introduction)
- [The Problem](#the-problem)
- [Proposed Solution](#proposed-solution)
  - [High-Level Flow](#high-level-flow)
  - [Ideal Usage](#ideal-usage)
  - [Possible User Stories](#possible-user-stories)
- [Implementation](#implementation)
- [Alternative Solution](#alternative-solution)
- [Next Steps](#next-steps)

## Introduction

Kyverno policy rules contain an `exclude` block which allows excluding admission requests by resource (name, API group, versions, and kinds), resource labels, namespaces, namespace labels, users, use groups, service accounts, user role and groups. This construct allows flexibility in selecting when a policy should be applied to a admission review request.

This proposal introduces the concept of a `PolicyException` which is similar to the `exclude` declaration but is decoupled from the policy lifecycle.

## The Problem

Policies are often managed via GitOps, and a global policy set may apply across several clusters. In some cases a policy override may be required in specific clusters. Hence, managing exceptions in the policy declaration itself can become painful.

In addition, when a policy exception declaration is updated, there is no clear record of why that exception was added.

These pain points are well ariculated in the following GitHub issue and discussion:

- [Overwriting Existing policies](https://github.com/kyverno/kyverno/discussions/4310)
- [Feature: New CRDs for PolicyViolationExcepotions](https://github.com/kyverno/kyverno/issues/2627)

## Proposed Solution

The proposed solution is to introduce two new Custom Resource Definitions (CRDs) that enable managing policy exceptions:

- **PolicyExceptionRequest**: a namespaced request to exclude a resource from a policy rule
- **PolicyExceptionApprovals**: a cluster-wide resource that permits one or more exceptions

### High-Level Flow

1. A user that is impacted by a policy violaation can create a `PolicyExceptionRequest` for their workload:

`PolicyExceptionRequest` sample:

```yaml
---
spec:
  exceptions:
    - policyName: "..policyA"
      ruleNames:
        - ".."
        - ".."
    - policyName: "..policyB"
      ruleNames:
        - ".."
        - ".."
  reason: ..
  exclude: ..
  duration(optional): ..
status:
  state: PendingApproval / Approved / Rejected
  feedback:
    result: deny/accept
    description: ..
```

`spec` is where the user fill in for request.

- exceptions: an array of exception elements
  - policyName: target policy name (string)
  - ruleNames: an array of target rule names (string)
- reason: brief explanation for the exception
- exclude: specify the resource that need to be excluded from the target policy, it can be [MatchResources](https://github.com/kyverno/kyverno/blob/v1.8.0-rc2/api/kyverno/v1/match_resources_types.go#L10)
- duration(optional): how long the exception would be valid
  Once the CR is created, requester can only have the access to modify spec. We can use RBAC to restrict such accessibility.

After the `PolicyExceptionRequest` CR is applied to the cluster, the admin would be able to review the content and modify `spec` if necessary.
`status` is the field which can only be filled by the admin.

- feedback:
  - result: whether to accept or deny the request
  - description: reason to change / deny the request
- state: automatically updated by `exceptionController` (not necessary to be manually updated)

2. The admin can review the `PolicyExceptionRequest` and can either `approve` or `reject` it. If approved, `exceptionController` would create the `PolicyExceptionApproval` which serves as a log of approval decisions.

`PolicyExceptionApproval` sample:

```yaml
---
spec:
  exceptions:
    - policyName: "..policyA"
      ruleNames:
        - ".."
        - ".."
    - policyName: "..policyB"
      ruleNames:
        - ".."
        - ".."
  exclude: ..
  duration: ..
```

- exceptions: an array of exception elements
  - policyName: target policy name (string)
  - ruleNames: an array of target rule names (string)
- exclude: specify the resource that need to be excluded from the target policy, it can be [MatchResources](https://github.com/kyverno/kyverno/blob/v1.8.0-rc2/api/kyverno/v1/match_resources_types.go#L10)
- duration: how long the exception would be valid

If the exception is rejected, then nothing would go on.

The CRs would get recycled once the resources are reconciled OR it is outdated.

### Ideal Usage

1. User creates a CR `PolicyExceptionRequest` to request for exception.
2. Admin reviews the request, approve or reject.
3. All approved exceptions would be cached by a newly introduced `exceptionController`, and the controller would automatically update CR `PolicyExceptionApproval` for each exception.
4. Kyverno loads the exception from the `PolicyExceptionApproval` CR.
5. User creates the corresponding resource. Kyverno matches the resource with corresponding policies and exceptions. If all broken rules can be matched with exceptions, let the request pass.

### Possible User Stories

**As a user, I want to be excepted from label tag rule for test.**

**a. What policy blocks what resources/user?**

The policy disables the user to use an image tag called `latest`.

```yaml
apiVersion: kyverno.io/v1
kind: ClusterPolicy
metadata:
  name: disallow-latest-tag
  annotations: ...
spec:
  validationFailureAction: enforce
  background: true
  rules:
    - name: validate-image-tag
      match:
        resources:
          kinds:
            - Deployment
      validate:
        message: "Using a mutable image tag e.g. 'latest' is not allowed."
        pattern:
          spec:
            template:
              spec:
                containers:
                  - image: "!*:latest"
```

**b. What scenario the requester want to exclude from the policy?**

The requester may not want to create a specific image tag which just used for test. The default one would be 'latest'. Then this would break the policy.

**c. Why it is not necessary to modify the policy?**

The original rule is still valid for the production. We just need an exception for the dev test.

**d. How does the requester write the new CR?**

The requester needs to specify the target policy / rule, and to what kind of resource the exception would take effect.

```yaml
apiVersion: kyverno.io/v1
kind: PolicyExceptionRequest
metadata:
  name: allow-tag-for-test
spec:
  exceptions:
    - policyName: disallow-latest-tag
      ruleNames:
        - validate-image-tag
  reason: allow-latest-tag-for-test
  exclude:
    all:
      - name: test
        kind:
          - Deployment
```

The `state` would be automatically changed to `pendingApproval`.

**e. How does the administrator review the CR?**

Once the CR is created, the admin can review the CR and modify the spec if necessary.

(1) The admin reject the request by filling in the `result` and `description`.

```yaml
spec:
...
status:
  state: Rejected
  feedback:
    result: deny
    description: ..
```

(2) The admin approve the request by filling the `result` and `description`.

```yaml
spec:
...
status:
  state: Approved
  feedback:
    result: accept
    description: ..
```

**f. What happened if the CR get rejected?**

The `state` in the CR would be updated to `Rejected` until the request get outdated and deleted.

**g. What happened next if the CR get approved?**

The `state` in the CR would be updated to `Approved` until the request get outdated and deleted.

A new CR `PolicyExceptionApproval` would be generated by `exceptionController`. This would be read by Kyverno for rule reconciliation.

```yaml
apiVersion: kyverno.io/v1
kind: PolicyExceptionApproval
metadata:
  name: allow-tag-for-test
spec:
  exceptions:
    - policyName: disallow-latest-tag
      ruleNames:
        - validate-image-tag
  exclude:
    all:
      - name: test
        kinds:
          - Deployment
        namespaces:
          - dev
  duration: 3
```

**h. What does the requester need to do after?**

Once the `PolicyExceptionApproval` CR is applied to the cluster, the exceptions would automatically take effect. The requester can now create the resource.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test
  labels:
    app.kubernetes.io/name: test
spec:
  containers:
    - name: test
      image: test:latest
```

**i. What if the user try to break another rule that is not included by the exception request?**

If the breaking rule cannot be matched with the exceptions, the requested resources should be blocked still.

## Implementation

We will introduce a new operator: `exceptionController` and two new CRDs: `PolicyExceptionRequest` and `PolicyExceptionApproval`.

We can use `Kubebuilder` to scaffold the new operator. The CRD `PolicyExceptionRequest` is also defined in this project scaffold project.

The `exceptionController` would handle the incoming `PolicyExceptionRequest` . It would update the `PolicyExceptionRequest` status based on the requester & admin action. If the admin approves the `PolicyExceptionRequest`, the controller would then generate the `PolicyExceptionApproval` CR.

The new CRD `PolicyExceptionApproval` can be defined and managed by the current Kyverno operator.

We may introduce a new `Informer` to get the latest `PolicyExceptionApproval` in the cluster.

When the user apply the target resource, the Kyverno admission controller would first go through the policy check. If it's blocked, compare the violation report with the approved exception. If all the violations are matched with exceptions, let the request pass.

![flow](https://user-images.githubusercontent.com/48944635/191593154-262f0676-cc3a-41d6-9839-8918e213e9bf.png)



## Alternative Solution

An alternative is just to modify the current Kyverno operator, which means add the exception logic.

## Next Steps

This proposal just provides a solution for basic exception. There are many other things need to be taken into consideration.

1. More user stories / functions
2. Recycling Mechanism
3. Documentation
