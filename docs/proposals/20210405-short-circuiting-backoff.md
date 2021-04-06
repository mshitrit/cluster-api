---
title: short-circuiting backoff

authors:
- "@mshitrit"

reviewers:
- "@detiber"
- "@vincepri"
- "@ncdc"
- "@timothysc"

creation-date: 2021-04-05

last-updated: 2021-04-05

status: implementable

see-also:
- https://github.com/kubernetes-sigs/cluster-api/blob/master/docs/proposals/20191030-machine-health-checking.md

---
# Title
- Support backoff when short-circuiting

Table of Contents
=================

* [Title](#title)
* [Table of Contents](#table-of-contents)
    * [Glossary](#glossary)
    * [Summary](#summary)
    * [Motivation](#motivation)
        * [Goals](#goals)
        * [Non-Goals](#non-goals)
    * [Proposal](#proposal)
        * [User Stories](#user-stories)
            * [Story 1](#story-1)
            * [Story 2](#story-2)
        * [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints)
            * [MHC struct enhancement](#mhc-struct-enhancement)
            * [Example CRs](#example-crs)
        * [Risks and Mitigations](#risks-and-mitigations)
    * [Design Details](#design-details)
        * [Open Questions](#open-questions)
        * [Test Plan](#test-plan)
        * [Graduation Criteria](#graduation-criteria)
            * [Examples](#examples)
                * [Dev Preview -&gt; Tech Preview](#dev-preview---tech-preview)
                * [Tech Preview -&gt; GA](#tech-preview---ga)
                * [Removing a deprecated feature](#removing-a-deprecated-feature)
        * [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
        * [Version Skew Strategy](#version-skew-strategy)
    * [Implementation History](#implementation-history)
    * [Drawbacks](#drawbacks)
    * [Alternatives](#alternatives)
    * [Infrastructure Needed [optional]](#infrastructure-needed-optional)

Created by [gh-md-toc](https://github.com/ekalinin/github-markdown-toc)

## Glossary

Refer to the [Cluster API Book Glossary](https://cluster-api.sigs.k8s.io/reference/glossary.html).

## Summary

By using `MachineHealthChecks` a cluster admin can configure automatic remediation of unhealthy machines and nodes.
The machine healthcheck controller's remediation strategy is deleting the machine, and letting the provider create a new one.

Currently, machines have a delay for a fixed amount of time before a health check fails and remediation is triggered.
These machines match one of three use cases:
1. Any Machine CRs with no Node and `.Status.Phase == Failed` - are remediated immediately.
2. Any other Machine CRs with no Node - are remediated after a `nodeStartupTimeout` interval.
3. Any Machine CRs with a Node and deemed to be unhealthy by MHC - are remediated after `MHC.Spec.UnhealthyConditions.Timeout`.

Use-case #1 Any Machine that enters the `Failed` state is remediated immediately, without waiting, by the MHC.
When this occurs, if the error which caused the failure is persistent (spot price too low, configuration error), replacement Machines will also be `Failed`.
As replacement machines start and fail, MHC causes a hot loop of Machine being deleted and recreated.
This hot looping makes it difficult for users to find out why their Machines are failing.
Another side effect of machines constantly failing, is the risk of hitting the benchmark of machine failures percentage - thus triggering the "short-circuit" mechanism which will prevent all remediations.

Use-case #2 covers the detection of nodes that appear to be provisioned correctly by the provider, but never join the cluster. By default, we wait 10min in this situation in order to give the node time to boot and join the cluster.  The time needed to do this varies  in different environments (bare metal, cloud etc...), and the consequence of a value that is too small is constantly triggering remediations (which ultimately are wasting resources and hindering the cluster integrity).
Another side effect might be unwanted triggering of the `maxUnhealthy` benchmark, which will prevent ALL remediations.


We'd like to suggest the following improvement:
Instead of using fixed remediation times (`nodeStartupTimeout`), we'd like the time to be determined dynamically - therefore avoiding the pathological behaviour of having too short a value configured (or by default) while preserving the ability for the cluster to complete recovery once the underlying issue has been addressed.

## Motivation

- Prevent unnecessary and unbounded recurring remediations.
- Creating a large enough time frame for an admin to fix certain issues by delaying the remediation time (admins don’t appreciate nodes being rebooted under them)
- Reduce the occurrences of remediation short-circuiting events.
- avoid delaying remediation for other machines in a set when one has a persistent failure (e.g. a bad disk on bare metal) (implication of the #1 case pr).

### Goals

- Create a dynamic mechanism which determines the optimal remediation time for machines that are not in failed state (but did not pass the health check).
- Create a dynamic mechanism which determines the optimal remediation time for machines that are in a failed state (and are without a node).

### Non-Goals

- Adaptive recovery intervals for Masters and other nodes not associated with a MachineSet

## Proposal

We propose modifying the MachineHealthCheck CRD with a new value `failedNodeStartupTimeout`, in order to support a failed node startup timeout. `failedNodeStartupTimeout` defines a baseline for the period after which a `Failed` machine will be remediated by the MachineHealthCheck.

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

```go
    type MachineHealthCheckSpec struct {
        ...
    
        // +optional
        FailedNodeStartupTimeout metav1.Duration `json:"failedNodeStartupTimeout,omitempty"`
    }
```

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

MachineHealthCheck:
```yaml
    kind: MachineHealthCheck
    apiVersion: machine.openshift.io/v1beta1
    metadata:
      name: REMEDIATION_GROUP
      namespace: NAMESPACE_OF_UNHEALTHY_MACHINE
    spec:
      selector:
        matchLabels: 
          ...
      failedNodeStartupTimeout: 48h
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

- [x] 04/05/2021: Opened enhancement PR

## Drawbacks

no known drawbacks

## Alternatives


## Infrastructure Needed [optional]

Use this section if you need things from the project. Examples include a new
subproject, repos requested, github details, and/or testing infrastructure.

Listing these here allows the community to get the process for these resources
started right away.


