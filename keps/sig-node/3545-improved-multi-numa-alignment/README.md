# KEP-3545: Improved multi-numa alignment in Topology Manager

<!--
This is the title of your KEP. Keep it short, simple, and descriptive. A good
title can help communicate what the KEP is and should be considered as part of
any review.
-->

<!--
A table of contents is helpful for quickly jumping to sections of a KEP and for
highlighting any additional information provided beyond the standard KEP
template.

Ensure the TOC is wrapped with
  <code>&lt;!-- toc --&rt;&lt;!-- /toc --&rt;</code>
tags, and then generate with `hack/update-toc.sh`.
-->

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
- [Proposal](#proposal)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Proposed Change](#proposed-change)
  - [Implementation strategy](#implementation-strategy)
  - [Calculating average distance](#calculating-average-distance)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Alpha to Beta Graduation](#alpha-to-beta-graduation)
    - [Beta to G.A Graduation](#beta-to-ga-graduation)
  - [Graduation Criteria of Options](#graduation-criteria-of-options)
    - [Graduation of Options to <code>Beta-quality</code> (non-hidden)](#graduation-of-options-to-beta-quality-non-hidden)
    - [Graduation of Options from <code>Beta-quality</code> to <code>G.A-quality</code>](#graduation-of-options-from-beta-quality-to-ga-quality)
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
- [Infrastructure Needed (Optional)](#infrastructure-needed-optional)
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

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [x] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [x] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [x] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
- [x] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [x] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

We propose an enhancement to the `TopologyManager` that allows it to favor sets of NUMA nodes with shorter distance between nodes when making admission decisions. The proposed enhancement is only applicable when comparing sets of NUMA nodes that are equal in size.

## Motivation

To support latency-critical execution and high-throughput applications (including those running on Kubernetes), NUMA locality (topology) of resources assigned to the workloads is crucial. The best NUMA locality can be achieved by minimizing the number of NUMA nodes required to execute the workload, and minimizing the NUMA distance between those nodes. NUMA distance is a metric which exposes relative physical distance between NUMA cells which relates to latency that is introduced when a CPU needs to access resources from different NUMA nodes. The distance is always within range 10-254 and for a local access its value is 10.

At present, the `TopologyManager` provides policies to align resources on either a single NUMA node or the minimum number of NUMA nodes (in cases where more than one NUMA node is required). However, the `TopologyManager` is not aware of NUMA distances and does not take them into account when making admission decisions.

This limitation surfaces in multi-socket, as well as single-socket multi NUMA systems (commonly present in AMD hardware), and can cause significant performance degradation in latency-critical execution and high-throughput applications if the `TopologyManager` decides to align resources on non-adjacent NUMA nodes.


### Goals

* Introduce a new kubelet flag `topology-manager-policy-options` for passing options to a TopologyManager policy to modify its behaviour.
* Enable `TopologyManager` to prefer sets of NUMA nodes with shorter NUMA distances for all TopologyManager policies
* Preserve all other properties of all policies

## Proposal

### Risks and Mitigations

| Risk                                             | Impact | Mitigation                                                                                                                                                                               |
|--------------------------------------------------|--------|------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------|
| Bugs in the implementation lead to kubelet crash | High   | Disable the policy option and restart the kubelet. The workload will run but resources allocation can be spread among NUMA nodes which are non-adjacent what can lead to higher latency. |

## Design Details


### Proposed Change

We propose to
- add a new flag in Kubelet called `TopologyManagerPolicyOptions` in the kubelet config or command line argument called `topology-manager-policy-options` which allows the user to specify the Topology Manager policy option.
- add a new topology manager option called `prefer-closest-numa-nodes`; if present, this option will enable further refinements of the existing `restricted` and `best-effort` policies, this option has no effect for `none` and `single-numa-node` policies.

Best-effort or restricted policy chooses resources that will fit into the smallest number of NUMA nodes.
This enhancement wants to change how the best hint is chosen from choosing narrower `NUMANodeAffinity` bitmask to choosing the one that is the narrowest and has the shortest average distance between NUMA nodes.

To summarize properties of `prefer-closest-numa-nodes` policy:

* Preserve all properties of `best-effort` and `restricted` policies
* Choose `NUMANodeAffinity` bitmask which is the narrowest and if the bitmask width is the same prefer the bitmask with smaller average distance.

When `prefer-closest-numa-nodes` policy is enabled, we need to retrieve information regarding distances between NUMA nodes.
Right now Topology manager discovers Node layout using [CAdvisor API](https://github.com/google/cadvisor/blob/master/info/v1/machine.go#L40), which already exposes NUMA distance between nodes since version `v0.46.0`.

### Implementation strategy

- Introduce new flag in Kubelet called `topology-manager-policy-options`, which when specified with `prefer-closest-numa-nodes` will modify the behavior of `best-effort` and `restricted` policy to pick NUMA nodes based on average distance between them.
- The `TopologyManagerPolicyOptions` flag is propagated to `ContainerManager` and later to `TopologyManager`.
- Enable `TopologyManager` NUMA distances discovery:
  - Temporarily add distances discovery logic into `kubelet` (similarly to [the introduction](https://github.com/kubernetes/kubernetes/commit/ecc14fe661c22f5da967a7ff50cfb3aead60905b) of `GetNUMANode()`).
  - Once `cadvisor` exposes the NUMA distance information through `MachineInfo` ([PR](https://github.com/google/cadvisor/pull/3179) exposing distance in `cadvisor` is merged, waiting for it to be released), remove the logic out of `kubelet` (similarly to [the removal](https://github.com/kubernetes/kubernetes/commit/a047e8aa1b705bb7e5be881fb63cf90a218b60d0) of `GetNUMANode()`).
  - The removal of the logic `kubelet` is [an Alpha to Beta graduation criteria](#alpha-to-beta-graduation).
- When `TopologyManager` is being created it discovers distances between NUMA nodes and stores them inside `manager` struct. This is temporary until `distance` information lands in `cadvisor`.
- Pass `TopologyManagerPolicyOptions` to best-effort and restricted policy. When this is specified best-hint is picked based on average distance between NUMA nodes. This would require modification to `compareHints` function to change how the best hint is calculated:

```go

// NUMADistance is a matrix representing distances between NUMA nodes
type NUMADistance [][]uint64

func (n NUMADistance) CalculateAvgDistanceFor(bm bitmask.BitMask) int {
   // implementation
   return avgDistance
}

type PolicyOpts struct {
    PreferClosestNuma bool
    Distances NUMADistance
}

func compareHints(bestNonPreferredAffinityCount int, current *TopologyHint, candidate *TopologyHint, policyOpts *PolicyOpts) *TopologyHint {
    /*
    ...
    */

    if current.Preferred && candidate.Preferred {
        if candidate.NUMANodeAffinity.IsNarrowerThan(current.NUMANodeAffinity) {
            return candidate
        }
        if policyOpts.PreferClosestNuma && candidate.NUMANodeAffinity.IsEqual(current.NUMANodeAffinity) {
            candidateDistance := policyOpts.Distances.CalculateAvgDistanceFor(candidate)
            currentDistance := policyOpts.Distances.CalculateAvgDistanceFor(current)
            // candidate avg distance is lower
            if candidateDistance < currentDistance {
                return candidate
            }

            return current
        }
    }

    /*
    ...
    */
}

```

### Calculating average distance

Let's consider following distance table:

|  node/node | node0 | node1 | node2 | node3 |
| ---------- | ----- | ----- | ----- | ----- |
| node0      | 10    | 11    | 12    | 12    |
| node1      | 11    | 10    | 12    | 12    |
| node2      | 12    | 12    | 10    | 11    |
| node3      | 12    | 12    | 11    | 10    |


If resources cannot fit onto one NUMA node, the new policy option will prefer hints with bitmasks that have lower average distance between NUMA nodes. Such bitmasks include:

* 1100 -> (10 + 11 + 11 + 10) /4 = 10.5
* 1010 -> (10 + 12 + 10 + 12) /4 = 11
* 1001 -> (10 + 12 + 10 + 12) /4 = 11
* 0011 -> (10 + 11 + 10 + 11 ) /4 = 10.5
* 0110 -> (10 + 12 + 10 + 12 ) /4 = 11

So the bitmasks 1100 and 0011 have the lowest average distance between NUMA nodes.

If we consider a system with 8 NUMA nodes:

|  node/node | node0 | node1 | node2 | node3 | node4 | node5 | node6 | node7 |
| ---------- | ----- | ----- | ----- | ----- | ----- | ----- | ----- | ----- |
| node0      | 10    | 11    | 12    | 12    | 30    | 30    | 30    | 30    |
| node1      | 11    | 10    | 12    | 12    | 30    | 30    | 30    | 30    |
| node2      | 12    | 12    | 10    | 11    | 30    | 30    | 30    | 30    |
| node3      | 12    | 12    | 11    | 10    | 30    | 30    | 30    | 30    |
| node4      | 30    | 30    | 30    | 30    | 10    | 11    | 12    | 12    |
| node5      | 30    | 30    | 30    | 30    | 11    | 10    | 12    | 12    |
| node6      | 30    | 30    | 30    | 30    | 12    | 12    | 10    | 11    |
| node7      | 30    | 30    | 30    | 30    | 12    | 12    | 11    | 10    |

And the following bitmasks:

* 10001000 -> (10 + 30 + 10 +30) /4 = 20
* 11100000 -> (10 + 11 + 12 + 10 + 11 + 12 + 12 + 12 + 10) / 6 = 16.7

In the second case, even though the average distance is lower, the number of NUMA nodes is larger (meaning that the first case should always be preferred).

### Test Plan

- [x] We understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates


##### Unit tests

- `k8s.io/kubernetes/pkg/kubelet/cm/topologymanager`: `09-23-2022` - `92.4`
- `k8s.io/kubernetes/pkg/kubelet/cm/topologymanager`: `06-12-2023` - `93.2`
- `k8s.io/kubernetes/pkg/kubelet/cm/topologymanager`: `09-30-2024` - `92.4`

##### Integration tests

 These cases will be added in the existing integration tests:
  - Feature gate enable/disable tests
  - `prefer-closest-numa-nodes` policy option works as expected. When policy option is enabled
     - `Merge` prefers hints with the lowest average distance between NUMA nodes.
  - Verify no significant performance degradation

##### e2e tests

These cases will be added in the existing e2e tests:
  - Feature gate enable/disable tests
  - `prefer-closest-numa-nodes` policy option works as expected.

### Graduation Criteria

#### Alpha
- [X] Implement the new policy option.
- [X] Temporarily include NUMA distances discovery logic in kubelet code.
- [X] Add proper e2e node tests.

#### Alpha to Beta Graduation
- [X] Gather feedback from the consumer of the `prefer-closest-numa-nodes` policy option.
- [X] Remove NUMA distances discovery logic from kubelet in favor of updated cAdvisor.
- [X] No major bugs reported in the previous cycle.

#### Beta to G.A Graduation
- [X] Allowing time for feedback (1 year).
- [X] Risks have been addressed.
- [X] Presence of E2E test validating the described behavior on 4+ NUMA nodes system.

### Graduation Criteria of Options

In 1.26 we are releasing this feature to Alpha. We propose the following management of TopologyManager policy options graduation:

- `TopologyManagerPolicyOptions` for enabling/disabling the entire feature. As this is an alpha feature, this feature gate would be disabled by default.
  Explicitly enabling `TopologyManagerPolicyOptions` feature gate provides us the ability to supply `TopologyManagerPolicyOptions` or `topology-manager-policy-options` flag in Kubelet.

- `TopologyManagerPolicyAlphaOptions` is not enabled by default. Topology Manager alpha options (only one as of 1.26), are hidden by default
   and only when both the `TopologyManagerPolicyOptions` and the `TopologyManagerPolicyAlphaOptions` feature gates are enabled will the user be able to use alpha options.

- `TopologyManagerPolicyBetaOptions` is not enabled by default. Topology Manager beta options (none as of 1.26), are hidden by default
  and only when both the `TopologyManagerPolicyOptions`and the `TopologyManagerPolicyBetaOptions` feature gates are enabled will the user be able to use beta options.


When an option graduates, its visibility should be moved to be controlled by the corresponding feature-flag.
The introduction of these feature gates gives us the ability to move the option to beta and later stable without implying that all available options are stable.
This approach is similliar to graduation criteria for `CPUManagerPolicyOptions` introduced [here](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2625-cpumanager-policies-thread-placement#graduation-criteria-of-options).

In 1.28 this feature is being promoted to Beta. We propose following changes to TopologyManager policy options default visibility:

- `TopologyManagerPolicyOptions` feature flag for enabling/disabling the entire feature will be enabled by default.
- `TopologyManagerPolicyBetaOptions` feature flag for enabling/disabling beta options will be enabled by default.
- `prefer-closest-numa-nodes` will be moved to Beta options.

In 1.32 this feature is being promoted to Beta. We propose following changes to TopologyManager policy options default visibility:

- `prefer-closest-numa-nodes` will be moved to stable options.

The graduation Criteria of options is described below:

#### Graduation of Options to `Beta-quality` (non-hidden)
- [X] Gather feedback from the consumer of the policy option.
- [X] No major bugs reported in the previous cycle.

#### Graduation of Options from `Beta-quality` to `G.A-quality`
- [X] Allowing time for feedback (1 year) on the policy option.
- [X] Risks have been addressed.


### Upgrade / Downgrade Strategy

We expect no impact. The new policiy options are opt-in.

### Version Skew Strategy

No changes needed

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate names:
    - `TopologyManagerPolicyAlphaOptions`
    - `TopologyManagerPolicyBetaOptions`
    - `TopologyManagerPolicyOptions`
  - Components depending on the feature gate: kubelet
- [x] Change the kubelet configuration to set a `TopologyManager` policy to `restricted` or `best-effort` and a `TopologyManagerPolicyOptions` to `prefer-closest-numa-nodes`
  - Will enabling / disabling the feature require downtime of the control
    plane?
    No.
  - Will enabling / disabling the feature require downtime or reprovisioning
    of a node?
    Yes, a kubelet restart is required.

###### Does enabling the feature change any default behavior?

- No, it makes the behaviour of the TopologyManager restricted and best-effort policies to choose NUMA nodes based on distance only if proper `kubelet` flag is provided.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

- Yes, disabling the `TopologyManagerPolicyOptions` feature gate shuts down the feature completely.
- Yes, through kubelet configuration - disable given policy option.

###### What happens if we reenable the feature if it was previously rolled back?

- No changes. The allocation of resources won't be changed for exiting containers. It will change for new containers only.

###### Are there any tests for feature enablement/disablement?

- There will be specific e2e test demonstrating that default behaviour is preserved when feature gate is disabled.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

Kubelet may fail to start. The kubelet may crash.

###### What specific metrics should inform a rollback?

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

N/A

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

Inspect the kubelet configuration of the nodes: check feature gate and usage of the new option.

###### How can someone using this feature know that it is working for their instance?

In order to verify this feature is working, one should:
Pick a node with at least 4 NUMA nodes and with at least 3 different distance values between them.
Ensure no other pods are running on given node.
Launch pod that requires resources from 2 NUMA nodes. For example if there are 8 CPUs per NUMA node, ask for 16 CPUs.
Verify that CPUs are assigned from NUMA nodes with lowest distance.
To verify the list of CPUs allocated to the container, their NUMA nodes and the distance between them:
- `exec` into uthe container and run `taskset -cp 1` to retrieve the list of CPUs assigned to PID 1 and then `numactl -H` to retrieve information which CPU is on which NUMA node and what is the distance between nodes. Assuming those commands are available inside the POD.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

N/A

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

No change.

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

N/A.

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

No.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

N/A.

###### What are other known failure modes?

There are 2 scenarios where Kubelet may fail to start due to using this feature:

- Bad policy option name or using policy option without enabling appropriate feature flag. we are emitting appropriate error message for this case,
  Kubelet will fail to start and print error message what happened. To recover one just have to provide fix policy option name or disable/enable feature flags.

- Cadvisor is not exposing distances for NUMA domains. In this case Kubelet will fail with `error getting NUMA distances from cadvisor` message.
  Reading NUMA distances is only performed when `prefer-closest-numa-nodes` option is specified.
  To recover one has to either disable `TopologyManagerPolicyOptions` feature-flag or stop using `prefer-closest-numa-nodes` option.

###### What steps should be taken if SLOs are not being met to determine the problem?

N/A.

## Implementation History

- 2021-09-26: KEP created
- 2023-06-12: KEP updated for Beta release
- 2024-09-30: KEP updated for Stable release


## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->
Adds complexity to Topology Manager.

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->

Adding new Topology Manager policy similar to `best-effort` which also takes NUMA distance into consideration.
This approach was ruled out as NUMA distance can be an important factor not only for `best-effort` but also `restricted` policy.

So introducing a mechanism similar to CPUManagerPolicyOptions was an already familiar concept that could influence logic of multiple policies.

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->

To be able to do e2e testing it would be required that CI machines with at least 4 NUMA nodes exist to be able to use `prefer-closest-numa-nodes` policy option properly.
The [effort to add machines for sig-node e2e tests](https://github.com/kubernetes/k8s.io/issues/7339) is still open and it shouldn't block this KEP from graduating to GA.
