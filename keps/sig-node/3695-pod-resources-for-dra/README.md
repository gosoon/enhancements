 KEP-3695: Extend the PodResources API to include resources allocated by DRA

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [Proposed API](#proposed-api)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
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
<!-- /toc -->

## Release Signoff Checklist

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [x] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [x] (R) KEP approvers have approved the KEP status as `implementable`
- [x] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [x] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md)
- [x] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [x] "Implementation History" section is up-to-date for milestone
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

We propose an enhancement to the PodResources API to include resources allocated by [Dynamic Resource Allocation (DRA)](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/3063-dynamic-resource-allocation). 
This KEP extends [2043-pod-resource-concrete-assigments](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2043-pod-resource-concrete-assigments) and [2403-pod-resources-allocatable-resources](https://github.com/kubernetes/enhancements/tree/master/keps/sig-node/2403-pod-resources-allocatable-resources).

## Motivation

One of the primary motivations for this KEP is to extend the PodResources API to allow node monitoring agents to access information about resources allocated by DRA.

### Goals

- To allow node monitoring agents to know the allocated DRA resources for Pods on a node.
- To allow node components to use the PodResourcesAPI to use the DRA information to develop new features and integrations.

### Non-Goals

To enhance the GetAllocatableResources() call in the PodResources API to account for resources managed by DRA. With DRA there is no standard way to get the capacity, for example.

## Proposal

### Risks and Mitigations

This API is read-only, which removes a large class of risks. The aspects that we consider below are as follows:
- What are the risks associated with the API service itself?
- What are the risks associated with the data itself?

| Risk                                                      | Impact        | Mitigation |
| --------------------------------------------------------- | ------------- | ---------- |
| Too many requests risk impacting the kubelet performances | High          | Implement rate limiting and or passive caching, follow best practices for gRPC resource management. |
| Improper access to the data | Low | Server is listening on a root owned unix socket. This can be limited with proper pod security policies. |

## Design Details

### Proposed API

Our proposal is to extend the existing PodResources gRPC service of the Kubelet
with a repeated `DynamicResource` field in the ContainerResources message. This
new field will contain information about the DRA resource class, the DRA
claim name, claim namespace, and a list of claimed resources allocated by a DRA driver.
Currently, the claim resources only contain a list of CDI Devices but in the future they
may be extended to support plugins that allocate different types of resource (not just CDI Devices).
Additionally, we propose adding a `Get()` method to the existing gRPC service
to allow querying specific pods for their allocated resources.

**Note:** The new `Get()` call is a strict subset of the `List()` call (which
returns the list of PodResources for *all* pods across *all* namespaces in the
cluster). That is, it allows one to specify a specific pod and namespace to
retrieve PodResources from, rather than having to query all of them all at
once. `Get()` returns error if the pod is known to the kubelet, but is terminated.

The full PodResources API (including our proposed extensions) can be seen below:

```protobuf
// PodResourcesLister is a service provided by the kubelet that provides information about the
// node resources consumed by pods and containers on the node
service PodResourcesLister {
    rpc List(ListPodResourcesRequest) returns (ListPodResourcesResponse) {}
    rpc GetAllocatableResources(AllocatableResourcesRequest) returns (AllocatableResourcesResponse) {}
    rpc Get(GetPodResourcesRequest) returns (GetPodResourcesResponse) {}
}

message AllocatableResourcesRequest {}

// AllocatableResourcesResponses contains informations about all the devices known by the kubelet
message AllocatableResourcesResponse {
    repeated ContainerDevices devices = 1;
    repeated int64 cpu_ids = 2;
    repeated ContainerMemory memory = 3;
}

// ListPodResourcesRequest is the request made to the PodResourcesLister service
message ListPodResourcesRequest {}

// ListPodResourcesResponse is the response returned by List function
message ListPodResourcesResponse {
    repeated PodResources pod_resources = 1;
}

// PodResources contains information about the node resources assigned to a pod
message PodResources {
    string name = 1;
    string namespace = 2;
    repeated ContainerResources containers = 3;
}

// ContainerResources contains information about the resources assigned to a container
message ContainerResources {
    string name = 1;
    repeated ContainerDevices devices = 2;
    repeated int64 cpu_ids = 3;
    repeated ContainerMemory memory = 4;
    repeated DynamicResource dynamic_resources = 5;
}

// ContainerMemory contains information about memory and hugepages assigned to a container
message ContainerMemory {
    string memory_type = 1;
    uint64 size = 2;
    TopologyInfo topology = 3;
}

// ContainerDevices contains information about the devices assigned to a container
message ContainerDevices {
    string resource_name = 1;
    repeated string device_ids = 2;
    TopologyInfo topology = 3;
}

// Topology describes hardware topology of the resource
message TopologyInfo {
    repeated NUMANode nodes = 1;
}

// NUMA representation of NUMA node
message NUMANode {
    int64 ID = 1;
}

// DynamicResource contains information about the devices assigned to a container by DRA
message DynamicResource {
    // tombstone: removed in 1.31 because claims are no longer associated with one class
    // string class_name = 1;
    string claim_name = 2;
    string claim_namespace = 3;
    repeated ClaimResource claim_resources = 4;
}

// ClaimResource contains resource information. The driver name/pool name/device name
// triplet uniquely identifies the device. Should DRA get extended to other kinds
// of resources, then device_name will be empty and other fields will get added.
// Each device at the DRA API level may map to zero or more CDI devices.
message ClaimResource {
    repeated CDIDevice cdi_devices = 1 [(gogoproto.customname) = "CDIDevices"];
    string driver_name = 2;
    string pool_name = 3;
    string device_name = 4;
}

// CDIDevice specifies a CDI device information
message CDIDevice {
    // Fully qualified CDI device name
    // for example: vendor.com/gpu=gpudevice1
    // see more details in the CDI specification:
    // https://github.com/container-orchestrated-devices/container-device-interface/blob/main/SPEC.md
    string name = 1;
}

// GetPodResourcesRequest contains information about the pod
message GetPodResourcesRequest {
    string pod_name = 1;
    string pod_namespace = 2;
}

// GetPodResourcesResponse contains information about the pod the devices
message GetPodResourcesResponse {
    PodResources pod_resources = 1;
}
```

Under the hood, retrieval of the information needed to populate the new
`DynamicResource` field will be pulled from an in-memory cache stored within the
`DRAManager` of the kubelet. This is similar to how the fields for
`ContainerDevices` (from the `DeviceManager`) and `cpu_ids` (from the
`CPUManager`) are populated today.

The one difference being that the `DeviceManager` and `CPUManager` checkpoint
the state necessary to fill their in-memory caches, so that it can be
repopulated across a kubelet restart. We will need to add a similar
checkpointing mechanism in the `DRAManager` so that it can repopulate its
in-memory cache as well. This will ensure that the information needed by the
PodResources API is available for all running containers without needing to call
out to each DRA resource driver to retrieve this information on-demand. We will
follow the same pattern used by the `DeviceManager` and `CPUManager` to
implement this checkpointing mechanism.

**Note:** Checkpointing is possible in the `DRAManager` because the set of CDI
devices allocated to a container cannot change across its lifetime (just as the
set of traditional devices injected into a container by the `DeviceMmanager`
cannot change across its lifetime). Moreover, the set of CDI devices that have
been injected into a container are not tied to the "availability" of the DRA
driver that injected them -- i.e. once a DRA driver allocates a set of CDI
devices to a container, that container will have full access to those devices
for its entire lifetime (even if the DRA driver that injected them temporarily
goes offline). In this way, the in-memory cache maintained by the `DRAManager`
will always have the most up-to-date information for all running containers (so
long as checkpointing is added as described to repopulate it across kubelet
restarts).

##### Unit tests

- `k8s.io/kubernetes/pkg/kubelet/apis/podresources`: `10-08-2024` - `75.3%`

##### Integration tests

 These cases will be added in the existing integration tests:
  - Feature gate enable/disable tests.
  - Get API work with DRA and device plugin.
  - List API work with DRA and Device plugin.

##### e2e tests

These cases will be added in the existing e2e tests:
  - Feature gate enable/disable tests.
  - Get API work with DRA and device plugin.
  - List API work with DRA and Device plugin.

[Get](https://github.com/kubernetes/kubernetes/pull/116846): [sig-node-kubelet](https://testgrid.k8s.io/sig-node-kubelet?include-filter-by-regex=PodResources), [triage](https://storage.googleapis.com/k8s-triage/index.html?test=PodResources)

Other improvements are addressed in:
- https://github.com/kubernetes/kubernetes/pull/132028
- https://github.com/kubernetes/kubernetes/pull/132345

### Graduation Criteria

#### Alpha

- [x] Feature implemented behind a feature flag. (https://github.com/kubernetes/kubernetes/pull/115847)
- [x] e2e tests completed and enabled. (https://github.com/kubernetes/kubernetes/pull/116846)

#### Beta

- [x] Gather feedback from consumers of the DRA feature.
  - Integration with the NVIDIA DCGM exporter (https://github.com/NVIDIA/dcgm-exporter/pull/501) to gather per pod Dynamic Resources managed by [k8s-dra-driver-gpu](https://github.com/NVIDIA/k8s-dra-driver-gpu).
- [x] No major bugs reported in the previous cycle.

#### GA

- [ ] Allowing time for feedback (1 year).
- [ ] Risks have been addressed.
- [ ] Add explicit feature enablement/disablement tests, before the feature is turned on by default.
- [ ] Additional test cases are needed to verify Get() behavior in diverse scenarios
  - Pods with multiple containers.
  - Pods that do not use any exclusive resources.
  - Comparison of List() and Get() on returned pods to validate consistency.
  - Pod exists but container name is invalid
  - Get() is called on terminated pods to validate appropriate error handling.

### Upgrade / Downgrade Strategy

With gRPC the version is part of the service name.
Old versions and new versions should always be served and listened by the kubelet.

To a cluster admin upgrading to the newest API version, means upgrading Kubernetes to a newer version as well as upgrading the monitoring component.

To a vendor changes in the API should always be backwards compatible.

### Version Skew Strategy

Kubelet will always be backwards compatible, so going forward existing plugins are not expected to break.

## Production Readiness Review Questionnaire
### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [x] Feature gate (also fill in values in `kep.yaml`)
  - Feature gate name: `DynamicResourceAllocation` is existing feature gate to
  enable / disable DRA feature.
  - Components depending on the feature gate: kube-apiserver, kube-controller-manager,
  kube-scheduler, kubelet
  - Feature gate name: `KubeletPodResourcesDynamicResources` new feature gate to
  enable / disable PodResources API List method to populate `DynamicResource`
  information from the `DRAManager`.
  `DynamicResourceAllocation` feature gate has to be enabled as well.
  - Components depending on the feature gate: kubelet, 3rd party consumers.
  - Feature gate name: `KubeletPodResourcesGet` new feature gate to enable / disable
    PodResources API Get method. In case `DynamicResourceAllocation` or
    the `KubeletPodResourcesDynamicResources` are disabled and `KubeletPodResourcesGet`
    is enabled, the Get method will retrieve resources allocated by device plugins,
    memory and cpus (but omit those allocated by DRA resource drivers).
    In case `KubeletPodResourcesGet`, `DynamicResourceAllocation` and `KubeletPodResourcesDynamicResources`
    are all enabled, the `Get()` method will also retrieve the resources allocated via DRA.
  - Components depending on the feature gate: kubelet, 3rd party consumers.

###### Does enabling the feature change any default behavior?

No.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes, through feature gates.

###### What happens if we reenable the feature if it was previously rolled back?

The API becomes available again. The API is stateless, so no recovery is needed, clients can just consume the data.

###### Are there any tests for feature enablement/disablement?

This e2e test (https://github.com/kubernetes/kubernetes/blob/master/test/e2e_node/podresources_test.go#L1169) will demonstrate that when the feature gate is disabled, the API returns the appropriate error code. The explicit on/off tests are scattered across the existing tests:
- https://github.com/kubernetes/kubernetes/blob/master/test/e2e_node/podresources_test.go#L1115
- https://github.com/kubernetes/kubernetes/blob/master/test/e2e_node/podresources_test.go#L1820


### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback fail? Can it impact already running workloads?

Kubelet may fail to start. The new API may report inconsistent data, or may cause the kubelet to crash.

###### What specific metrics should inform a rollback?

`pod_resources_endpoint_errors_get` - but only with feature gate `KubeletPodResourcesGet` enabled. Otherwise the API will always return a known error.

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

Not Applicable. Because this change:

- Is read-only in the kubelet’s in-memory state.
- Is behind a feature gate, so turning it off simply disables the new endpoints without affecting any existing behavior.

In practice, restart the kubelet with the gate disabled (rollback) or re-enabled (upgrade), and the API behavior reverts or returns without loss of data or consistency. Therefore we don’t need a special upgrade/downgrade test matrix for this KEP.

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

No.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

Look at the `pod_resources_endpoint_requests_list` and `pod_resources_endpoint_requests_get` metric exposed by the kubelet.

###### How can someone using this feature know that it is working for their instance?
Call the PodResources API and see the result.

- [ ] Events
  - Event Reason:
- [ ] API .status
  - Condition name:
  - Other field:
- [ ] Other (treat as last resort)
  - Details:

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

100% in normal operation. The proposed API exposes in read only mode kubelet internal data, critical for functioning of the kubelet.
This data has to be available 100% of the time for the proper functioning of the kubelet, thus is expected to be available 100% of time.
The only possible error source is the API calls being throttled by the rate-limiting introduced with the GA graduation of the parent KEP 606.

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

- [X] Metrics
  - Metric name:  `pod_resources_endpoint_requests_total`, `pod_resources_endpoint_requests_list` and `pod_resources_endpoint_requests_get`.
  - Components exposing the metric: kubelet

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

As part of this feature enhancement, per-API-endpoint resources metrics are being added; to observe this feature the `pod_resources_endpoint_requests_get` and `pod_resources_endpoint_requests_list` metric should be used. We will add `pod_resources_endpoint_errors_get` error counter.

### Dependencies

The container runtime must support CDI.

###### Does this feature depend on any specific services running in the cluster?

A third-party resource driver is required for allocating resources.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No. Enabling this feature does not change the number of API objects returned. But it may increase the size of each object whenever there are Dynamic Resources to report where each ContainerResources now has an extra dynamic_resources field.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No. Feature is out of existing any paths in kubelet.

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?
Negligible amount of CPU and memory. Because the API is purely read-only and piggy-backs on the kubelet’s existing cache and checkpointing machinery, exposing Dynamic Resources incurs only similar minimal serialization and storage as CPUManager and DeviceManager—so any extra CPU, memory, disk, or I/O impact is negligible.

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

No, because the endpoint queries existing data structures inside the kubelet.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

No impact, the feature is node-local.

###### What are other known failure modes?

feature gate disabled: The API will always return a well-known error. In normal operation, the API is expected to never return an error and always return a valid response, because it utilizes internal kubelet data which is always available.
Bugs may cause the API to return unexpected errors, or to return inconsistent data.
Consumers of the API should treat unexpected errors as bugs of this API.

###### What steps should be taken if SLOs are not being met to determine the problem?

Check the error code to learn if the consumer of the API is being throttle by rate limiting introduced in the parent KEP 606.
Check the kubelet logs to learn about resource allocation errors.

## Implementation History

- 2023-01-12: KEP created

- 2024-09-10: KEP Updated to reflect the current state of the implementation.

- 2025-05-27: Beta version of the KEP.

## Drawbacks

N/A

## Alternatives

N/A
