---
title: short-circuiting backoff

authors:
- @mshitrit

reviewers:
- @beekhof
- @n1r1
- @slintes
- @rgolan

approvers:
- @JoelSpeed
- @michaelgugino
- @enxebre

creation-date: 2021-03-01

last-updated: 2021-03-10

status: implementable

see-also:
- https://github.com/openshift/enhancements/blob/master/enhancements/machine-api/machine-health-checking.md

---

# Support backoff when short-circuiting

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [openshift-docs](https://github.com/openshift/openshift-docs/)

## Summary

By using `MachineHealthChecks` a cluster admin can configure automatic remediation of unhealthy machines and nodes.
The machine healthcheck controller's remediation strategy is deleting the machine, and letting the provider create a new one.

Currently, machines have a delay for a fixed amount of time before a health check fails and remediation is triggered.
These machines match one of three use cases:
1. Any Machine CRs with no Node and `.Status.Phase == Failed` - are remediated after a `failedNodeStartupTimeout` interval (48h by default). (Existing downstream PR only)
2. Any other Machine CRs with no Node - are remediated after a `nodeStartupTimeout` interval.
3. Any Machine CRs with a Node and deemed to be unhealthy by MHC - are remediated after `MHC.Spec.UnhealthyConditions.Timeout`.

Use-case #1 Here is the reasoning behind the currently implemented logic:
We allow a longer recovery interval for nodes that fail during the provisioning phase, instead of entering a hot-loop, or an error state and giving up completely. This avoids wasting cluster resources performing reprovisioning cycles that require human interaction (eg. updating api credentials, or replacing broken hardware) in order to complete.
However the current implementation can still be improved, since a constant predefined time might be too short (thus causing remediation while an admin is working on the node) or too long (which can prevent the cluster from discovering that the node has been fixed and can become healthy again).  Expecting the admin to get the value exactly right is unrealistic.


Use-case #2 covers the detection of nodes that appear to be provisioned correctly by the provider, but never join the cluster. By default we wait 10min in this situation in order to give the node time to boot and join the cluster.  The time needed to do this varies  in different environments (bare metal, cloud etc...) and the consequence of a value that is too small is constantly triggering remediations (which ultimately are wasting resources and hindering the cluster integrity).
Another side effect might be unwanted triggering of the `maxUnhealthy` benchmark, which will prevent ALL remediations.


We'd like to suggest the following improvement:
Instead of using fixed remediation times (`failedNodeStartupTimeout` and `nodeStartupTimeout`), we'd like the time to be determined dynamically - therefore avoiding the pathological behaviour of having too short a value configured (or by default) while preserving the ability for the cluster to complete recovery once the underlying issue has been addressed.

## Motivation

- Prevent unnecessary and unbounded recurring remediations.
- Creating a large enough time frame for an admin to fix certain issues by delaying the remediation time (admins don’t appreciate nodes being rebooted under them)
- Reduce the occurrences of remediation short-circuiting events.
- avoid delaying remediation for other machines in a set when one has a persistent failure (eg. a bad disk on bare metal) (implication of the #1 case pr).

### Goals

- Create a dynamic mechanism which determines the optimal remediation time for machines that are not in failed state (but did not pass the health check).
- Create a dynamic mechanism which determines the optimal remediation time for machines that are in a failed state (and are without a node).

### Non-Goals

- Adaptive recovery intervals for Masters and other nodes not associated with a MachineSet

## Proposal

We propose adding annotations to the MachineSet CR to support a dynamic machineSet based mechanism which will determine the optimal remediation time.

Each machineSet will manage 2 remediation timeframes:
- A - `AttemptsCountNonFailedStateMachines` - proportionate to `nodeStartupTimeout`
- B - `AttemptsCountFailedStateMachines` - proportionate to `failedNodeStartupTimeout`

The number of remediation attempts (within a machine set) will be tracked on the machineSet annotation (separately for each category).
An increase in the remediation attempts will cause an increase in delay time (before remediation) which will be proportionate to either `nodeStartupTimeout` or `failedNodeStartupTimeout`.
If all the machines (belonging to one of the categories within a machine set) will reach a healthy state - the relevant marker will be reset.


### User Stories

#### Story 1

As an admin of a hardware based cluster, I would like to prevent unnecessary remediations which waste resources and impair the cluster health.

#### Story 2

As an admin of a hardware based cluster, I would like faster remediations in order to restore cluster health more rapidly.

### Implementation Details/Notes/Constraints

#### MHC struct enhancement

#### Example CRs

MachineSet:
```yaml
apiVersion: machine.openshift.io/v1beta1
kind: MachineSet
metadata:
 annotations:
   machine.openshift.io/attempts-count-non-failed-state-machines: "0"
   machine.openshift.io/attempts-count-failed-state-machines: "0"
   ...
 ...
spec:
 replicas: 2
 selector:
   ...
 template:
   metadata:
     labels:
       ...
   spec:
     providerSpec:
       ...
     versions:
       ...
```

### Risks and Mitigations

No known risks

## Design Details

### Open Questions

### Test Plan

The existing remediation tests will be reviewed / adapted / extended as needed.

### Graduation Criteria

TBD

#### Examples

TBD

##### Dev Preview -> Tech Preview

TBD

##### Tech Preview -> GA

TBD

##### Removing a deprecated feature

### Upgrade / Downgrade Strategy

### Version Skew Strategy

## Implementation History

- [x] 03/10/2021: Opened enhancement PR

## Drawbacks

no known drawbacks

## Alternatives


## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.


