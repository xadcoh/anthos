# Anthos

## Why do you need hybrid multicluster?

There are number of scenarios where hybrid multi-cluster setup could helpful:

- Let's say your organization stores and process everything in its private DC, but you would like setup multiple ingresses for your App. So, you could deploy k8s clusters across the Globe in available regions and connect these clusters to your on-prem installation, service discovery, traffic routing, encryption, etc will be handles by service-mesh
- Let's say you would like to process the data as close as possible to the client but still have failover. Service mesh can send traffic locl-first and if local endpoints unavailable send to backup ones
- You could save some money on cloud computing if you have running on-prem infrastructure and don't want deprecate it yet. Just run your App in your DC and delegate some ingress handling to the cloud.
- Hybrid multi-cloud allows you to scale fast and provides freedom of choice in terms of cloud solution to run your workloads
- If you plan to migrate to the cloud, you could leverage hybrid multi-cluster to do it smoothely. Migrate apps one by one saving the connectivity between cloud and on-prem installation
- Also it leads us to all sorts of backup, resiliency and other disaster recovery setups

## Pros and cons of multicluster

Pros:

- Everything from [Why do you need hybrid multicluster?](#why-do-you-need-hybrid-multicluster) really
- Limitation of blast radius. If something goes wrong with one of the clusters, others could handle the workloads
- Possibility of the same configuration. With tools like GitOps it's much easier to handle multiple clusters Apps configuration

Cons:

- Aligned configuratuion require additional software. Like Configuration Managers like Ansible to configure multiple hosts you will need something to keep your clusters configuration aligned. GitOps could help here.
- Software to overview and manage clusters themselves
- Monitoring, logging nd alerting solution become more complex. Would be handy it these will be centralized
- Upgrade procedure become more complex

## What are the options to build multicluster?

You could build multi-cluster in different ways. Main idea here is to automate Service Discovery and Traffic Routing. That's it. You could do it yourself using all sorts of network protocols, load-balancers and other tools. Or you could leverage Service Mesh. All the services within Service mesh will talk to each other without knowing where destintaion is, Service Mesh will handle the traffic routing.

These are some variants:

- Single Service Mesh control plane and multiple data plances
- Multiple Service Mesh Control Planes for each cluster
- Service Mesh can be installed on top of existing installation and handle only part of cluster workloads leaving other as it is
- Service Mesh can be installed as a part of cluster CNI
- Service Mesh can work on top of same L2 subnet (if you already have connectivity in-place)

## Why do you need to choose GKE Anthos?

Yes, why indeed? Well, Anthos covered all the bits and pieces of the multi-cluster setups:

- You could deploy Anthos cluster on-prem and in cloud (not only GCP)
- Clusters could be connected using Anthos Service Mesh
- Clusters Apps could be managed with Anthos Configuration Management
- Anthos collects metrics and logs using Stackdriver and sends them to the cloud
- Anthos cloud console is a single pane of glass for these services

## Process how to setup Anthos GKE

- For GKE in GCP it's straightforward, use gcloud or web interface to create new GKE cluster and register it to the Anthos
- For on-prem process will look as follows
  - Create Google Account and project  
  - Prepare jump host
  - Prepare node hosts
  - Prepare manifest for `bmctl`
  - Provision bare-metal cluster with `bmctl`
  - Register bare-metal cluster to Anthos
  - [Optional] Install Anthos Service Mesh
  - [Optional] Install Anthos Configuration Management

## Problems with Anthos

Anthos has limitations. Probably these will be solved in the future releases.

- Anthos Service Mesh doesn't support hybrid multi-cloud out-of-box, but it is still possible to setup using overlay files (IstioOperator CRDs modifications)
- Anthos Service Mesh Console can display and configure only Anthos Service Mesh deployed in GCP
- Anthos Configuration Management has limited features. Basically it syncs git to cluster and that's it, no healthchecks or so whatever
- There is a prometheus for metrics and log shipper to stackdriver in bare-metal setup, but comparing to GKE in GCP experience will differ. Console for GKE in GCP will have more data
- Anthos Cloud Run could be installed on GKE in GCP on in VMWare only (no bare-metal)

## Alternative options for Anthos

|Topic|Anthos|OSS Alternatives|
|-----|------|------------|
|Cluster Management|Google Anthos Console|Rancher, OKD, kubesphere, cloudfoundry, Cluster API|
|Cluster Installation|bmctl|rke/rke2, kubespray, kubekey|
|Config Management|Google Anthos Console|Argo, Flux, Rancher Fleet|
|Service Mesh Installation|asmcli|istioctl, helm, istio operator, Meshery|
|Service Mesh|Anthos Service Mesh|istio, linkerd, cilium mesh|
|Service Mesh UI|Anthos Service Mesh Console|Kiali, Jaeger|
|Service Mesh Config|Anthos Service Mesh Console|Admiral, Meshery|
|Serverless|Cloud Run|KNative|
|Logging|Stackdriver|Loki, Elastic/Opensearch|
|Metrics|Stackdriver|Prometheus|