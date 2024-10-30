# In-place Update of Pod Resources

## Table of Contents

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [API Changes](#api-changes)
    - [Allocated Resources](#allocated-resources)
    - [Subresource](#subresource)
    - [Validation](#validation)
    - [Container Resize Policy](#container-resize-policy)
    - [Resize Status](#resize-status)
    - [CRI Changes](#cri-changes)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Kubelet and API Server Interaction](#kubelet-and-api-server-interaction)
    - [Kubelet Restart Tolerance](#kubelet-restart-tolerance)
  - [Scheduler and API Server Interaction](#scheduler-and-api-server-interaction)
  - [Flow Control](#flow-control)
    - [Container resource limit update ordering](#container-resource-limit-update-ordering)
    - [Container resource limit update failure handling](#container-resource-limit-update-failure-handling)
    - [CRI Changes Flow](#cri-changes-flow)
    - [Notes](#notes)
  - [Lifecycle Nuances](#lifecycle-nuances)
  - [Atomic Resizes](#atomic-resizes)
  - [Sidecars](#sidecars)
  - [QOS Class](#qos-class)
  - [Resource Quota](#resource-quota)
  - [Affected Components](#affected-components)
  - [Instrumentation](#instrumentation)
  - [Static CPU &amp; Memory Policy](#static-cpu--memory-policy)
  - [Future Enhancements](#future-enhancements)
    - [Mutable QOS Class &quot;Shape&quot;](#mutable-qos-class-shape)
    - [Design Sketch: Workload resource resize](#design-sketch-workload-resource-resize)
    - [Design Sketch: Explicit QOS Class](#design-sketch-explicit-qos-class)
    - [Design Sktech: Pod-level Resources](#design-sktech-pod-level-resources)
  - [Test Plan](#test-plan)
    - [Prerequisite testing updates](#prerequisite-testing-updates)
    - [Unit Tests](#unit-tests)
    - [Integration tests](#integration-tests)
    - [Pod Resize E2E Tests](#pod-resize-e2e-tests)
    - [CRI E2E Tests](#cri-e2e-tests)
    - [Resource Quota and Limit Ranges](#resource-quota-and-limit-ranges)
    - [Resize Policy Tests](#resize-policy-tests)
    - [Backward Compatibility and Negative Tests](#backward-compatibility-and-negative-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [Stable](#stable)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Allocated Resources](#allocated-resources-1)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This proposal aims at allowing Pod resource requests & limits to be updated
in-place, without a need to restart the Pod or its Containers.

The **core idea** behind the proposal is to make PodSpec mutable with regards to
Resources, denoting **desired** resources. Additionally, PodStatus is extended to
provide information about **actual** resources applied to the Pod and its Containers.

This document builds upon [proposal for live and in-place vertical scaling][]
and [Vertical Resources Scaling in Kubernetes][].

[proposal for live and in-place vertical scaling]:
https://github.com/kubernetes/community/pull/1719
[Vertical Resources Scaling in Kubernetes]:
https://docs.google.com/document/d/18K-bl1EVsmJ04xeRq9o_vfY2GDgek6B6wmLjXw-kos4

This proposal also aims to improve the Container Runtime Interface (CRI) APIs for
managing a Container's CPU and memory resource configurations on the runtime.
It seeks to extend UpdateContainerResources CRI API such that it works for
Windows, and other future runtimes besides Linux. It also seeks to extend
ContainerStatus CRI API to allow Kubelet to discover the current resources
configured on a Container.

## Motivation

Resources allocated to a Pod's Container(s) can require a change for various
reasons:
* load handled by the Pod has increased significantly, and current resources
  are not sufficient,
* load has decreased significantly, and allocated resources are unused,
* resources have simply been set improperly.

Currently, changing resource allocation requires the Pod to be recreated since
the PodSpec's Container Resources is immutable.

While many stateless workloads are designed to withstand such a disruption,
some are more sensitive, especially when using low number of Pod replicas.

Moreover, for stateful or batch workloads, Pod restart is a serious disruption,
resulting in lower availability or higher cost of running.

Allowing Resources to be changed without recreating the Pod or restarting the
Containers addresses this issue directly.

Additionally, In-Place Pod Vertical Scaling feature relies on Container Runtime
Interface (CRI) to update CPU and/or memory requests/limits for a Pod's Container(s).

The current CRI API set has a few drawbacks that need to be addressed:
1. UpdateContainerResources CRI API takes a parameter that describes Container
   resources to update for Linux Containers, and this may not work for Windows
   Containers or other potential non-Linux runtimes in the future.
1. There is no CRI mechanism that lets Kubelet query and discover the CPU and
   memory limits configured on a Container from the Container runtime.
1. The expected behavior from a runtime that handles UpdateContainerResources
   CRI API is not very well defined or documented.

### Goals

* Primary: allow to change container resource requests & limits without
  necessarily restarting the container.
* Secondary: allow actors (users, VPA, StatefulSet, JobController) to decide
  how to proceed if in-place resource resize is not possible.
* Secondary: allow users to specify which Containers can be resized without a
  restart.

Additionally, this proposal has two goals for CRI:
  - Modify UpdateContainerResources to allow it to work for Windows Containers,
    as well as Containers managed by other runtimes besides Linux,
  - Provide CRI API mechanism to query the Container runtime for CPU and memory
    resource configurations that are currently applied to a Container.

An additional goal of this proposal is to better define and document the
expected behavior of a Container runtime when handling resource updates.

### Non-Goals

The explicit non-goal of this KEP is to avoid controlling full lifecycle of a
Pod which failed in-place resource resizing. This should be handled by actors
which initiated the resizing.

Other identified non-goals are:
* allow to change Pod QoS class
* to change resources of non-restartable InitContainers
* eviction of lower priority Pods to facilitate Pod resize
* updating extended resources or any other resource types besides CPU, memory
* support for CPU/memory manager policies besides the default 'None' policy
* resolving race conditions with the scheduler

Definition of expected behavior of a Container runtime when it handles CRI APIs
related to a Container's resources is intended to be a high level guide.  It is
a non-goal of this proposal to define a detailed or specific way to implement
these functions. Implementation specifics are left to the runtime, within the
bounds of expected behavior.

## Proposal

### API Changes

Container resource requests & limits can now be mutated via the `/resize` pod subresource.
PodStatus is extended to show the resources applied to the Pod and its Containers.

* Pod.Spec.Containers[i].Resources becomes purely a declaration, denoting the
  **desired** state of Pod resources
* Pod.Status.ContainerStatuses[i].Resources (new field, type
  v1.ResourceRequirements) shows the **actual** resources held by the Pod and
  its Containers for running containers, and the allocated resources for non-running containers.
* Pod.Status.Resize (new field, type map[string]string) explains what is
  happening for a given resource on a given container.

The actual resources are reported by the container runtime in some cases, and for all other
resources are copied from the latest snapshot of allocated resources. Currently, the resources
reported by the runtime are CPU Limit (translated from quota & period), CPU Request (translated from
shares), and Memory Limit.

Additionally, a new `Pod.Spec.Containers[i].ResizePolicy[]` field (type
`[]v1.ContainerResizePolicy`) governs whether containers need to be restarted on resize. See
[Container Resize Policy](#container-resize-policy) for more details.

#### Allocated Resources

When the Kubelet admits a pod initially or admits a resize, all resource requirements from the spec
are cached and checkpointed locally. When a container is (re)started, these are the requests and
limits used.

The alpha implementation of In-Place Pod Vertical Scaling included `AllocatedResources` in the
container status, but only included requests. This field will remain in alpha, guarded by the
separate `InPlacePodVerticalScalingAllocatedStatus` feature gate, and is a candidate for future
removal. With the allocated status feature gate enabled, Kubelet will continue to populate the field
with the allocated requests from the checkpoint.

The scheduler uses `max(spec...resources, status...resources)` for fit decisions, but since the
actual resources are only relevant and reported for running containers, the Kubelet sets
`status...resources` equal to the allocated resources for non-running containers.

See [`Alternatives: Allocated Resources`](#allocated-resources-1) for alternative APIs considered.

The allocated resources API should be reevaluated prior to GA.

#### Subresource

Resource changes can only be made via the new `/resize` subresource, which accepts Update and Patch
verbs. The request & response types for this subresource are the full pod object, but only the
following fields are allowed to be modified:

* `.spec.containers[*].resources`
* `.spec.initContainers[*].resources` (only for sidecars)
* `.spec.resizePolicy`

The `.status.resize` field will be reset to `Proposed` in the response, but cannot be modified in the
request.

#### Validation

Resource fields remain immutable via pod update (a change from the alpha behavior), but are mutable
via the new `/resize` subresource. The following API validation rules will be applied for updates via
the `/resize` subresource:

1. Resources & ResizePolicy must be valid under pod create validation.
2. Computed QOS class cannot change. See [QOS Class](#qos-class) for more details.
3. Running pods without the `Pod.Status.ContainerStatuses[i].Resources` field set cannot be resized.
   See [Version Skew Strategy](#version-skew-strategy) for more details.

#### Container Resize Policy

To provide fine-grained user control, PodSpec.Containers is extended with
ResizeRestartPolicy - a list of named subobjects (new object) that supports
'cpu' and 'memory' as names. It supports the following restart policy values:
* NotRequired - default value; resize the Container without restart, if possible.
* RestartContainer - the container requires a restart to apply new resource values.
  (e.g.  Java process needs to change its Xmx flag) By using ResizePolicy, user
  can mark Containers as safe (or unsafe) for in-place resource update. Kubelet
  uses it to determine the required action.

Note: `NotRequired` restart policy for resize does not *guarantee* that a container
won't be restarted. The runtime may choose to stop the container if it is unable to
apply the new resources without restarts.

Setting the flag to separately control CPU & memory is due to an observation
that usually CPU can be added/removed without much problem whereas changes to
available memory are more probable to require restarts.

If more than one resource type with different policies are updated at the same
time, then `RestartContainer` policy takes precedence over `NotRequired` policy.

If a pod's RestartPolicy is `Never`, the ResizePolicy fields must be set to
`NotRequired` to pass validation.  That said, any in-place resize may result
in the container being stopped *and not restarted*, if the system can not
perform the resize in place.

The `ResizePolicy` field is mutable. Only append is strictly required to support in-place resize
(for new resource requirements), and we may reevaluate full mutability if additional edge cases are
discovered in implementation.

#### Resize Status

In addition to the above, a new field `Pod.Status.Resize[]`
will be added.  This field indicates whether kubelet has accepted or rejected a
proposed resize operation for a given resource.  Any time the
`Pod.Spec.Containers[i].Resources.Requests` field differs from the
`Pod.Status.ContainerStatuses[i].Resources` field, this new field explains why.

This field can be set to one of the following values:
* `Proposed` - the proposed resize (in Spec...Resources) has not been accepted or
  rejected yet. Desired resources != Allocated resources.
* `InProgress` - the proposed resize has been accepted and is being actuated. A new resize request
  will reset the status to `Proposed`. Desired resources == Allocated resources != Actual resources.
* `Deferred` - the proposed resize is feasible in theory (it fits on this node)
  but is not possible right now; it will be re-evaluated.
  Desired resources != Allocated resources.
* `Infeasible` - the proposed resize is not feasible and is rejected; it will not
  be re-evaluated. Desired resources != Allocated resources.
* (no value) - there is no proposed resize.
  Desired resources == Allocated resources == Acutal resources.

Any time the apiserver observes a proposed resize (a modification of a
`Spec...Resources` field), it will automatically set this field to `Proposed`.

To make this field future-safe, consumers should assume that any unknown value
means the same as `Deferred`.

#### CRI Changes

As of Kubernetes v1.20, the CRI has included support for in-place resizing of containers via the
`UpdateContainerResources` API, which is implemented by both containerd and CRI-O. Additionally, the
`ContainerStatus` message includes a `ContainerResources` field, which reports the current resource
configuration of the container.

Even though pod-level cgroups are currently managed by the Kubelet, runtimes may rely need to be
notified when the resource configuration changes. For example, this information should be passed
through to NRI plugins. To this end, we will add a new `UpdatePodSandboxResources` API:

```proto
service RuntimeService {
  ...

  // UpdatePodSandboxResources synchronously updates the PodSandboxConfig with
  // the pod-level resource configuration. This method is called _after_ the
  // Kubelet reconfigures the pod-level cgroups.
  // This request is treated as best effort, and failure will not block the
  // Kubelet with proceeding with a resize.
  rpc UpdatePodSandboxResources(UpdatePodSandboxResourcesRequest) returns (UpdatePodSandboxResourcesResponse) {}
}

message UpdatePodSandboxResourcesRequest {
    // ID of the PodSandbox to update.
    string pod_sandbox_id = 1;

    // Optional overhead represents the overheads associated with this sandbox
    LinuxContainerResources overhead = 2;
    // Optional resources represents the sum of container resources for this sandbox
    LinuxContainerResources resources = 3;

    // Unstructured key-value map holding arbitrary additional information for
    // sandbox resources updating. This can be used for specifying experimental
    // resources to update or other options to use when updating the sandbox.
    map<string, string> annotations = 4;
}

message UpdatePodSandboxResourcesResponse {}
```

The Kubelet will call `UpdatePodSandboxResources` _after_ it has reconfigured the pod-level cgroups.
This ordering is consistent with pod creation, where the Kubelet configures the pod-level cgroups
before calling `RunPodSandbox`.

For now, the `UpdatePodSandboxResources` call will be treated as best-effort by the Kubelet. This
means that in the case of an error, Kubelet will log the the error but otherwise ignore it and
proceed with the resize.

Note: Windows resources are not included here since they are not present in the
WindowsPodSandboxConfig.

### Risks and Mitigations

1. Backward compatibility: When Pod.Spec.Containers[i].Resources becomes
   representative of desired state, and Pod's actual resource configurations are
   tracked in Pod.Status.ContainerStatuses[i].Resources, applications
   that query PodSpec and rely on Resources in PodSpec to determine resource
   configurations will see values that may not represent actual configurations. As a
   mitigation, this change needs to be documented and highlighted in the
   release notes, and in top-level Kubernetes documents.
1. Resizing memory lower: Lowering cgroup memory limits may not work as pages
   could be in use, and approaches such as setting limit near current usage may
   be required. This issue needs further investigation.
1. Scheduler race condition: If a resize happens concurrently with the scheduler evaluating the node
   where the pod is resized, it can result in a node being over-scheduled, which will cause the pod
   to be rejected with an `OutOfCPU` or `OutOfMemory` error. Solving this race condition is out of
   scope for this KEP, but a general solution may be considered in the future.

## Design Details

### Kubelet and API Server Interaction

When a new Pod is created, Scheduler is responsible for selecting a suitable
Node that accommodates the Pod.

For a newly created Pod, `(Init)ContainerStatuses` will be nil until the Pod is
scheduled to a node. When Kubelet admits a Pod, it will record the admitted
requests & limits to its internal allocated resources checkpoint.

When a Pod resize is requested, Kubelet attempts to update the resources
allocated to the Pod and its Containers. Kubelet first checks if the new desired
resources can fit the Node allocable resources by computing the sum of resources
allocated for all Pods in the Node, except the Pod being resized. For the Pod
being resized, it adds the new desired resources (i.e
Spec.Containers[i].Resources.Requests) to the sum.

* If new desired resources fit, Kubelet accepts the resize, updates
  the allocated resourcesa, and sets Status.Resize to
  "InProgress".  It then invokes the UpdateContainerResources CRI API to update
  Container resource limits.  Once all Containers are successfully updated, it
  updates Status...Resources to reflect new resource values and unsets
  Status.Resize.
* If new desired resources don't fit, Kubelet will update the Status.Resize
  field to "Infeasible" and does not act on the resize.
* If new desired resources fit but are in-use at the moment, Kubelet will
  update the Status.Resize field to "Deferred".

In addition to the above, kubelet will generate Events on the Pod whenever a
resize is accepted or rejected, and if possible at key steps during the resize
process.  This will allow humans to know that progress is being made.

If multiple Pods need resizing, they are handled sequentially in an order
defined by the Kubelet (e.g. in order of arrivial).

Scheduler may, in parallel, assign a new Pod to the Node because it uses cached
Pods to compute Node allocable values. If this race condition occurs, Kubelet
resolves it by rejecting that new Pod if the Node has no room after Pod resize.

Note: After a Pod is rejected, the scheduler could try to reschedule the
replacement pod on the same node that just rejected it.  This is a general
statement about Kubernetes and is outside the scope of this KEP.

#### Kubelet Restart Tolerance

If Kubelet were to restart amidst handling a Pod resize, then upon restart, all
Pods are re-admitted based on their current allocated resources (restored from
 checkpoint). Pending resizes are handled after all existing Pods have been
added. This ensures that resizes don't affect previously admitted existing Pods.

### Scheduler and API Server Interaction

Scheduler continues to use Pod's Spec.Containers[i].Resources.Requests for
scheduling new Pods, and continues to watch Pod updates, and updates its cache.
To compute the Node resources allocated to Pods, it must consider pending
resizes, as described by Status.Resize.

For containers which have `Status.Resize = "Infeasible"`, it can
simply use `Status.ContainerStatus[i].Resources`.

For containers which have `Status.Resize = "Proposed"` or `"InProgress"`, it must be pessimistic
and assume that the resize will be imminently accepted.  Therefore it must use
the larger of the Pod's `Spec...Resources.Requests` and
`Status...Resources.Requests` values.

### Flow Control

The following steps denote the flow of a series of in-place resize operations
for a Pod with ResizePolicy set to NotRequired for all its Containers.
This is intentionally hitting various edge-cases to demonstrate.

```
T=0: A new pod is created
    - `spec.containers[0].resources.requests[cpu]` = 1
    - all status is unset

T=1: apiserver defaults are applied
    - `spec.containers[0].resources.requests[cpu]` = 1
    - `status.containerStatuses` = unset
    - `status.resize[cpu]` = unset

T=2: kubelet runs the pod and updates the API
    - `spec.containers[0].resources.requests[cpu]` = 1
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1
    - `status.resize[cpu]` = unset
    - `status.containerStatuses[0].resources.requests[cpu]` = 1

T=3: Resize #1: cpu = 1.5 (via PUT or PATCH or /resize)
    - apiserver validates the request (e.g. `limits` are not below
      `requests`, ResourceQuota not exceeded, etc) and accepts the operation
    - apiserver sets `resize[cpu]` to "Proposed"
    - `spec.containers[0].resources.requests[cpu]` = 1.5
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1
    - `status.resize[cpu]` = "Proposed"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1

T=4: Kubelet watching the pod sees resize #1 and accepts it
    - kubelet sends patch {
        `resourceVersion` = `<previous value>` # enable conflict detection
        `status.containerStatuses[0].allocatedResources[cpu]` = 1.5
        `status.resize[cpu]` = "InProgress"'
      }
    - `spec.containers[0].resources.requests[cpu]` = 1.5
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.5
    - `status.resize[cpu]` = "InProgress"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1

T=5: Resize #2: cpu = 2
    - apiserver validates the request and accepts the operation
    - apiserver sets `resize[cpu]` to "Proposed"
    - `spec.containers[0].resources.requests[cpu]` = 2
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.5
    - `status.resize[cpu]` = "Proposed"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1

T=6: Container runtime applied cpu=1.5
    - kubelet sends patch {
        `resourceVersion` = `<previous value>` # enable conflict detection
        `status.containerStatuses[0].resources.requests[cpu]` = 1.5
        `status.resize[cpu]` = unset
      }
    - apiserver fails the operation with a "conflict" error

T=7: kubelet refreshes and sees resize #2 (cpu = 2)
    - kubelet decides this is possible, but not right now
    - kubelet sends patch {
        `resourceVersion` = `<updated value>` # enable conflict detection
        `status.containerStatuses[0].resources.requests[cpu]` = 1.5
        `status.resize[cpu]` = "Deferred"
      }
    - `spec.containers[0].resources.requests[cpu]` = 2
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.5
    - `status.resize[cpu]` = "Deferred"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1.5

T=8: Resize #3: cpu = 1.6
    - apiserver validates the request and accepts the operation
    - apiserver sets `resize[cpu]` to "Proposed"
    - `spec.containers[0].resources.requests[cpu]` = 1.6
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.5
    - `status.resize[cpu]` = "Proposed"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1.5

T=9: Kubelet watching the pod sees resize #3 and accepts it
    - kubelet sends patch {
        `resourceVersion` = `<previous value>` # enable conflict detection
        `status.containerStatuses[0].allocatedResources[cpu]` = 1.6
        `status.resize[cpu]` = "InProgress"'
      }
    - `spec.containers[0].resources.requests[cpu]` = 1.6
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.6
    - `status.resize[cpu]` = "InProgress"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1.5

T=10: Container runtime applied cpu=1.6
    - kubelet sends patch {
        `resourceVersion` = `<previous value>` # enable conflict detection
        `status.containerStatuses[0].resources.requests[cpu]` = 1.6
        `status.resize[cpu]` = unset
      }
    - `spec.containers[0].resources.requests[cpu]` = 1.6
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.6
    - `status.resize[cpu]` = unset
    - `status.containerStatuses[0].resources.requests[cpu]` = 1.6

T=11: Resize #4: cpu = 100
    - apiserver validates the request and accepts the operation
    - apiserver sets `resize[cpu]` to "Proposed"
    - `spec.containers[0].resources.requests[cpu]` = 100
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.6
    - `status.resize[cpu]` = "Proposed"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1.6

T=12: Kubelet watching the pod sees resize #4
    - this node does not have 100 CPUs, so kubelet cannot accept
    - kubelet sends patch {
        `resourceVersion` = `<previous value>` # enable conflict detection
        `status.resize[cpu]` = "Infeasible"'
      }
    - `spec.containers[0].resources.requests[cpu]` = 100
    - `status.containerStatuses[0].allocatedResources[cpu]` = 1.6
    - `status.resize[cpu]` = "Infeasible"
    - `status.containerStatuses[0].resources.requests[cpu]` = 1.6
```

#### Container resource limit update ordering

When in-place resize is requested for multiple Containers in a Pod, Kubelet
updates resource limit for the Pod and its Containers in the following manner:
  1. If resource resizing results in net-increase of a resource type (CPU or
     Memory), Kubelet first updates Pod-level cgroup limit for the resource
     type, and then updates the Container resource limit.
  1. If resource resizing results in net-decrease of a resource type, Kubelet
     first updates the Container resource limit, and then updates Pod-level
     cgroup limit.
  1. If resource update results in no net change of a resource type, only the
     Container resource limits are updated.

In all the above cases, Kubelet applies Container resource limit decreases
before applying limit increases.

#### Container resource limit update failure handling

If multiple Containers in a Pod are being updated, and UpdateContainerResources
CRI API fails for any of the containers, Kubelet will backoff and retry at a
later time. Kubelet does not attempt to update limits for containers that are
lined up for update after the failing container. This ensures that sum of the
container limits does not exceed Pod-level cgroup limit at any point. Once all
the container limits have been successfully updated, Kubelet updates the Pod's
Status.ContainerStatuses[i].Resources to match the desired limit values.

#### CRI Changes Flow

Below diagram is an overview of Kubelet using UpdateContainerResources and
ContainerStatus CRI APIs to set new container resource limits, and update the
Pod Status in response to user changing the desired resources in Pod Spec.

```
   +-----------+                   +-----------+                  +-----------+
   |           |                   |           |                  |           |
   | apiserver |                   |  kubelet  |                  |  runtime  |
   |           |                   |           |                  |           |
   +-----+-----+                   +-----+-----+                  +-----+-----+
         |                               |                              |
         |       watch (pod update)      |                              |
         |------------------------------>|                              |
         |     [Containers.Resources]    |                              |
         |                               |                              |
         |                            (admit)                           |
         |                               |                              |
         |                               |  UpdateContainerResources()  |
         |                               |----------------------------->|
         |                               |                         (set limits)
         |                               |<- - - - - - - - - - - - - - -|
         |                               |                              |
         |                               |      ContainerStatus()       |
         |                               |----------------------------->|
         |                               |                              |
         |                               |     [ContainerResources]     |
         |                               |<- - - - - - - - - - - - - - -|
         |                               |                              |
         |      update (pod status)      |                              |
         |<------------------------------|                              |
         | [ContainerStatuses.Resources] |                              |
         |                               |                              |

```

* Kubelet invokes UpdateContainerResources() CRI API in ContainerManager
  interface to configure new CPU and memory limits for a Container by
  specifying those values in ContainerResources parameter to the API. Kubelet
  sets ContainerResources parameter specific to the target runtime platform
  when calling this CRI API.

* Kubelet calls ContainerStatus() CRI API in ContainerManager interface to get
  the CPU and memory limits applied to a Container. It uses the values returned
  in ContainerStatus.Resources to update ContainerStatuses[i].Resources.Limits
  for that Container in the Pod's Status.

#### Notes

* If CPU Manager policy for a Node is set to 'static', then only integral
  values of CPU resize are allowed. If non-integral CPU resize is requested
  for a Node with 'static' CPU Manager policy, that resize is rejected, and
  an error message is logged to the event stream.
* To avoid races and possible gamification, all components will use Pod's
  Status.ContainerStatuses[i].Resources when computing resources used
  by Pods.
* If additional resize requests arrive when a Pod is being resized, those
  requests are handled after completion of the resize that is in progress. And
  resize is driven towards the latest desired state.
* Lowering memory limits may not always take effect quickly if the application
  is holding on to pages. Kubelet will use a control loop to set the memory
  limits near usage in order to force a reclaim, and update the Pod's
  Status.ContainerStatuses[i].Resources only when limit is at desired value.
* Impact of Pod Overhead: Kubelet adds Pod Overhead to the resize request to
  determine if in-place resize is possible.
* At this time, Vertical Pod Autoscaler should not be used with Horizontal Pod
  Autoscaler on CPU, memory. This enhancement does not change that limitation.

### Lifecycle Nuances

* Terminated containers can be "resized" in that the resize is permitted by the API, and the Kubelet
  will accept the changes. This makes race conditions where the container terminates around the
  resize "fail open", and prevents a resize of a terminated container from blocking the resize of a
  running container (see [Atomic Resizes](#atomic-resizes)).
* Resizing pods in a graceful shutdown state is permitted, and will be actuated best-effort.

### Atomic Resizes

A single resize request can change multiple values, including any or all of:
* Multiple resource types
* Requests & Limits
* Multiple containers

These resource requests & limits can have interdependencies that Kubernetes may not be aware of. For
example, two containers (in the same pod) coordinating work may need to be scaled in tandem. It probably doesn't makes
sense to scale limits independently of requests, and scaling CPU without memory could just waste
resources. To mitigate these issues and simplify the design, the Kubelet will treat the requests &
limits for all containers in the spec as a single atomic request, and won't accept any of the
changes unless all changes can be accepted. If multiple requests mutate the resources spec before
the Kubelet has accepted any of the changes, it will treat them as a single atomic request.

Note: If a second infeasible resize is made before the Kubelet allocates the first resize, there can
be a race condition where the Kubelet may or may not accept the first resize, depending on whether
it admits the first change before seeing the second. This race condition is accepted as working as
intended.

The atomic resize requirement should be reevaluated prior to GA, and in the context of pod-level resources.

### Sidecars

Sidecars, a.k.a. restartable InitContainers can be resized the same as regular containers. There are
no special considerations here. Non-restartable InitContainers cannot be resized.

### QOS Class

A pod's QOS class is immutable. This is enforced during validation, which requires that after a
resize the computed QOS Class matches the previous QOS class.

[Future enhancements: Mutable QOS Class "Shape"](#mutable-qos-class-shape) proposes a potential
change to partially relax this restriction, but is removed from Beta scope.

[Future enhancements: explicit QOS Class](#design-sketch-explicit-qos-class) proposes an alternative
enhancement on that, to make QOS class explicit and improve semantics around [workload resource
resize](#design-sketch-workload-resource-resize).

### Resource Quota

With InPlacePodVerticalScaling enabled, resource quota needs to consider pending resizes. Similarly
to how this is handled by scheduling, resource quota will use the maximum of
`.spec.container[*].resources.requests` and `.status.containerStatuses[*].resources` to
determine the effective request values.

To properly handle scale-down, this means that the resource quota controller now needs to evaluate
pod updates where `.status...resources` changed.

### Affected Components

Pod v1 core API:
* extend API
* auto-reset Status.Resize on changes to Resources
* added validation allowing only CPU and memory resource changes,
* set default for ResizePolicy

Admission Controllers: LimitRanger, ResourceQuota need to support Pod Updates:
* for ResourceQuota, podEvaluator.Handler implementation is modified to allow
  Pod updates, and verify that sum of Pod.Spec.Containers[i].Resources for all
  Pods in the Namespace don't exceed quota,
* PodResourceAllocation admission plugin is ordered before ResourceQuota.
* for LimitRanger we check that a resize request does not violate the min and
  max limits specified in LimitRange for the Pod's namespace.

Kubelet:
* set Pod's Status.ContainerStatuses[i].Resources for Containers upon placing
  a new Pod on the Node,
* update Pod's Status.Resize and Status...AllocatedResources upon resize,
* change UpdateContainerResources CRI API to work for both Linux & Windows.

Scheduler:
* compute resource allocations using actual Status...Resources.

Other components:
* check how the change of meaning of resource requests influence other
  Kubernetes components.

### Instrumentation

The following new metric will be added to track total resize requests, counted at the pod level. In
otherwords, a single pod update changing multiple containers and/or resources will count as a single
resize request.

`kubelet_container_resize_requests_total` - Total number of resize requests observed by the Kubelet.

Label: `state` - Count resize request state transitions. This closely tracks the [Resize status](#resize-status) state transitions, omitting `InProgress`. Possible values:
  - `proposed` - Initial request state
  - `infeasible` - Resize request cannot be completed.
  - `deferred` - Resize request cannot initially be completed, but will retry
  - `completed` - Resize operation completed successfully (`spec.Resources == allocated resources == status.Resources`)
  - `canceled` - Pod was terminated before resize was completed, or a new resize request was started.

In steady state, `proposed` should equal `infeasible + completed + canceled`.

The metric is recorded as a counter instead of a gauge to ensure that usage can be tracked over
time, irrespective of scrape interval.

### Static CPU & Memory Policy

Resizing pods with static CPU & memory policy configured is out-of-scope for the beta release of
in-place resize. If a pod is a guaranteed QOS on a node with a static CPU or memory policy
configured, then the resize will be marked as infeasible.

This will be reconsidered post-beta as a future enhancement.

### Future Enhancements

1. Kubelet (or Scheduler) evicts lower priority Pods from Node to make room for
   resize. Pre-emption by Kubelet may be simpler and offer lower latencies.
1. Allow ResizePolicy to be set on Pod level, acting as default if (some of)
   the Containers do not have it set on their own.
1. Extend ResizePolicy to separately control resource increase and decrease
   (e.g. a Container can be given more memory in-place but decreasing memory
   requires Container restart).
1. Handle resize of guaranteed pods with static CPU or memory policy.
1. Extend controllers (Job, Deployment, etc) to propagate Template resources
   update to running Pods.
1. Allow resizing local ephemeral storage.
1. Handle pod-scoped resources (https://github.com/kubernetes/enhancements/pull/1592)

#### Mutable QOS Class "Shape"

This change was originally proposed for Beta, but moved out of the scope. It may still be considered
for a future enhancement to relax the constraints on resizes.

A pod's QOS class **cannot be changed** once the pod is started, independent of any resizes.

To clarify the discussion of the proposed QOS Class changes, the following terms are defined:

* "QOS Class" - The QOS class that was computed based on the original resource requests & limits
  when the pod was first created.
* "QOS Shape" - The QOS class that _would_ be computed based on the current resource requests &
  limits.

On creation, the QOS Class is equal to the QOS Shape. After a resize, the QOS Shape must be greater
than or equal to the original QOS Class:

* Guaranteed pods: must maintain `requests == limits`, and must be set for both CPU & memory
* Burstable pods: _can_ be resized such that `requests == limits`, but their original QOS
class will stay burstable. Must retain at least one CPU or memory request or limit.
* BestEffort pods: can be freely resized, but stay BestEffort.

Even though the QOS Shape is allowed to change, the original QOS class is used for all
decisions based on QOS class:

* `.status.qosClass` always reports the original QOS class
* Pod cgroup hierarchy is static, using the original QOS class
* Non-guaranteed pods remain ineligible for guaranteed CPUs or NUMA pinning
* Preemption uses the original QOS Class
* OOMScoreAdjust is calculated with the original QOS Class
* Memory pressure eviction is unaffected (doesn't consider QOS Class)

The original QOS Class is persisted to the status. On restart, the Kubelet is allowed to read the
QOS class back from the status.

See [future enhancements: explicit QOS Class](#design-sketch-explicit-qos-class) for a possible
change to make QOS class explicit and improve semantics around
[workload resource resize](#design-sketch-workload-resource-resize).

#### Design Sketch: Workload resource resize

The following [workload resources](https://kubernetes.io/docs/concepts/workloads/) are considered
for in-place resize support:

* Deployment
* ReplicaSet
* StatefulSet
* DaemonSet
* Job
* CronJob

Each of these resources will have a new `ResizePolicy` field added to the spec. In the case of
Deployments or Cronjobs, the child (ReplicaSet/Job) will inherit the policy. The resize policy is
set to one of: `InPlace` or `Recreate` (default). If the policy is set to recreate, the behavior is
unchanged, and generally induces a rolling update.

If the policy is set to in-place, the controller will *attempt* to issue an in-place resize to all
the child pods. If the resize is not a legal in-place resize, such as changing from guaranteed to
burstable, the replicas will be recreated.

Open Questions:
* Will resizes be issued through a new `/resize` subresource? If so, what happens if a resize is
  made that doesn't go through the subresource?
* Does ResizePolicy need to be per-resource type (similar to the resize restart policy on pods)?
* Can you do a rolling-in-place-resize, or are all child pod resizes issued more or less
  simultaneously?

#### Design Sketch: Explicit QOS Class

Workload resource resize presents a problem for QOS handling. For example:

1. ReplicaSet created with a burstable pod shape
2. Initial burstable replicas created
3. Resize to a guaranteed shape
4. Initial replicas are still burstable, but with a guaranteed shape
5. Horizontally scale the RS to add additional replicas
6. New replicas are created with the guaranteed resource shape, and assigned the guaranteed QOS class
7. Resize back to a burstable shape (undoing step 3)

After step 6, there are a mix of burstable & guaranteed replicas. In step 7, the burstable pods can
be resized in-place, but the guaranteed pods will need to be recreated.

To mitigate this, we can introduce an explicit QOSClass field to the pod spec. If set, it must be
less than or equal to the QOS shape. In other words, you can set a guaranteed resource shape but an
explicit QOSClass of burstable, but not the other way around. If set, the status QOSClass is synced
to the explicit QOSClass, and the rest of the behavior is unchanged from the
[QOS Class Proposal](#qos-class).

Going back to the earlier example, if the original ReplicaSet set an explicit Burstable QOSClass,
then the heterogeneity in step 6 is avoided. Alternatively, if there was a real desire to switch to
guaranteed in step 3, then the explicit QOSClass can be changed, triggering a recreation of all
replicas.

#### Design Sktech: Pod-level Resources

Adding resize capabilities to [Pod-level Resources](https://github.com/kubernetes/enhancements/issues/2837)
should largely mirror container-level resize. This includes:
- Add actual resources to `PodStatus.Resources`
- Track allocated pod-level resources
- Factor pod-level resource resize into ResizeStatus logic
- Pod-level resizes are treated as atomic with container level resizes.

Open questions:
- Details around defaulting logic, pending finalization in the pod-level resources KEP
- If the resize policy is `RestartContainer`, are all containers restarted on pod-level resize? Or
  does it depend on whether container-level cgroups are changing?

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*
The goal is to ensure that we don't accept enhancements with inadequate testing.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

#### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

#### Unit Tests

Unit tests will cover the sanity of code changes that implements the feature,
and the policy controls that are introduced as part of this feature.

CRI unit tests are updated to reflect use of ContainerResources object in
UpdateContainerResources and ContainerStatus APIs.

#### Integration tests

Comprehensive E2E tests provide good coverage for alpha. We may replicate and/or move
some of the E2E tests functionality into integration tests before Beta using data from
any issues we uncover that are not covered by planned and implemented tests.

#### Pod Resize E2E Tests

End-to-End tests resize a Pod via PATCH to Pod's Spec.Containers[i].Resources.
The e2e tests use docker as container runtime.
  - Resizing of Requests are verified by querying the values in Pod's
    Status.ContainerStatuses[i].AllocatedResources field.
  - Resizing of Limits are verified by querying the cgroup limits of the Pod's
    containers.

E2E test cases for Guaranteed class Pod with one container:
1. Increase, decrease Requests & Limits for CPU only.
1. Increase, decrease Requests & Limits for memory only.
1. Increase, decrease Requests & Limits for CPU and memory.
1. Increase CPU and decrease memory.
1. Decrease CPU and increase memory.
1. Add memory request & limit for CPU only container.
1. Remove memory request & limit for CPU & memory container.

E2E test cases for Burstable class single container Pod that specifies
both CPU & memory:
1. Increase, decrease Requests - CPU only.
1. Increase, decrease Requests - memory only.
1. Increase, decrease Requests - both CPU & memory.
1. Increase, decrease Limits - CPU only.
1. Increase, decrease Limits - memory only.
1. Increase, decrease Limits - both CPU & memory.
1. Increase, decrease Requests & Limits - CPU only.
1. Increase, decrease Requests & Limits - memory only.
1. Increase, decrease Requests & Limits - both CPU and memory.
1. Increase CPU (Requests+Limits) & decrease memory(Requests+Limits).
1. Decrease CPU (Requests+Limits) & increase memory(Requests+Limits).
1. Increase CPU Requests while decreasing CPU Limits.
1. Decrease CPU Requests while increasing CPU Limits.
1. Increase memory Requests while decreasing memory Limits.
1. Decrease memory Requests while increasing memory Limits.
1. CPU: increase Requests, decrease Limits, Memory: increase Requests, decrease Limits.
1. CPU: decrease Requests, increase Limits, Memory: decrease Requests, increase Limits.
1. Set requests == limits, ensure QOS class remains Burstable

E2E tests for Burstable class single container Pod that specifies CPU only:
1. Increase, decrease CPU - Requests only.
1. Increase, decrease CPU - Limits only.
1. Increase, decrease CPU - both Requests & Limits.

E2E tests for Burstable class single container Pod that specifies memory only:
1. Increase, decrease memory - Requests only.
1. Increase, decrease memory - Limits only.
1. Increase, decrease memory - both Requests & Limits.

E2E tests for BestEffort class single container Pod:
1. Add CPU requests & limits, QOS class remains BestEffort
2. Add Memory requests & limits, QOS class remains BestEffort

E2E tests for Guaranteed class Pod with three containers (c1, c2, c3):
1. Increase CPU & memory for all three containers.
1. Decrease CPU & memory for all three containers.
1. Increase CPU, decrease memory for all three containers.
1. Decrease CPU, increase memory for all three containers.
1. Increase CPU for c1, decrease c2, c3 unchanged - no net CPU change.
1. Increase memory for c1, decrease c2, c3 unchanged - no net memory change.
1. Increase CPU for c1, decrease c2 & c3 - net CPU decrease for Pod.
1. Increase memory for c1, decrease c2 & c3 - net memory decrease for Pod.
1. Increase CPU for c1 & c3, decrease c2 - net CPU increase for Pod.
1. Increase memory for c1 & c3, decrease c2 - net memory increase for Pod.

E2E tests for sidecar containers
1. InitContainer, then sidecar - can increase & decrease CPU & memory of sidecar
2. Sidecar then InitContainer - can increase & decrease CPU & memory of sidecar
3. Resize sidecar along with container

#### CRI E2E Tests

1. E2E test is added to verify UpdateContainerResources API with containerd runtime.
1. E2E test is added to verify ContainerStatus API using containerd runtime.
1. E2E test is added to verify backward compatibility using containerd runtime.

#### Resource Quota and Limit Ranges

Setup a namespace with ResourceQuota and a single, valid Pod.
1. Resize the Pod within resource quota - CPU only.
1. Resize the Pod within resource quota - memory only.
1. Resize the Pod within resource quota - both CPU and memory.
1. Resize the Pod to exceed resource quota - CPU only.
1. Resize the Pod to exceed resource quota - memory only.
1. Resize the Pod to exceed resource quota - both CPU and memory.

Setup a namespace with min and max LimitRange and create a single, valid Pod.
1. Increase, decrease CPU within min/max bounds.
1. Increase CPU to exceed max value.
1. Decrease CPU to go below min value.
1. Increase memory to exceed max value.
1. Decrease memory to go below min value.

#### Resize Policy Tests

Setup a guaranteed class Pod with two containers (c1 & c2).
1. No resize policy specified, defaults to NotRequired. Verify that CPU and
   memory are resized without restarting containers.
1. NotRequired (cpu, memory) policy for c1, RestartContainer (cpu, memory) for c2.
   Verify that c1 is resized without restart, c2 is restarted on resize.
1. NotRequired cpu, RestartContainer memory policy for c1. Resize c1 CPU only,
   verify container is resized without restart.
1. NotRequired cpu, RestartContainer memory policy for c1. Resize c1 memory only,
   verify container is resized with restart.
1. NotRequired cpu, RestartContainer memory policy for c1. Resize c1 CPU & memory,
   verify container is resized with restart.

#### Backward Compatibility and Negative Tests

1. Verify that Node is allowed to update only a Pod's AllocatedResources field.
1. Verify that only Node account is allowed to udate AllocatedResources field.
1. Verify that updating Pod Resources in workload template spec retains current
   behavior:
   - Updating Pod Resources in Job template is not allowed.
   - Updating Pod Resources in Deployment template continues to result in Pod
     being restarted with updated resources.
1. Verify Pod updates by older version of client-go doesn't result in current
   values of AllocatedResources and ResizePolicy fields being dropped.
1. Verify that only CPU and memory resources are mutable by user.

TODO: Identify more cases

### Graduation Criteria

#### Alpha
- In-Place Pod Resouces Update functionality is implemented for running Pods,
- LimitRanger and ResourceQuota handling are added,
- Resize Policies functionality is implemented,
- Unit tests and E2E tests covering basic functionality are added,
- E2E tests covering multiple containers are added.
- UpdateContainerResources API changes are done and tested with containerd
  runtime, backward compatibility is maintained.
- ContainerStatus API changes are done. Tests are ready but not enforced.

#### Beta
- E2E tests covering Resize Policy, LimitRanger, and ResourceQuota are added.
- Negative tests are identified and added.
- A "/resize" subresource is defined and implemented.
- Pod-scoped resources are handled if that KEP is past alpha
- ContainerStatus API change tests are enforced and containerd runtime must comply.
- ContainerStatus API change tests are enforced and Windows runtime should comply.

#### Stable
- VPA integration of feature moved to beta,
- User feedback (ideally from at least two distinct users) is green,
- No major bugs reported for three months.
- Pod-scoped resources are handled if that KEP is past alpha
- `UpdatePodSandboxResources` is implemented by containerd & CRI-O
- Re-evaluate the following decisions:
  - Resize atomicity
  - Exposing allocated resources in the pod status
  - QOS class changes

### Upgrade / Downgrade Strategy
Scheduler and API server should be updated before Kubelets in that order.
Kubelet and the runtime versions should use the same CRI version in lock-step.
Upgrade involves draining all pods from a node, installing a CRI runtime with this
version of the API and update to a matching kubelet and making node schedulable again.
Downgrade involves doing the above in reverse.

### Version Skew Strategy
CRI changes were merged in v1.25 in order to enable runtimes to implement support.
  - containerd added support for this feature in 1.6.9

Previous versions of clients that are unaware of the new ResizePolicy fields would set them
to nil. API server mutates such updates by copying non-nil values from old Pod to the current
Pod.

Prior to v1.31, with InPlacePodVerticalScaling disabled, the kubelet interprets mutation to Pod
Resources as a Container definition change and will restart the container with the new Resources.
This could lead to Node resource over-subscription. In v1.31, the kubelet no longer considers
resource changes a change in the pod definition and doesn't restart the container. In this case, the
change to the new resource value happens if the container is restart for any other reason, making
the change non-deterministic and not reflected in the API. Both of these cases are undesirable, so
the API server should reject a resize request if the Kubelet does not support it
(InPlacePodVerticalScaling enabled).

To achieve this, the apiserver will check if the `.status.containerStatuses[*].resources` field is
non-nil on any running containers. This field is set by the kubelet on running containers if and
only if IPPVS is enabled, and can therefore be used as a proxy to determine if the Kubelet running
the pod has the feature enabled. The apiserver logic to determine if a resource mutation is allowed
then becomes:

```go
if !InPlacePodVerticalScaling {
  return false
}
for _, c := range pod.Status.ContainerStatuses {
  if c.State.Running != nil {
    return c.Resources != nil
  }
}
// No running containers
return true
```

Note that even if the container does not specify any resources requests, the status
Resources is still set to the non-nill empty value `{}`.

If a pod has not yet been scheduled, the resize is allowed, and the new values are used when
scheduling & starting the pod.

If a pod has been scheduled but does not have any running containers, there is no signal indicating
whether the assigned node supports resize, so we default to allowing resize. If the node does not
have resize enabled in this case, then a resized container will be started with the new resource
value. It is possible that the node could end up over-provisioned in this case.

It is also possible for a race condition to occur: resize on a non-running container is allowed, but
the Kubelet simultaneously starts the container. The resulting behavior would depend on the version:
prior to v1.31, the container is restarted with the new values. After v1.31, the container continues
running with the old resource values. Since this race condition only exists during enablement skew,
we choose to accept it as a known-issue.

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/20190731-production-readiness-review-process.md.

The production readiness review questionnaire must be completed for features in
v1.19 or later, but is non-blocking at this time. That is, approval is not
required in order to be in the release.

In some cases, the questions below should also have answers in `kep.yaml`. This
is to enable automation to verify the presence of the review, and to reduce review
burden and latency.

The KEP must have a approver from the
[`prod-readiness-approvers`](http://git.k8s.io/enhancements/OWNERS_ALIASES)
team. Please reach out on the
[#prod-readiness](https://kubernetes.slack.com/archives/CPNHUMN74) channel if
you need any help or guidance.

-->

### Feature Enablement and Rollback

_This section must be completed when targeting alpha to a release._

* **How can this feature be enabled / disabled in a live cluster?**
  - [x] Feature gate (also fill in values in `kep.yaml`)
    - Feature gate name: `InPlacePodVerticalScaling`
      - Components depending on the feature gate: kubelet, kube-apiserver, kube-scheduler
    - Feature gate name: `InPlacePodVerticalScalingAllocatedStatus`
      - Components depending on the feature gate: kubelet, kube-apiserver
      - Requires `InPlacePodVerticalScaling` be enabled

* **Does enabling the feature change any default behavior?**

  - Kubelet sets several pod status fields: `AllocatedResources`, `Resources`

* **Can the feature be disabled once it has been enabled (i.e. can we roll back
  the enablement)?** Yes

  - `InPlacePodVerticalScaling` can be disabled without issue in the control plane.
  - `InPlacePodVerticalScaling` can be disabled on nodes, but if there are any pending resizes
    container resource configurations may be left in an unknown state. This can be avoided by
    draining the node before disabling in-place resize.
  - `InPlacePodVerticalScalingAllocatedStatus` can be disabled and reenabled without consequence.

* **What happens if we reenable the feature if it was previously rolled back?**
  - API will once again permit modification of Resources for 'cpu' and 'memory'.
  - Actual resources applied will be reflected in in Pod's ContainerStatuses.

* **Are there any tests for feature enablement/disablement?**
  Unit tests and E2E tests.
   - Unit tests verify that feature does not introduce any regression.
   - E2E tests run against a local cluster verify that feature works as expected.

### Rollout, Upgrade and Rollback Planning

_This section must be completed when targeting beta graduation to a release._

* **How can a rollout fail? Can it impact already running workloads?**

  - Failure scenarios are already covered by the version skew strategy.

* **What specific metrics should inform a rollback?**

  - Scheduler indicators:
    - `scheduler_pending_pods`
    - `scheduler_pod_scheduling_attempts`
    - `scheduler_pod_scheduling_duration_seconds`
    - `scheduler_unschedulable_pods`
  - Kubelet indicators:
    - `kubelet_pod_worker_duration_seconds`
    - `kubelet_runtime_operations_errors_total{operation_type=update_container}` 


* **Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?**

  Testing plan:

  1. Create test pod
  2. Upgrade API server
  3. Attempt resize of test pod
     - Expected outcome: resize is rejected (see version skew section for details)
  4. Create upgraded node
  5. Create second test pod, scheduled to upgraded node
  6. Attempt resize of second test pod
    - Expected outcome: resize successful
  7. Delete upgraded node
  8. Restart API server with feature disabled
    - Ensure original test pod is still running
  9. Attempt resize of original test pod
    - Expected outcome: request rejected by apiserver
  10. Restart API server with feature enabled
    - Verify original test pod is still running

* **Is the rollout accompanied by any deprecations and/or removals of features, APIs,
fields of API types, flags, etc.?**

  No.

### Monitoring Requirements

_This section must be completed when targeting beta graduation to a release._

* **How can an operator determine if the feature is in use by workloads?**

  Metric: `kubelet_container_resize_requests_total` (see [Instrumentation](#instrumentation))

* **What are the SLIs (Service Level Indicators) an operator can use to determine
the health of the service?**
  - [x] Metrics
    - Metric name: `kubelet_container_resize_requests_total`
      - Components exposing the metric: kubelet
    - Metric name: `runtime_operations_duration_seconds{operation_type=container_update}`
      - Components exposing the metric: kubelet
    - Metric name: `runtime_operations_errors_total{operation_type=container_update}`
      - Components exposing the metric: kubelet

* **What are the reasonable SLOs (Service Level Objectives) for the above SLIs?**

  - Using `kubelet_container_resize_requests_total`, `completed + infeasible + canceled` request count
  should approach `proposed` request count in steady state.
  - Resource update operations should complete quickly (`runtime_operations_duration_seconds{operation_type=container_update} < X` for 99% of requests)
  - Resource update error rate should be low (`runtime_operations_errors_total{operation_type=container_update}/runtime_operations_total{operation_type=container_update}`)

* **Are there any missing metrics that would be useful to have to improve observability
of this feature?**

  - Kubelet admission rejections: https://github.com/kubernetes/kubernetes/issues/125375
  - Resize operate duration (time from the Kubelet seeing the request to actuating the changes): this would require persisting more state about when the resize was first observed.

### Dependencies

_This section must be completed when targeting beta graduation to a release._

* **Does this feature depend on any specific services running in the cluster?**

  Compatible container runtime (see [CRI changes](#cri-changes)).

### Scalability

_For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them._

_For beta, this section is required: reviewers must answer these questions._

_For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field._

* **Will enabling / using this feature result in any new API calls?** Yes
  Describe them, providing:
  - API call type (e.g. PATCH pods)
    - One new PATCH PodStatus API call in response to Pod resize request.
    - No additional overhead unless Pod resize is invoked.
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
    - Kubelet
  focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)

* **Will enabling / using this feature result in introducing new API types?** No
  Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)

* **Will enabling / using this feature result in any new calls to the cloud
provider?** No

* **Will enabling / using this feature result in increasing size or count of
the existing API objects?** Yes
  Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
    - type Container has new field ResizePolicy, a list that adds upto 50 bytes.
    - type PodStatus has a new field, a list that adds upto 32 bytes.
    - type ContainerStatus has new field of type v1.ResourceList that mirrors
      Container.Resources.Requests in size.
    - type ContainerStatus has new field of type v1.ResourceRequirements that
      mirrors Container.Resources in size.

* **Will enabling / using this feature result in increasing time taken by any
operations covered by [existing SLIs/SLOs]?** No
  Think about adding additional work or introducing new steps in between
  (e.g. need to do X to start a container), etc. Please describe the details.

* **Will enabling / using this feature result in non-negligible increase of
resource usage (CPU, RAM, disk, IO, ...) in any components?** No
  Things to keep in mind include: additional in-memory state, additional
  non-trivial computations, excessive access to disks (including increased log
  volume), significant amount of data sent and/or received over network, etc.
  This through this both in small and large cases, again with respect to the
  [supported limits].

* **Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?** No

### Troubleshooting

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.

_This section must be completed when targeting beta graduation to a release._

* **How does this feature react if the API server and/or etcd is unavailable?**

  - If the API is unavailable prior to the resize request being made, the request wil not go through.
  - If the API is unavailable before the Kubelet observes the resize, the request will remain pending until the Kubelet sees it.
  - If the API is unavailable after the Kubelet observes the resize, then the pod status may not
    accurately reflect the running pod state. The Kubelet tracks the resource state internally.

* **What are other known failure modes?**

  - Race condition with scheduler can cause pods to be rejected with `OutOfCPU` or
    `OutOfMemory`.
  - Race condition with pod startup on version-skewed clusters can lead to pods running in an
    unknown resource configuration. See [Version Skew Strategy](#version-skew-strategy) for more
    details.
  - Shrinking memory limit below memory usage can leave the resize in an `InProgress` state
    indefinitely. Race conditions around reading usage info could cause container to OOM on resize.

* **What steps should be taken if SLOs are not being met to determine the problem?**

  - Investigate Kubelet and/or container runtime logs.

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos

## Implementation History

- 2018-11-06 - initial KEP draft created
- 2019-01-18 - implementation proposal extended
- 2019-03-07 - changes to flow control, updates per review feedback
- 2019-08-29 - updated design proposal
- 2019-10-25 - Initial CRI changes KEP draft created
- 2019-10-25 - update key open items and move KEP to implementable
- 2020-01-06 - API review suggested changes incorporated
- 2020-01-13 - Test plan and graduation criteria added
- 2020-01-14 - CRI changes test plan and graduation criteria added
- 2020-01-21 - Graduation criteria updated per review feedback
- 2020-11-06 - Updated with feedback from reviews
- 2020-12-09 - Add "Deferred"
- 2021-02-05 - Final consensus on allocatedResources[] and resize[]
- 2022-05-01 - KEP 2273-kubelet-container-resources-cri-api-changes merged with this KEP
- 2023-04-08 - Catch up KEP details to what is actually implemented

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

There are no drawbacks that we are aware of.

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

We considered having scheduler approve the resize. We also considered PodSpec as
the location to checkpoint allocated resources.

### Allocated Resources

If we need allocated resources & limits in the pod status API, the following options have been
considered:

**Option 1: New field "AcceptedResources"**

We can't change the type of the existing field, so instead we
introduce a new field `ContainerStatus.AcceptedResources` of type `ResourceRequirements`, to track both
allocated requests & limits, and remove the old `AllocatedResources` field. For consistency, we also
add `AcceptedResourcesStatus` and remove `AllocatedResourcesStatus`.

Pros:
- Consistent type across PodSpec.Container.Resources (desired), ContainerStatus.AcceptedResources (allocated), and ContainerStatus.Resources (actual)
- If/when we implement in-place resize for DRA resources, Claims are already included in the API.
- No need for local checkpointing, if the Kubelet can read back from the status API.

Cons:
- No path to beta without waiting a release (new fields need to start in alpha)
- Extra code churn to migrate to the new fields
- Inconsistent with PVC API (which has AllocatedResources), and the Node Allocatable resources.
- The Claims field is currently unnecessary, and needs its behavior defined.

Variations:
- Use an alternative type that is a subset of the ResourceRequirements type without Claims, adding back Claims only when needed.
- Field name ContainerStatus.Allocated, as a struct holding both the allocated resources and and the allocated resource status

**Option 2: New field "AllocatedResourceLimits"**

Rather than changing the type with a new field, we could use a flattened API structure and just add
`ContainerStatus.AllocatedResourceLimits` alongside `AllocatedResources` (requests).

Pros:
- Preserves the "Allocated" name
- Less churn to implement
- Does not prematurely import Claims into the problem space

Cons:
- Uglier API: unnested fields adds noise to the documentation and makes it harder for humans to read the status.
- Inconsistent types between Allocated* and Resources
- We will want to mirror the same structure in the PodStatus for pod-level resources, and may eventually want to add AllocatedResourceClaims for DRA resource resize

**Option 3: Pod-level "AllocatedResources", drop container-level API**

If we assume that outside the node, controllers and people only care about pod-level allocated
resources, then we could drop the container-level allocated resources, and just add a
`PodStatus.AllocatedResources` field of type `ResourceRequirements`. The Kubelet still needs to track
container-level allocation, and would use a checkpoint to do so.

Pros:
- Minimalist API, without unnecessary or redundant information
- Preserve the "Allocated" name while still getting the advantages of type consistency
- Similar path to beta as Option 2

Cons:
- Requires long-term checkpointing to track container allocation
- Extra risk in assuming nothing outside the node ever needs to know container-level allocated
  resources, such as for hierarchical or container/task level scheduling.
- No observability into container allocation
- No recourse if erroneous values are reported by the runtime
