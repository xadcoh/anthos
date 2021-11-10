# Anthos Experience

Anthos declares a bunch of management solutions for k8s clusters deployed in Google and outside it.

- [Anthos Experience](#anthos-experience)
  - [Goal](#goal)
  - [Results](#results)
    - [Intermediate](#intermediate)
  - [Alternatives](#alternatives)
  - [Installation](#installation)
    - [Bare-metal](#bare-metal)
    - [GCP](#gcp)
  - [Multi-cluster](#multi-cluster)
    - [ASM GCP](#asm-gcp)
    - [ASM Bare-metal](#asm-bare-metal)
    - [ASM Hybrid](#asm-hybrid)
    - [Istio Hybrid](#istio-hybrid)
  - [Cluster Management](#cluster-management)
  - [Config Management](#config-management)
  - [Service Mesh](#service-mesh)
  - [Monitoring](#monitoring)
    - [Logging](#logging)
    - [Metrics](#metrics)
  - [TODO](#todo)

> I'll often ask myself a question: 'Can I solve this without Anthos and how hard can it be?'. The idea behind this is to avoid vendor lock-in (Google Anthos in this case) and try to estimate how much effort do I need to achieve similar results. Yes, Google provides not only the solution itself but it manages everything for you so you get HA and FT, when on-premise is fully yours responsibility.

## Goal

The idea is to validate business case for DB migration from on-prem k8s cluster to GCP hosted GKE cluster using hybrid multi-cluster setup on Anthos components

## Results

### Intermediate

Cons:

- ASM doesn't support hybrid setup (on-prem + GCP)
- ASM web console can show only GCP hosted cluster data
- ASM installed on GCP hosted GKE is barely configurable (no istio-system namespace or any istio-related resources in the cluster)
- Config management has very limited functionality
- Managed Anthos Service Mesh can use multiple GKE clusters in a single Google Cloud project only
- The IstioOperator API isn't supported for managed ASM
- The GKE cluster must be Standard, because Autopilot clusters have Webhooks limitations that don't allow the MutatingWebhookConfiguration for the istio-sidecar-injector.

Pros:

- Config management is easy to setup
- Stackdriver collects all the logs and some metrics from both installations
- containerd CRI
- ASM in-cluster could be configured same way as vanilla istio

Other notes:

- on-prem cluster doesn't automatically register itself to Anthos console
- jump machine require a lot of resources
- on-prem Anthos cluster requires a lot of resources
- istio is installed by-default on on-prem Anthos clusters (istiod: gcr.io/anthos-baremetal-release/asm/pilot:1.10.4-asm.14) along with istio-ingress and it's not configured to inject proxies
- CNI is cilium (anetd: gcr.io/anthos-baremetal-release/cilium/cilium:v1.9.5-gke.46)
- Where ASM sidecar comes from (init,proxy: gcr.io/gke-release/asm/proxyv2:1.11.2-asm.17) - istioctl matters! Google ships their own flavor of istioctl
  - vanilla 'multucluster different networks' guide from istio works fine with asm istioctl and vanilla istioctl

## Alternatives

|Topic|Anthos|OSS Alternatives|
|-----|------|------------|
|Cluster Management|Google Anthos Console|Rancher, OpenShift, kubesphere, cloudfoundry|
|Cluster Installation|bmctl|rke/rke2, kubespray, kubekey|
|Config Management|Google Anthos Console|Argo, Flux, Rancher Fleet|
|Service Mesh Installation|asmcli|istioctl, helm, istio operator|
|Service Mesh|Anthos Service Mesh|istio, linkerd, cilium mesh|
|Service Mesh UI|Anthos Service Mesh Console|Kiali, Jaeger|
|Service Mesh Config|Anthos Service Mesh Console|Admiral, Meshery|
|Serverless|Cloud Run|KNative|
|Logging|Stackdriver|Loki, Elastic/Opensearch|
|Metrics|Stackdriver|Prometheus|

## Installation

### Bare-metal

- 4 CPU, 16Gb RAM, 110Gb storage is required per node
- Jump node should be powerfull enough
- istio is installed by default (not ASM)
- cilium is CNI
- bare-metal cluster automatic registration is commited, but additional steps are required
- bmctl reminds me of rke/rke2
- bmctl will enable required Google APIs automatically
- can't say `hybrid` mode is resource efficient

### GCP

- 6 CPU, 12 RAM: 3 node cluster are minimum suitable setup, but apps will require more resources
- ASM is not installed by default
- istio installed not in `istio-system`, not shure how to configure it (check managed setup)
- istio cni is installed with ASM (?) - this is used by managed ASM, need to double-check the docs

## Multi-cluster

### ASM GCP

- Works out-of-box (tried guide from docs)

### ASM Bare-metal

- Uses same network name (?)
- By default endpoints will be private (one endpoint must be private, second should point to east-west gateway IP of the remote cluster). Changing network from default to two separate networks works, but both endpoints pointed to east-west gw.

### ASM Hybrid

- Not available via ASM
- Easaly could be configured with vanilla istio

### Istio Hybrid

- Just works
  - Install: https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/
  - Works, but issues with certs (https://istio.io/latest/docs/ops/common-problems/validation/)
  - Observability
    - https://istio.io/latest/docs/ops/integrations/prometheus/
    - https://istio.io/latest/docs/ops/integrations/kiali/#installation

## Cluster Management

To get most of Anthos cluster should be GKE

- Can anthos update bare-metal Anthos GKE cluster?

## Config Management

- Really easy to install to the connected clusters
- Functionality is very limited

## Service Mesh

- ASM installed on bare-metal won't be reflected in cloud console
- ASM doesn't support hybrid multi-cluster setups

## Monitoring

### Logging

- Looks like everythin is fine

### Metrics

- Fine

## TODO

- try to label ns with istio.io/rev=asm-managed-rapid instead of istio-injection to check istio deployed by bmctl - failed as it seems like it's ingress only?
- investigate injection (https://istio.io/latest/docs/setup/additional-setup/sidecar-injection/) - no mutatingwebhookconfiguration out-of-box, istioctl flavor matters!
- investigate mesh CA
- investigate ASM in-cluster control plane on GCP and bare-metal (will I see the ASM in Console for GCP?) - Yes
- cilium has service-mesh, why not to leverage it?
- how istio is better then linkerd?
- treat GKE cluster as bare-mteal for ASM - seems like it works fine
- try overlay file - seems like it works fine
- blueprint:
  - AWS
  - Guide
    - Anthos FTW
    - Replace ASM with istio (and what for)
    - Argo instead of CM (and what for)
    - Highlight limitations and how we can address them
  - Anthos Roadmap
  - Competitiors (EKS Anywhere)