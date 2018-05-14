---
kep-number: 0008
title: Sidecar Containers
authors:
  - "@joseph-irving"
owning-sig: sig-apps
participating-sigs:
  - sig-apps
  - sig-node
reviewers:
  - TBD
approvers:
  - TBD
editor: TBD
creation-date: 2018-05-14
last-updated: 2018-05-14
status: provisional
---

# Sidecar Containers

## Table of Contents

* [Table of Contents](#table-of-contents)
* [Summary](#summary)
* [Motivation](#motivation)
    * [Goals](#goals)
    * [Non-Goals](#non-goals)
* [Proposal](#proposal)
    * [Implementation Details/Notes/Constraints](#implementation-detailsnotesconstraints-optional)
    * [Risks and Mitigations](#risks-and-mitigations)
* [Graduation Criteria](#graduation-criteria)
* [Implementation History](#implementation-history)
* [Alternatives](#alternatives-optional)

## Summary

To solve the issue of sidecars not being well supported we can create a new type of container called a sidecar which are handled like normal containers but with the caveat that running sidecar containers will always be terminated when all non-sidecar containers have terminated. This will solve the problem of Jobs not completing/failing when a sidecar container never exits due to it having no awareness of the 'primary' container.

## Motivation

SideCar containers have always been used in some ways but just not formally identified as such, with the introduction of Jobs and Cronjobs it became clear that they're not handled very well. If you have a Job with two containers one of which is actually doing the main processing of the job and the other is just facilitating it, you encounter a problem when the main process finishes; your sidecar container will carry on running so the job will never finish.

The only way around this problem is to manage the sidecar container's lifecycle manually and arrange for it to exit when the main container exits. This is typically achieved by building an ad-hoc signalling mechanism to communicate completion status between containers. Common implementations use a shared scratch volume mounted into all pods, where lifecycle status can be communicated by creating and watching for the presence of files. This pattern has several disadvantages:

* Repetitive lifecycle logic must be rewritten in each instance a sidecar is deployed.
* Third-party containers typically require a wrapper to add this behaviour, normally provided via an entrypoint wrapper script implemented in the k8s container spec. This adds undesirable overhead and introduces repetition between the k8s and upstream container image specs.
* The wrapping typically requires the presence of a shell in the container image, so this pattern does not work for minimal containers which ship without a toolchain.

### Goals

Solve issue [25908](https://github.com/kubernetes/kubernetes/issues/25908) so that sidecars can be easily handled without needing modifications to the container itself

### Non-Goals

Changing the way jobs are handled

## Proposal

Create a way to define containers as sidecars, this will be an additional field to the Container Spec: `sidecar: true`.
e.g:
```yaml
apiVersion: v1
kind: Pod
metadata:
  name: myapp-pod
  labels:
    app: myapp
spec:
  containers:
  - name: myapp
    image: myapp
    command: ['do something']
  - name: sidecar
    image: sidecar-image
    sidecar: true
    command: ["do something to help my app"]

```
These sidecar containers will be handled the same way as a non-sidecar container in almost all respects apart from the fact that when all non-sidecar containers have permanently terminated all running sidecar containers will also be terminated with a reason of `Completed`. The sidecars will be terminated by receiving a SIGTERM and will obey the `terminationGracePeriodSeconds` defined on the pod before being sent a SIGKILL. Sidecar containers will be terminated in the same way as non-sidecars when a pod is deleted.

For example, in a job you can have as many sidecars as you like and all the running sidecars will be terminated when all the non-sidecar containers have terminated, causing your job to finish.
If all the non-sidecars `Completed` the pod status will be `Succeeded`, if any of the non-sidecars terminated with `Error` the pod status will be `Failed` and if any of the sidecars terminated with `Error` before the non-sidecars terminated, the pod status will be `Failed`.

A possible extension would be to allow this to work with init containers, effectively having all init sidecars start at the beginning of the init phase and then being terminated when all non-sidecar init containers have completed.

### Implementation Details/Notes/Constraints

As this is a change to the Container spec we will be using feature gating, you will be required to explicitly enable this feature on the api server as recommended [here](https://github.com/kubernetes/community/blob/master/contributors/devel/api_changes.md#adding-unstable-features-to-stable-versions).

### Risks and Mitigations

You could set all containers to be `sidecar: true`, this seems wrong, so maybe the api should do a validation check that at least one container is not a sidecar.

Init containers would be able to have `sidecar: true` applied to them as it's an additional field to the container spec, this doesn't currently make sense as init containers are ran sequentially. We could get around this by having the api throw a validation error if you try to use this field on an init container or just ignore the field.

## Graduation Criteria

//TODO

## Implementation History

- 14th May 2018: Proposal Submitted


## Alternatives

One alternative would be to have a new field in the Pod Spec of `sidecarContainers:` where you could define a list of sidecar containers, however this would require a lot more work in terms of updating tooling to support this.
Another alternative would be to change the Job Spec to have a `primaryContainer` field to tell it which containers are important. However I feel this is perhaps too specific to job when this Sidecar concept could be useful in other scenarios.
