# Introduction

[gke](https://cloud.google.com/kubernetes-engine/) exposes a number of alpha and beta features regularly. [Alpha features](https://cloud.google.com/kubernetes-engine/docs/concepts/alpha-clusters) typically are useful for developers especially MI teams to try out new types of clusters for a short term. Such clusters would get automatically deleted after 30 days. Beta features on the other hand are quite useful for DevSecOps teams. They can be used safely in dev and pre-production environments and even in production environments depending on the risk appetite.

The below notes are tailored for terraform, however, the concepts and designs can be reused

## 1. Mix-and-Match google and google-beta providers

Terraform supports a separate beta provider for Google cloud. If a provider is not specified it defaults to `google`. Use the snippet below to mix-and-match providers

```terraform
provider "google" {
  version     = "~> 3.3"
  credentials = file(var.GCP_AUTH_JSON)
  project     = var.GCP_PROJECT
  region      = var.GCP_REGION
}

provider "google-beta" {
  credentials = file(var.GCP_AUTH_JSON)
  project     = var.GCP_PROJECT
  region      = var.GCP_REGION
}
```

Note there is no version for beta.

Then in a subsequent resource block do

```terraform
resource "google_container_cluster" "dtrack_prod_cluster" {
  name                     = var.k8s_cluster_name
  provider                 = google-beta
...
```

Notice the absence of quotes around `google-beta`

## 2. Don't trust the cloud foundation toolkit

[Cloud Foundation Toolkit](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit) is an attempt by Google to code best practices. At this stage it's a nice start. For Google to get better at this like aws probably might take a while. For example, here is the [link](https://github.com/GoogleCloudPlatform/cloud-foundation-toolkit/tree/master/infra/terraform/gke) for their best practice which is quite basic really.

## 3. Use the web console to design and look at the REST data

Unlike AWS web console, Google's version is quite upto date with their sdk and cli. The REST data closely matches with the equivalent terraform structure with some slight deviations.

- convert camelCase to underscore \_ separated variables
- for certain structures with a simple enable flag might have equivalent enable\_ variable. Eg:

```json
"shieldedNodes": {
    "enabled": true
}
```

The above json is equivalent to the below terraform snippet.

```terraform
enable_shielded_nodes = true
```

## 4. Enable security options

Decide whether you need the default `Project Calico` based security or Istio based one. Tweak the network_policy accordingly.

```terraform
network_policy {
    enabled = false
}
```

Enable [shielded nodes](https://cloud.google.com/kubernetes-engine/docs/how-to/shielded-gke-nodes?hl=en_GB)

```terraform
enable_shielded_nodes = true
```

Enable [binary authorization](https://cloud.google.com/binary-authorization/?hl=en_GB). Run the binary authorization in dry-mode for while before implementing cluster-specific rules.

```terraform
enable_binary_authorization = true
```

Use the pod address range to limit the number of pods and services that can be spun up.

Eg: 10.10.0.0/27

Control the oauth scopes

The default compute service account has the following permissions via OAuth scopes.

- https://www.googleapis.com/auth/devstorage.read_only (Read only)
- https://www.googleapis.com/auth/logging.write (Write)
- https://www.googleapis.com/auth/monitoring (Read/Write)
- https://www.googleapis.com/auth/servicecontrol (Read/Write)
- https://www.googleapis.com/auth/service.management.readonly (Read only)
- https://www.googleapis.com/auth/trace.append (Read/Write)

If this setting cannot be determined upfront then it is possible to use full access to cloud-platform scope

- https://www.googleapis.com/auth/cloud-platform

After enabling full-access we need to create bespoke Kubernetes service accounts (KSA) based on Google service accounts (GSA) and [assign](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) the correct KSA to the pods.

Enable master authorized networks for adhoc cluster management activities. Otherwise, just limit to cloudbuild pipeline based deployments and management.

Disable smt and enable audit for pods

- https://github.com/GoogleCloudPlatform/k8s-node-tools/blob/master/disable-smt/gke/disable-smt.yaml
- https://github.com/GoogleCloudPlatform/k8s-node-tools/blob/master/os-audit/cos-auditd-logging.yaml

```
kubectl apply -f <yaml>
```

## 5. Protect your terraform state

A terraform state file is a clear-text json representation of the infrastructure. It will contain all passwords, keys and credentials used in clear-text. Always store the tfstate in an encrypted store and guard it like any other credentials.

- If gcs backend is used then dedicate a separate project and limit the access to only the required services and users.
