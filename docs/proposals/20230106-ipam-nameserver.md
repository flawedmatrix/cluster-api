---
title: IPAddress Supports Nameserver Field
authors:
  - "@flawedmatrix"
  - "@tylerschultz"
  - "@christianang"
  - "@adobley"
reviewers:
creation-date: 2023-01-06
last-updates: 2023-01-10
status: provisional
---

# IPAddress Supports Nameserver Field

## Table of Contents

- [Title](#title)
  - [Table of Contents](#table-of-contents)
  - [Glossary](#glossary)
  - [Summary](#summary)
  - [Motivation](#motivation)
    - [Goals](#goals)
    - [Non-Goals/Future Work](#non-goalsfuture-work)
  - [Proposal](#proposal)
    - [User Stories](#user-stories)
      - [Story 1](#story-1)
      - [Story 2](#story-2)
    - [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
    - [Security Model](#security-model)
    - [Risks and Mitigations](#risks-and-mitigations)
  - [Alternatives](#alternatives)
  - [Upgrade Strategy](#upgrade-strategy)
  - [Additional Details](#additional-details)
    - [Test Plan [optional]](#test-plan-optional)
    - [Graduation Criteria [optional]](#graduation-criteria-optional)
    - [Version Skew Strategy [optional]](#version-skew-strategy-optional)
  - [Implementation History](#implementation-history)

## Glossary

Refer to the [Cluster API Book Glossary](https://cluster-api.sigs.k8s.io/reference/glossary.html).

### IPAM Provider

A controller that watches `IPAddressClaims` and fulfils them with `IPAddresses`
that it allocates from an IP Pool. It comes with it's own IP Pool custom
resource definition to provide parameters for allocating addresses. Providers
can work in-cluster relying only on custom resources, or integrate with
external systems like Netbox or Infoblox.

### IPAddress

An `IPAddress` is a custom resource that gets created by an IPAM provider to
fulfil an `IPAddressClaim`, which then gets consumed by the infrastructure
provider that created the claim.

### IP Pool

An IP Pool is a custom resource that is provided by a IPAM provider, which
holds configuration for a single pool of IP addresses that can be allocated
from using an `IPAddressClaim`.

## Summary

This proposal adds an additional optional field to the `IPAddress` object:
Nameservers. The IPAM Provider is responsible for setting this fields based on
IP Pool data or provider configuration. This would then allow infrastructure
providers to configure the nodes with DNS.


## Motivation

When using DHCP nameservers are often returned in the DHCP response from
the DHCP server. When deploying with IPAM such information must be provided to
each cluster by the user.

By adding Nameservers to the `IPAddress` object, the IPAM Provider can replace the
role of the DHCP server in supplying these nameservers without requiring users
to manually provide these nameservers.

### Goals

- Extend the API contract to
  - add a `nameservers` field to `IPAddress`
  - allow infrastructure providers to configure machines with these nameservers

### Non-Goals/Future Work

- This proposal does not mandate that IPAM providers must supply nameservers.
- Adding additional fields supplied by other DHCP options such as NTP, MTU,
  etc. is not in the scope of this proposal

## Proposal

The proposal is to add a new `nameservers` field to the `IPAddress` type. The
nameservers fields will accept addresses and search domains similar to what
[netplan](https://netplan.io/reference#common-properties-for-all-device-types)
or
[cloud-init](https://cloudinit.readthedocs.io/en/latest/reference/network-config-format-v2.html#nameservers-mapping)
do already.

Existing IPAM Providers can choose to configure and supply nameservers to the
IPAddress resource when allocated, but do not have to.

Existing Infrastructure Providers that support the IPAM API, currently only
CAPV, should use nameservers if supplied by the IPAddress resource.

### User Stories

#### Story 1

As a cluster operator I want to configure my IPAM Provider with nameservers
that will automatically be used by any machine that claims ip
addresses from my pool.

### Implementation Details/Notes/Constraints

The following is a strawdog proposal of what is being added to the API:

```go
// IPAddressSpec is the desired state of an IPAddress.
type IPAddressSpec struct {
	// ClaimRef is a reference to the claim this IPAddress was created for.
	ClaimRef corev1.LocalObjectReference `json:"claimRef"`

	// PoolRef is a reference to the pool that this IPAddress was created from.
	PoolRef corev1.TypedLocalObjectReference `json:"poolRef"`

	// Address is the IP address.
	Address string `json:"address"`

	// Prefix is the prefix of the address.
	Prefix int `json:"prefix"`

	// Gateway is the network gateway of the network the address is from.
	Gateway string `json:"gateway"`

	// Nameservers is the dns servers and search domains to be configured on the machine.
	Nameservers IPAddressNameservers `json:"nameservers,omitempty"`
}

// IPAddressNameservers is the dns servers and search domains.
type IPAddressNameservers struct {
	// Search are the set of search domains to be configured on the machine.
	Search []string `json:"search"`

	// Addresses are the set of dns servers that should be used for dns queries.
	Addresses []string `json:"addresses"`
}
```

#### Ordering and Precedence

When there are multiple `IPAddresses` each with different `Nameservers` fields, the nameservers shall
be ordered and merged according to the order that `IPAddresses` are assigned to the machine.

Using CAPV as an example, if a machine specifies IP pool references like so:

```yaml
apiVersion: infrastructure.cluster.x-k8s.io/v1alpha3
kind: VSphereMachineTemplate
metadata:
  name: example
  namespace: vsphere-site1
spec:
  template:
    spec:
      cloneMode: FullClone
      numCPUs: 8
      memoryMiB: 8192
      diskGiB: 45
      network:
        devices:
        # Device 0 has one address from testpool-1
        - dhcp4: false
          addressesFromPools:
          - group: ipam.cluster.x-k8s.io/v1alpha1
            kind: IPPool
            name: testpool-1
        # Device 1 has one address from testpool-2
        - dhcp4: false
          addressesFromPools:
          - group: ipam.cluster.x-k8s.io/v1alpha1
            kind: IPPool
            name: testpool-2
        # Device 2 has two addresses from testpool-3 and testpool-4
        - dhcp4: false
          addressesFromPools:
          - group: ipam.cluster.x-k8s.io/v1alpha1
            kind: IPPool
            name: testpool-3
          addressesFromPools:
          - group: ipam.cluster.x-k8s.io/v1alpha1
            kind: IPPool
            name: testpool-4
```

and supposing that `IPAddressClaims` for each of these three pools were generated and yielded
`IPAddresses` in the following order:
```yaml
---
# The first IPAddress has no nameservers provided
apiVersion: ipam.cluster.x-k8s.io/v1alpha1
kind: IPAddress
metadata:
  name: example-0-0
  namespace: vsphere-site1
spec:
  address: "10.0.0.10"
  poolRef:
    apiGroup: ipam.cluster.x-k8s.io
    kind: IPPool
    name: test-pool-1

---
apiVersion: ipam.cluster.x-k8s.io/v1alpha1
kind: IPAddress
metadata:
  name: example-1-0
  namespace: vsphere-site1
spec:
  address: "10.0.1.10"
  poolRef:
    apiGroup: ipam.cluster.x-k8s.io
    kind: IPPool
    name: test-pool-2
  nameservers:
    search: ["corp.domain"]
    addresses: ["10.0.0.1"]

---
apiVersion: ipam.cluster.x-k8s.io/v1alpha1
kind: IPAddress
metadata:
  name: example-2-0
  namespace: vsphere-site1
spec:
  address: "10.0.2.10"
  poolRef:
    apiGroup: ipam.cluster.x-k8s.io
    kind: IPPool
    name: test-pool-3
  nameservers:
    search: ["alternate.domain"]
    addresses: ["10.100.0.1"]

---
apiVersion: ipam.cluster.x-k8s.io/v1alpha1
kind: IPAddress
metadata:
  name: example-2-1
  namespace: vsphere-site1
spec:
  address: "10.0.3.10"
  poolRef:
    apiGroup: ipam.cluster.x-k8s.io
    kind: IPPool
    name: test-pool-4
  nameservers:
    search: ["yet.another.domain"]
    addresses: ["10.200.0.1"]

```

then the resulting `nameservers` configuration on the machine will be the following
```yaml
nameservers:
  search: ["corp.domain", "alternate.domain", "yet.another.domain"]
  addresses: ["10.0.0.1", "10.100.0.1", "10.200.0.1"]

```

Additionally, any nameserver settings on the Machine should take precedence
over nameservers provided by `IPAddresses`.

The behavior when `IPAddress.spec.nameservers` are provided and DHCP is enabled
and providing nameservers is undefined.

### Security Model

Since this is an existing field on `IPAddress`, it follows the same security
model as the [initial IPAM API proposal](./20220125-ipam-integration.md).

### Risks and Mitigations

This shouldn't add any risk over what can already be done via dhcp or setting
nameservers directly on the machine. The above ordering and precedence rules
should also mitigate any potential issues with "breaking" changes since
explicitly setting the nameservers on the machine takes precedence over
nameservers from the IPAM API.

## Alternatives

The alternative to this is to have the user continue configuring nameservers
directly on the machine, while this may be fine consuming nameservers via the
IPAM API reduces manual steps a user may have to take for each cluster.

## Implementation History

- [x] 01/10/2023: Compile a proposal following the CAEP template
- [ ] MM/DD/YYYY: Proposed idea at CAPI office hours
- [ ] MM/DD/YYYY: Open proposal PR

<!-- Links -->
[community meeting]: https://docs.google.com/document/d/1ushaVqAKYnZ2VN_aa3GyKlS4kEd6bSug13xaXOakAQI/edit#heading=h.pxsq37pzkbdq
