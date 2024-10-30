<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [x] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [x] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [x] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [x] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [x] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->
# KEP-3288: Split Stdout and Stderr Log Stream of Container

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
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
  - [Notes/Constraints/Caveats (Optional)](#notesconstraintscaveats-optional)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Changes of kube-apiserver](#changes-of-kube-apiserver)
  - [Changes of kubelet](#changes-of-kubelet)
  - [Changes of kubectl](#changes-of-kubectl)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
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
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests for meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [x] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

Currently, kubelet actually [has the potential](https://github.com/kubernetes/kubernetes/blob/69cabe778bccfc5f4fa9b6788c9def005e7d959d/pkg/kubelet/kuberuntime/logs/logs.go#L281)
to return certain log stream of a container, but this ability is not exposed to the user.
This proposal enables users to view certain log stream of a container,
and aims to extend kubelet and api-server with the ability to return certain
container log stream.

## Motivation

Users could fetch logs of the container, but now kubelet always returns
combined stdout and stderr logs, which is not convenient for users who only want to
view certain log stream.
Meanwhile, [many users](https://github.com/kubernetes/kubernetes/issues/28167) have been interested in the ability to
retrieve certain log stream of a container, so it is great to implement this long-wanted feature.

### Goals

- Enable api-server to return specific log stream of a container
- Enable users to fetch specific log stream of a container

### Non-Goals

- Supporting the combination of a specific `Stream`(stdout or stderr) and `TailLines` in the first iteration. However, 
if `Stream` is set to `all`, both `Stream` and `TailLines` can be specified together.
- Implementing a new log format

## Proposal

Add a new field `Stream` to `PodLogOptions` to allow users to indicate which log stream
they want to retrieve. To maintain backward compatibility, if this field is not set,
the combined stdout and stderr from the container will be returned to users.

Users are enabled to fetch the stderr log stream of a given container using kubectl as following:
```bash
kubectl logs --stream=stderr -c container pod
```

### User Stories

Bob runs a service "foo" that continuously prints informational messages to stdout,
and prints warnings/errors to stderr. When the service "foo" is misbehaving, Bob wants
to know what is going on as soon as possible, so he checks the logs of the service "foo",
but the error message is not easy to notice because the apiserver always returns the combined
stdout and stderr from the container.

This problem could be resolved if Bob is able to specify that he only wants stderr.

### Notes/Constraints/Caveats (Optional)

It can be tricky when `Stream` and `TailLines` of `PodLogOptions` are both specified.

For instance, users might want to fetch the last 10 lines of the stderr log stream for a container, but
this is prohibitively expensive to implement from kubelet's perspective:

At present, container's logs are stored in an encoded format, either "json-file" or "cri" format:

```
# json-file
{"log":"out1\n","stream":"stdout","time":"2024-08-20T09:31:37.985370552Z"}
{"log":"err1\n","stream":"stderr","time":"2024-08-20T09:31:37.985370552Z"}

# cri
2016-10-06T00:17:09.669794202Z stdout F out1
2016-10-06T00:17:09.669794202Z stderr F err1
```

Please note that the log stream is encoded in each log line. Let's see what will happen if kubelet needs to return 
the last 10 lines of the stderr log stream:
1. Kubelet has to decode each line of the log file from the bottom to determine whether it is from the stderr stream or not, 
which is a CPU-intensive operation in practice.
2. What's worse, once kubelet identifies that a line is from the stderr stream, it must keep track of the matched lines 
until a specified number of lines are found. This can be time-consuming, especially when only few lines of stderr logs 
amidst a large number of stdout logs.

In conclusion, it is not practical for kubelet to return the last N lines of a specific log stream.

Alternatively, kubelet could return logs that match the given stream within the last N lines. For example, consider the following logs:
```
{"log":"out1\n","stream":"stdout","time":"2024-08-20T09:31:37.985370552Z"}
{"log":"err1\n","stream":"stderr","time":"2024-08-20T09:31:37.985370552Z"}
{"log":"out2\n","stream":"stdout","time":"2024-08-20T09:31:37.985370552Z"}
{"log":"err2\n","stream":"stderr","time":"2024-08-20T09:31:37.985370552Z"}
```

If users run `kubectl logs --stream=stderr --tail=2 pod`, kubelet would only return the following:
```
err2
```

This approach is more efficient, as kubelet only needs to parse a deterministic number of log lines once,
rather than potentially all of them. However, this may go against users' expectations and could lead to confusion.

Taking all these considerations into account, I propose that we do not support the combination of specific `Stream` 
and `TailLines`in the first iteration. 
Additionally, the apiserver should validate the `PodLogOptions` to ensure that a specific `Stream` and `TailLines` 
are mutually exclusive.

### Risks and Mitigations


## Design Details

### Changes of kube-apiserver

Add a new field `Stream` to `k8s.io/kubernetes/pkg/apis/core.PodLogOptions`:

```go
// LogStreamType represents the desired log stream type.
type LogStreamType string

const (
	// LogStreamTypeStdout is the stream type for stdout.
	LogStreamTypeStdout LogStreamType = "stdout"
	// LogStreamTypeStderr is the stream type for stderr.
	LogStreamTypeStderr LogStreamType = "stderr"
	// LogStreamTypeAll represents the combined stdout and stderr.
	LogStreamTypeAll    LogStreamType = "all"
)

// PodLogOptions is the query options for a Pod's logs REST call
type PodLogOptions struct {
	...
	// If set to "stdout" or "stderr", return the given log stream of the container.
	// If set to "all" or not set, the combined stdout and stderr from the container is returned.
	// Available values are: "stdout", "stderr", "all", "" (empty string).
	// +optional
	Stream LogStreamType
}
```

When users want to query certain stream from container, they need to add a new query named `stream`
to the URL, i.e. `/api/v1/namespaces/default/pods/foo/log?stream=stderr&container=nginx`.
Then the kube-apiserver is able to know the desired `stream` and passes it to the kubelet.

To tell kubelet which stream to return, we need to update the [`LogLocation`](https://github.com/kubernetes/kubernetes/blob/ba502ee555924a49c1455b0d3fa96ec1e787715d/pkg/registry/core/pod/strategy.go#L398) function to make it be aware of
the new parameter:

```go
// LogLocation returns the log URL for a pod container. If opts.Container is blank
// and only one container is present in the pod, that container is used.
func LogLocation(
	ctx context.Context, getter ResourceGetter,
	connInfo client.ConnectionInfoGetter,
	name string,
	opts *api.PodLogOptions,
) (*url.URL, http.RoundTripper, error) {
	...
	params := url.Values{}
	// validate stream
	switch opts.Stream {
	case LogStreamTypeStdout, LogStreamTypeStderr, LogStreamTypeAll, "":
	default:
		return nil, nil, errors.NewBadRequest(fmt.Sprintf("invalid container log stream %s", opts.Stream))
	}
	// keep backwards compatibility
	if opts.Stream == "" {
		opts.Stream = LogStreamTypeAll
	}
	params.Add("stream", string(opts.Stream))
	...
}
```

### Changes of kubelet

Add a new field `Stream` to `k8s.io/api/core/v1.PodLogOptions`:

```go
// LogStreamType represents the desired log stream type.
// +enum
type LogStreamType string

const (
	// LogStreamTypeStdout is the stream type for stdout.
	LogStreamTypeStdout LogStreamType = "stdout"
	// LogStreamTypeStderr is the stream type for stderr.
	LogStreamTypeStderr LogStreamType = "stderr"
	// LogStreamTypeAll represents the combined stdout and stderr.
	LogStreamTypeAll    LogStreamType = "all"
)
// PodLogOptions is the query options for a Pod's logs REST call.
type PodLogOptions struct {
	...
	// If set, return the given log stream of the container.
	// Otherwise, the combined stdout and stderr from the container is returned.
	// +optional
	Stream LogStreamType
}

```

In the [`getContainerLogs`](https://github.com/kubernetes/kubernetes/blob/ba502ee555924a49c1455b0d3fa96ec1e787715d/pkg/kubelet/server/server.go#L610) method of `k8s.io/kubernetes/pkg/kubelet/server.Server`, we examine the `Stream` field
of `PodLogOptions` to decide which stream to return:

```go
// getContainerLogs handles containerLogs request against the Kubelet
func (s *Server) getContainerLogs(request *restful.Request, response *restful.Response) {
	...
	fw := flushwriter.Wrap(response.ResponseWriter)
	var stdout, stderr io.Writer
	switch logOptions.Stream {
	case corev1.LogStreamTypeStdout:
		stdout, stderr = fw, io.Discard
	case corev1.LogStreamTypeStderr:
		stdout, stderr = io.Discard, fw
	case corev1.LogStreamTypeAll:
		stdout, stderr = fw, fw
	}
	response.Header().Set("Transfer-Encoding", "chunked")
	if err := s.host.GetKubeletContainerLogs(ctx, kubecontainer.GetPodFullName(pod), containerName, logOptions, stdout, stderr); err != nil {
		response.WriteError(http.StatusBadRequest, err)
		return
	}
}
```

We also need to modify the [ReadLogs](https://github.com/kubernetes/kubernetes/blob/198dd7668a19c8749fe45d521d748ef3c653aa73/pkg/kubelet/kuberuntime/logs/logs.go#L281)
function to make it be able to filter out the unwanted stream.


### Changes of kubectl

Add a new flag `--stream`, whose value defaults to "all", to `kubectl logs`, hence users are able to specify the
log stream to return.

### Test Plan

<!--
**Note:** *Not required until targeted at a release.*

Consider the following in developing a test plan for this enhancement:
- Will there be e2e and integration tests, in addition to unit tests?
- How will it be tested in isolation vs with other components?

No need to outline all of the test cases, just the general strategy. Anything
that would count as tricky in the implementation, and anything particularly
challenging to test, should be called out.

All code is expected to have adequate tests (eventually with coverage
expectations). Please adhere to the [Kubernetes testing guidelines][testing-guidelines]
when drafting this test plan.

[testing-guidelines]: https://git.k8s.io/community/contributors/devel/sig-testing/testing.md
-->

[x] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

<!--
Based on reviewers feedback describe what additional tests need to be added prior
implementing this enhancement to ensure the enhancements have also solid foundations.
-->

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit
This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

- `k8s.io/kubernetes/pkg/registry/core/pod/strategy.go`: `2022-06-07` - `59.5%`
- `k8s.io/kubernetes/pkg/kubelet/server/server.go`: `2022-06-07` - `67.8%`
- `k8s.io/kubernetes/pkg/kubelet/kuberuntime/logs/logs.go`: `2022-06-07` - `71%`

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.
For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

Add unit tests to `pkg/kubelet/...` and `pkg/registry/core/pod/...` to make sure kubelet
and kube-apiserver is behaving as expected.

Specifically, when `TailLines` and `Stream` are specified in `PodLogOptions`, we need to ensure that
the counting of TailLines happens properly and only logs from a specified stream were counted.

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.
For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

Add test case and conformance test to `test/e2e/common/node/` and `test/e2e/kubectl/`

### Graduation Criteria

<!--
**Note:** *Not required until targeted at a release.*

Define graduation milestones.

These may be defined in terms of API maturity, [feature gate] graduations, or as
something else. The KEP should keep this high-level with a focus on what
signals will be looked at to determine graduation.

Consider the following in developing the graduation criteria for this enhancement:
- [Maturity levels (`alpha`, `beta`, `stable`)][maturity-levels]
- [Feature gate][feature gate] lifecycle
- [Deprecation policy][deprecation-policy]

Clearly define what graduation means by either linking to the [API doc
definition](https://kubernetes.io/docs/concepts/overview/kubernetes-api/#api-versioning)
or by redefining what graduation means.

In general we try to use the same stages (alpha, beta, GA), regardless of how the
functionality is accessed.

[feature gate]: https://git.k8s.io/community/contributors/devel/sig-architecture/feature-gates.md
[maturity-levels]: https://git.k8s.io/community/contributors/devel/sig-architecture/api_changes.md#alpha-beta-and-stable-versions
[deprecation-policy]: https://kubernetes.io/docs/reference/using-api/deprecation-policy/

Below are some examples to consider, in addition to the aforementioned [maturity levels][maturity-levels].
-->

#### Alpha

- Feature implemented behind a feature flag
- Add unit and e2e tests for the feature.

#### Beta

- Solicit feedback from the Alpha.
- Ensure tests are stable and passing.

<!--

#### GA

- N examples of real-world usage
- N installs
- More rigorous forms of testing—e.g., downgrade tests and scalability tests
- Allowing time for feedback

**Note:** Generally we also wait at least two releases between beta and
GA/stable, because there's no opportunity for user feedback, or even bug reports,
in back-to-back releases.

**For non-optional features moving to GA, the graduation criteria must include
[conformance tests].**

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md

#### Deprecation

- Announce deprecation and support policy of the existing flag
- Two versions passed since introducing the functionality that deprecates the flag (to address version skew)
- Address feedback on usage/changed behavior, provided on GitHub issues
- Deprecate the flag
-->

### Upgrade / Downgrade Strategy

<!--
If applicable, how will the component be upgraded and downgraded? Make sure
this is in the test plan.

Consider the following in developing an upgrade/downgrade strategy for this
enhancement:
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to maintain previous behavior?
- What changes (in invocations, configurations, API use, etc.) is an existing
  cluster required to make on upgrade, in order to make use of the enhancement?
-->

There is no extra work required for users to maintain previous behavior, the changes
caused by this enhancement are backwards compatible.

To make use of the enhancement, users will need to update the `kube-apiserver` and `kubelet`
to at least `v1.32` and turn on feature gate `SplitStdoutAndStderr` in both components.

### Version Skew Strategy

<!--
If applicable, how will the component handle version skew with other
components? What are the guarantees? Make sure this is in the test plan.

Consider the following in developing a version skew strategy for this
enhancement:
- Does this enhancement involve coordinating behavior in the control plane and
  in the kubelet? How does an n-2 kubelet without this feature available behave
  when this feature is used?
- Will any other components on the node change? For example, changes to CSI,
  CRI or CNI may require updating that component before the kubelet.
-->

TBD

## Production Readiness Review Questionnaire

<!--

Production readiness reviews are intended to ensure that features merging into
Kubernetes are observable, scalable and supportable; can be safely operated in
production environments, and can be disabled or rolled back in the event they
cause increased failures in production. See more in the PRR KEP at
https://git.k8s.io/enhancements/keps/sig-architecture/1194-prod-readiness.

The production readiness review questionnaire must be completed and approved
for the KEP to move to `implementable` status and be included in the release.

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

<!--
This section must be completed when targeting alpha to a release.
-->

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: SplitStdoutAndStderr
  - Components depending on the feature gate: kubelet, kube-apiserver

###### Does enabling the feature change any default behavior?

No. If the query parameter `stream` in the url of fetching logs from kube-apiserver is empty or not set,
combined stdout and stderr is returned, which is the default behavior.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes.

###### What happens if we reenable the feature if it was previously rolled back?

No harm, It becomes enabled again after the `kubelet` and `kube-apiserver` restart.
The log files do not change when the feature is on compared to when it is off.

###### Are there any tests for feature enablement/disablement?

Yes, unit tests for the feature when enabled and disabled will be
implemented in both kubelet and api server.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [ ] Events
  - Event Reason:
- [ ] API .status
  - Condition name:
  - Other field:
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

N/A

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

N/A

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

N/A

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->
No.

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

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

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->
2022-05-01: KEP opened

2022-06-08: KEP marked implementable

## Drawbacks

<!--
Why should this KEP _not_ be implemented?
-->

## Alternatives

<!--
What other approaches did you consider, and why did you rule them out? These do
not need to be as detailed as the proposal, but should include enough
information to express the idea and why it was not acceptable.
-->
Instead of filtering log stream on the server side, we could return a stream of CRI-format logs,
something like the following:
```
2016-10-06T00:17:09.669794202Z stdout P log content 1
2016-10-06T00:17:09.669794203Z stderr F log content 2
```
so that we could demultiplex the log stream on the client side.

The main drawback of this approach is that we change the format of the log stream and break backward compatibility,
there is extra overhead to demultiplex the stream on the client side.

## Infrastructure Needed (Optional)

<!--
Use this section if you need things from the project/SIG. Examples include a
new subproject, repos requested, or GitHub details. Listing these here allows a
SIG to get the process for these resources started right away.
-->
