# Nephio Installation Overview

## Table of Contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Required Dependencies](#common-required-dependencies)
- [Optional Dependencies](#other-dependencies)
- [Required Components](#required-components)
- [Optional Components](#optional-components)

## Introduction

There are many ways to assemble a Nephio installation. This Installation Guide
describes the common pieces across environments, and describes the choices that
need to be made to create a "Bring Your Own Cluster" Nephio installation.
Because there are so many combinations, a comprehensive guide is not practical.
Instead, several guides showing opinionated installations are available.

## Prerequisites

Regardless of the specific choices you make, the you will need the following
prerequisites. This is in addition to any prerequisites that are specific to
your environment and choices.
 - a cluster with Internet access (any non-EOL Kubernetes version is fine)
 - `kubectl` installed on your workstation
 - `kpt` installed on your workstation (version v1.0.0-beta.43 or later)
 - Sudo-less `docker`, `podman`, or `nerdctl`. If using `podman` or `nerdctl`,
   you must set the
   [`KPT_FN_RUNTIME`](https://kpt.dev/reference/cli/fn/render/?id=environment-variables)
   environment variable.
 - Default `kubectl` context pointing to the cluster
 - Cluster administrator privileges (in particular you will need to be able to
   create namespaces and other cluster-scoped resources).

## Common Required Dependencies

First we will install some required dependencies. Some of these, like the
resource-backend, will move out of the "required" category in later releases.

### Network Config Operator

This component is a controller for applying configuration to routers and
switches.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/network-config@v1.0.1
kpt fn render network-config
kpt live init network-config
kpt live apply network-config --reconcile-timeout=15m --output=table
```

### Resource Backend

The resource backend provides IP and VLAN allocation. If you have an alternative
solution for this, you may not use this directly. But the CRDs delivered as part
of this package will be needed to integrate with any similar solution.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/resource-backend@v1.0.1
kpt fn render resource-backend
kpt live init resource-backend
kpt live apply resource-backend --reconcile-timeout=15m --output=table
```

## Other Dependencies

### Git Providers

Nephio can support multiple Git providers for the repositories that contain
packages. In R1, only Gitea repositories can be provisioned directly by Nephio;
other Git providers will require manual provisioning of new repositories. But
most Git providers can be supported (via standard Git protocols) as repositories
for packages for read and write. It is also perfectly fine to use multiple
providers; in the R1 demo environment, GitHub is used for upstream external
repositories while Gitea is used for the workload cluster repositories.

A non-exhaustive list of options:

| Provider                                                        | Workloads | Provisioning  |
| --------------------------------------------------------------- | --------- | ------------- |
| [GitHub](https://github.com)                                    | Yes       | No            |
| [Gitea](https://about.gitea.com/)                               | Yes       | Yes           |
| [GitLab](https://about.gitlab.com/)                             | Yes       | No            |
| [Google CSR](https://cloud.google.com/source-repositories/docs) | Yes       | Yes, with KCC |

See the [Porch user
guide](https://kpt.dev/guides/porch-user-guide?id=repository-registration) to
see how to register repositories in Nephio.

### GitOps Tool

As configured in the R1 reference implementation, Nephio relies on ConfigSync.
However, it is possible to configure it to use a different GitOps tool, such as
Flux or ArgoCD to apply packages to the clusters.

### Cluster Provisioner

R1 uses Cluster API, but other options may be used such as Crossplane, Google
KCC, or AWS Controllers for Kubernetes. You can provision more than one.

| Provider                    | Notes                                                                                   |
| --------------------------- | --------------------------------------------------------------------------------------- |
| [Cluster API](capi.md)      | Kubernetes project cluster provisioner for a variety of cluster providers.              |
| [KCC](kcc.md)               | Google's Kubernetes Config Connector for GKE clusters and other GCP resources.          |
| [Crossplane](crossplane.md) | API composition framework with cluster and other infrastructure providers.              |

### Load Balancer

The R1 demo environment uses MetalLB, but if you are running in a cloud, you
probably do not need anything special here. However, depending on your choice of
GitOps tool and Git provider, some of the packages may need customization to
provision or use a well-known load balancer IP.

### Gateway or Ingress

If you wish to avoid running `kubectl port-forward`, the use of Kubernetes
Ingress or Gateway is recommended.

## Common Required Components

We will use `kpt` to install the base Nephio components. There are two essential
components: Porch, and Nephio Controllers.

### Porch

First, we will install the Porch components. This "Package Orchestration"
component provides the Kubernetes APIs for Repositories, PackageRevisions,
PackageVariants, and PackageVariantSets. Nephio relies on it to inventory,
clone, and mutate packages. It also provides the API layer that shields the
Nephio components from direct interaction with the Git storage layer.

Create a directory to hold our local package instances with the various
components:

```bash
mkdir nephio-install
cd nephio-install
```

Next fetch the package using `kpt`, and run any `kpt` functions, and then apply
the package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/porch-dev@v1.0.1
kpt fn render porch-dev
kpt live init porch-dev
kpt live apply porch-dev --reconcile-timeout=15m --output=table
```

### Nephio Controllers

The Nephio Controllers provide implementations of the Nephio-specific APIs. This
includes the code that implements the various package specialization features -
such as integration with IPAM and VLAN allocation, and NAD generation - as well
as controllers that can provision repositories and bootstrap new clusters into
Nephio.

To install the Nephio Controllers, repeat the `kpt` steps, but for that package:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/nephio-controllers@v1.0.1
kpt fn render nephio-controllers
kpt live init nephio-controllers
kpt live apply nephio-controllers --reconcile-timeout=15m --output=table
```

### Management Cluster GitOps Tool

In the R1 demo environment, a GitOps tool (ConfigSync) is installed to allow
GitOps-based application of packages to the management cluster itself. This is
not strictly needed if all you want to do is provision network functions, but it
is used extensively in the cluster provisioning workflows.

Different GitOps tools may be used, but these intructions only cover ConfigSync.
To install it on the management cluster, we again follow the same process.
Later, we will configure it to point to the `mgmt` repository:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/configsync@v1.0.1
kpt fn render configsync
kpt live init configsync
kpt live apply configsync --reconcile-timeout=15m --output=table
```

## Optional Components

### Nephio WebUI

To install the WebUI, we simply install a different kpt package.
First, we pull the package locally:

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-packages.git/nephio-webui@v1.0.1
```

Before we apply it to the cluster, however, we should configure it.

By default, it expects the webui to be reached via `http://localhost:7007`. If
you plan to expose the webui via a load balancer service instead, then you need
to configure the scheme, hostname, port, and service. Note that if you wish to
use HTTPS, you should set the `scheme` to `https`, but you will need to
terminate the TLS at the load balancer as the container currently only supports
HTTP.

This information is captured in the application ConfigMap for the webui, which
is generated by a KRM function. We can change the values in
`nephio-webui/gen-configmap.yaml` just using a text editor (change the
`hostname` and `port` values under `params:`), and those will take effect later
when we run `kpt fn render`. As an alternative to a text editor, you can run
these commands:

```bash
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-kind GenConfigMap -- 'by-path=params.scheme' 'put-value=SCHEME'
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-kind GenConfigMap -- 'by-path=params.hostname' 'put-value=HOSTNAME'
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-kind GenConfigMap -- 'by-path=params.port' 'put-value=PORT'
```

If you want to expose the UI via a load balancer service, you can manually
change the Service `type` to `LoadBalancer`, or run:

```bash
kpt fn eval nephio-webui --image gcr.io/kpt-fn/search-replace:v0.2.0 --match-kind Service -- 'by-path=spec.type' 'put-value=LoadBalancer'
```

In the default configuration, the Nephio WebUI *is wide open with no
authentication*. The webui itself authenticates to the cluster using a static
service account, which is bound to the cluster admin role. Any user accessing
the webui is *acting as a cluster admin*.

This configuration is designed for *testing and development only*. You must not
use this configuration in any other situation, and even for testing and
development it must not be exposed on the Internet (for example, via a
LoadBalancer service).

Configuring authentication for the WebUI is very specific to the particular
cluster environment. Guides for different environments are below:
- [Google OAuth or OIDC](webui-auth-gcp.md)
- [OIDC with Okta](webui-auth-okta.md)

Once that configuration is updated, you can proceed with the installation (note,
this uses `inventory-policy=adopt`, since in the previous steps we may have
created the namespace already).

```bash
kpt fn render nephio-webui
kpt live init nephio-webui
kpt live apply nephio-webui --reconcile-timeout=15m --output=table --inventory-policy=adopt
```

### Nephio Stock Repositories

The repositories with the Nephio packages used in the exercises are available to
be installed via a package for convenience. This will install Repository
resources pointing directly to the GitHub repositories, with read-only access.

```bash
kpt pkg get --for-deployment https://github.com/nephio-project/nephio-example-packages.git/nephio-stock-repos@v1.0.1
kpt fn render nephio-stock-repos
kpt live init nephio-stock-repos
kpt live apply nephio-stock-repos --reconcile-timeout=15m --output=table
```
