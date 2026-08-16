---
title: Scaling cert-manager
description: |
    Learn how to optimize cert-manager for your cluster.
---

Learn how to optimize cert-manager for your cluster.

## Overview

The defaults in the Helm chart and YAML manifests are intended for general use.
You may want to modify the configuration to suit the size and usage of your Kubernetes cluster.

## Set appropriate memory requests and limits

**When Certificate resources are the dominant use-case**,
such as when workloads need to mount the TLS Secret or when gateway-shim is used,
the memory consumption of the cert-manager controller will be roughly
proportional to the total size of those Secret resources that contain the TLS
key pairs.
Why? Because the cert-manager controller caches the entire content of these Secret resources in memory.
If large TLS keys are used (e.g. RSA 4096) the memory use will be higher than if smaller TLS keys are used (e.g. ECDSA).

The other Secrets in the cluster, such as those used for Helm chart configurations or for other workloads,
will not significantly increase the memory consumption, because cert-manager will only cache the metadata of these Secrets.

**When `CertificateRequest` resources are the dominant use-case**,
such as with csi-driver or with istio-csr,
the memory consumption of the cert-manager controller will be much lower,
because there will be fewer TLS Secrets and fewer resources to be cached.

> 📖️ Read [What Everyone Should Know About Kubernetes Memory Limits](https://home.robusta.dev/blog/kubernetes-memory-limit),
> to learn how to right-size the memory requests.

## Client-side rate limiting of Kubernetes API requests

Like all applications built on `client-go`, cert-manager ships with a client-side rate limiter:
a token bucket which delays cert-manager's own requests to the Kubernetes API server
once they exceed [20 queries per second, with bursts of up to 50](https://github.com/cert-manager/cert-manager/blob/v1.21.1/internal/apis/config/controller/v1alpha1/defaults.go#L58-L59).
It dates from the era before the Kubernetes API server could protect itself from its clients,
when well-behaved clients were expected to throttle themselves.

That era ended with [API Priority and Fairness](https://kubernetes.io/docs/concepts/cluster-administration/flow-control/)
(enabled by default since Kubernetes 1.20, GA since 1.29),
which protects the API server in a fundamentally different way from a rate limiter:

- It limits how many requests each traffic class may have **in flight at once** — concurrency,
  which is what actually determines API server load — rather than how many arrive per second.
- Excess requests are **queued at the server** and dispatched as capacity frees up,
  fairly interleaved so that a burst from one busy client
  (such as cert-manager re-syncing every Certificate) cannot starve other clients.
- Only when the queues overflow are requests **rejected with HTTP 429** and a `Retry-After` header,
  which `client-go` responds to by backing off.

There is no QPS or burst setting to tune in API Priority and Fairness,
because it does not count requests per second at all: bursts are absorbed by its queues.

On a cluster protected this way, a client-side rate limiter protects nothing;
it only slows cert-manager down.
During a mass re-sync of tens of thousands of Certificate resources,
cert-manager trickles requests at 20 per second to an API server that is nowhere near capacity,
and the only symptom is "client-side throttling" messages in the cert-manager logs.
Worse, the limiter is shared by all of cert-manager's internal clients,
including the one that renews its leader election lease,
so sustained throttling can delay lease renewal.
The Kubernetes ecosystem has reached the same conclusion:
[controller-runtime disables the client-side rate limiter by default since v0.21](https://github.com/kubernetes-sigs/controller-runtime/pull/3119),
on the advice of SIG API Machinery,
and [Flux](https://github.com/fluxcd/pkg/issues/269) disables it when it detects API Priority and Fairness.

**cert-manager `>= v1.21.0` needs no configuration.**
At startup, the controller [probes the API server](https://github.com/cert-manager/cert-manager/blob/v1.21.1/pkg/controller/context.go#L514-L555)
for the response header which indicates that API Priority and Fairness is enabled,
and if it is, [disables the client-side rate limiter](https://github.com/cert-manager/cert-manager/blob/v1.21.1/pkg/controller/context.go#L303-L306).
If the probe fails, cert-manager falls back to client-side rate limiting.

> ⚠️ In cert-manager `v1.21`, the `kubernetesAPIQPS` and `kubernetesAPIBurst` configuration options are ignored
> when API Priority and Fairness is detected, even if you set them explicitly.
> This matters if you *want* to cap cert-manager's request rate;
> for example, on a managed control plane which meters or bills API requests.
> Read [`cert-manager#9158`](https://github.com/cert-manager/cert-manager/issues/9158) for discussion of this limitation.

**cert-manager `< v1.21.0` always applies the client-side rate limiter.**
You can raise its thresholds high enough that they are never reached, using the following Helm values:

```yaml
# helm-values.yaml
config:
  kubernetesAPIQPS: 10000
  kubernetesAPIBurst: 10000
```

> 🔗 Read [`cert-manager#8757`](https://github.com/cert-manager/cert-manager/pull/8757);
> the pull request which introduced automatic detection of API Priority and Fairness,
> fixing [`cert-manager#6890`: Allow client-side rate-limiting to be disabled](https://github.com/cert-manager/cert-manager/issues/6890).
>
> 🔗 Read [`kubernetes#111880`: Disable client-side rate-limiting when AP&F is enabled](https://github.com/kubernetes/kubernetes/issues/111880);
> a proposal that the `kubernetes.io/client-go` module should do this automatically.
>
> 📖 Read [API documentation for ControllerConfiguration](../reference/api-docs.md#controller.config.cert-manager.io/v1alpha1.ControllerConfiguration) for a description of the `kubernetesAPIQPS` and `kubernetesAPIBurst` configuration options.

## Restrict the use of large RSA keys

Certificates with large RSA keys cause cert-manager to use more CPU resources.
When there are insufficient CPU resources, the reconcile queue length grows,
which delays the reconciliation of all Certificates.
A user who has permission to create a large number of RSA 4096 certificates,
might accidentally or maliciously cause a denial of service for other users on the cluster.

> 📖 Learn [how to enforce an Approval Policy](../policy/approval/README.md), to prevent the use of large RSA keys.
>
> 📖 Learn [how to set Certificate defaults automatically](../tutorials/certificate-defaults/README.md), using tools like Kyverno.


## Set `revisionHistoryLimit: 1` on all Certificate resources

> ℹ️ Not needed with cert-manager `>= v1.18.0`, because the default value was changed to `1`.

By default, cert-manager will keep all the `CertificateRequest` resources that **it** creates
([`revisionHistoryLimit`](../reference/api-docs.md#cert-manager.io/v1.CertificateSpec)):

> The maximum number of `CertificateRequest` revisions that are maintained in
> the Certificate's history. Each revision represents a single
> `CertificateRequest` created by this Certificate, either when it was
> created, renewed, or Spec was changed. Revisions will be removed by oldest
> first if the number of revisions exceeds this number.
>  If set, `revisionHistoryLimit` must be a value of `1` or greater. If unset
> (`nil`), revisions will not be garbage collected. Default value is `nil`.

On a busy cluster these will eventually overwhelm your Kubernetes API server;
because of the memory and CPU required to cache them all and the storage required to save them.

Use a tool like Kyverno to override the `Certificate.spec.revisionHistoryLimit` for all namespaces.

> 📖 Adapt [the Kyverno policies in the tutorial: how to set Certificate defaults automatically](../tutorials/certificate-defaults/README.md),
> to override rather than default the `revisionHistoryLimit` field.
>
> 📖 Learn [how to set `revisionHistoryLimit` when using Annotated Ingress resources](../usage/ingress.md#supported-annotations).
>
> 🔗 Read [`cert-manager#3958`: Sane defaults for Certificate revision history limit](https://github.com/cert-manager/cert-manager/issues/3958);
> a proposal to change the default `revisionHistoryLimit`, which will obviate this particular recommendation.

## Enable Server-Side Apply

By default, cert-manager [uses Update requests](https://kubernetes.io/docs/reference/using-api/api-concepts/#update-mechanism-update)
to create and modify resources like `CertificateRequest` and `Secret`,
but on a busy cluster there will be frequent conflicts as the control loops in cert-manager each try to update the status of various resources.

You will see errors, like this one, in the logs:

> `I0419 14:11:51.325377       1 controller.go:162] "re-queuing item due to optimistic locking on resource" logger="cert-manager.certificates-trigger" key="team-864-p6ts6/app-7" error="Operation cannot be fulfilled on certificates.cert-manager.io \"app-7\": the object has been modified; please apply your changes to the latest version and try again"`

This error is relatively harmless because the update attempt is retried,
but it slows down the reconciliation because each error triggers an exponential back off mechanism,
which causes increasing delays between retries.

The solution is to turn on the [Server-Side Apply Feature](../installation/configuring-components.md#feature-gates),
which causes cert-manager to use [HTTP PATCH using Server-Side Apply](https://kubernetes.io/docs/reference/using-api/api-concepts/#update-mechanism-server-side-apply) when ever it needs to modify an API resource.
This avoids all conflicts because each cert-manager controller sets only the fields that it owns.

You can enable the server-side apply feature gate with the following Helm chart values:

```yaml
# helm-values.yaml
config:
  featureGates:
    ServerSideApply: true
```

> 📖 Read [Using Server-Side Apply in a controller](https://kubernetes.io/docs/reference/using-api/server-side-apply/#using-server-side-apply-in-a-controller),
> to learn about the advantages of server-side apply for software like cert-manager.
