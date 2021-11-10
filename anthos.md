# Anthos

- How to adopt a multi-cluster strategy for your applications in Anthos: https://www.youtube.com/watch?v=ZhF-rTXq-Us
- Bare-metal Quickstart doc: https://cloud.google.com/anthos/clusters/docs/bare-metal/latest/quickstart#installing-bmctl
- Google Cloud Anthos Clusters on Bare Metal: https://www.youtube.com/watch?v=wDpdHjHWYMg
- Google Anthos Cluster on Bare Metal Hands-on Lab Tutorial: https://www.youtube.com/watch?v=Ed8M4TTldmQ
- Introduction to Anthos on bare metal: https://www.youtube.com/watch?v=RE0A3kHT3LA
- An introduction to Anthos (Google Cloud Community Day ‘19): https://www.youtube.com/watch?v=42RmVrM7B7E
- Globally available BoA: https://github.com/GoogleCloudPlatform/bank-of-anthos/tree/master/extras/cloudsql-multicluster
- Online book: https://cloudsolutions.academy/how-to/anthos-in-a-nutshell/deployment-options-with-anthos/
- Extending the Anthos Consistent Experience to AWS (Cloud Next ‘19 UK) https://www.youtube.com/watch?v=qnlrEXOGFz4, https://www.youtube.com/watch?v=_jPkVcyejI8
- https://docs.netapp.com/us-en/hci-solutions/anthos_task_complete_anthos_prerequisites.html
- https://www.youtube.com/watch?v=clu7t0LVhcw
- https://medium.com/google-cloud/kubernetes-engine-gke-multi-cluster-life-cycle-management-series-ee0f583d9b10

## Single cluster

```bash
gcloud auth login
```

> specify exact zone like `europe-central2-a` or node-pool will be created in each zone (2 node x 3 zones = total 6 nodes)

```bash
export PROJECT_ID=multicloud-330115
export ZONE=europe-central2-a

```


```bash
gcloud container clusters create cluster-1  \
    --project=${PROJECT_ID} \
    --zone=${ZONE}  \
    --machine-type=e2-standard-4 \
    --num-nodes=2 \
    --workload-pool=${PROJECT_ID}.svc.id.goog

```

get kubeconfig

```
gcloud container clusters get-credentials cluster-1  \
    --project=${PROJECT_ID} \
    --zone=${ZONE}
kubectl config set-context cluster-1 
```

```
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.11 > asmcli
chmod +x asmcli
```

```
./asmcli install \
  --project_id ${PROJECT_ID} \
  --cluster_name cluster-1 \
  --cluster_location ${ZONE} \
  --option legacy-default-ingressgateway \
  --enable_all
```

Deploy app: https://cloud.google.com/service-mesh/docs/unified-install/quickstart-asm#deploy_the_online_boutique_sample

## Multi-cluster

> https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster
> https://binx.io/blog/2021/07/23/how-to-deploy-a-multi-cluster-service-mesh-on-gke-with-anthos/

pre-requisites

```bash
export PROJECT_ID=multicloud-330115
export ZONE=europe-central2

```

```bash
gcloud container clusters create cluster-1  \
    --project=${PROJECT_ID} \
    --zone=${ZONE}-a  \
    --machine-type=e2-standard-4 \
    --num-nodes=2 \
    --workload-pool=${PROJECT_ID}.svc.id.goog

gcloud container clusters create cluster-2  \
    --project=${PROJECT_ID} \
    --zone=${ZONE}-b  \
    --machine-type=e2-standard-4 \
    --num-nodes=2 \
    --workload-pool=${PROJECT_ID}.svc.id.goog

./asmcli install \
  --project_id ${PROJECT_ID} \
  --cluster_name cluster-1 \
  --cluster_location ${ZONE}-a \
  --option legacy-default-ingressgateway \
  --enable_all

./asmcli install \
  --project_id ${PROJECT_ID} \
  --cluster_name cluster-2 \
  --cluster_location ${ZONE}-b \
  --option legacy-default-ingressgateway \
  --enable_all
```

## Hello World

```bash
for CTX in $(seq 1 2)
do
kubectl delete --kubeconfig /home/aokhotnikov/baremetal/bmctl-workspace/krk-bm-${CTX}/krk-bm-${CTX}-kubeconfig namespace sample
kubectl create --kubeconfig /home/aokhotnikov/baremetal/bmctl-workspace/krk-bm-${CTX}/krk-bm-${CTX}-kubeconfig namespace sample
kubectl label --kubeconfig /home/aokhotnikov/baremetal/bmctl-workspace/krk-bm-${CTX}/krk-bm-${CTX}-kubeconfig namespace sample \
    istio-injection- istio.io/rev=asm-1112-17 --overwrite

kubectl create --kubeconfig /home/aokhotnikov/baremetal/bmctl-workspace/krk-bm-${CTX}/krk-bm-${CTX}-kubeconfig \
    -f ${SAMPLES_DIR}/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample

kubectl create --kubeconfig /home/aokhotnikov/baremetal/bmctl-workspace/krk-bm-${CTX}/krk-bm-${CTX}-kubeconfig \
  -f ${SAMPLES_DIR}/samples/helloworld/helloworld.yaml \
  -l version=v${CTX} -n sample

kubectl apply --kubeconfig /home/aokhotnikov/baremetal/bmctl-workspace/krk-bm-${CTX}/krk-bm-${CTX}-kubeconfig \
   -f ${SAMPLES_DIR}/samples/sleep/sleep.yaml -n sample
done
```


## Mesh private

- https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster#private-clusters-endpoint

## Bare-metal

> https://cloud.google.com/anthos/clusters/docs/bare-metal/latest/installing/configure-sa

```bash
cd baremetal

gcloud services enable --project=istio-330412 \
servicemanagement.googleapis.com \
servicecontrol.googleapis.com
gcloud config set project istio-330412
gcloud services enable --project=istio-330412 \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    cloudresourcemanager.googleapis.com \
    anthos.googleapis.com
gcloud iam service-accounts create connect-agent-svc-account --project=istio-330412
gcloud projects add-iam-policy-binding  istio-330412 \
    --member="serviceAccount:connect-agent-svc-account@istio-330412.iam.gserviceaccount.com" \
    --role="roles/gkehub.connect"
gcloud iam service-accounts keys create connect-agent.json \
    --iam-account=connect-agent-svc-account@istio-330412.iam.gserviceaccount.com \
    --project=istio-330412
gcloud iam service-accounts create connect-register-svc-account \
    --project=istio-330412
gcloud projects add-iam-policy-binding istio-330412 \
    --member="serviceAccount:connect-register-svc-account@istio-330412.iam.gserviceaccount.com" \
    --role=roles/gkehub.admin
gcloud services enable --project istio-330412 \
    anthos.googleapis.com \
    anthosaudit.googleapis.com \
    anthosgke.googleapis.com \
    cloudresourcemanager.googleapis.com \
    gkeconnect.googleapis.com \
    gkehub.googleapis.com \
    serviceusage.googleapis.com \
    stackdriver.googleapis.com \
    monitoring.googleapis.com \
    logging.googleapis.com \
    opsconfigmonitoring.googleapis.com
gcloud iam service-accounts create logging-monitoring-svc-account \
    --project=istio-330412
gcloud projects add-iam-policy-binding istio-330412 \
    --member="serviceAccount:logging-monitoring-svc-account@istio-330412.iam.gserviceaccount.com" \
    --role="roles/logging.logWriter"
gcloud projects add-iam-policy-binding istio-330412 \
    --member="serviceAccount:logging-monitoring-svc-account@istio-330412.iam.gserviceaccount.com" \
    --role="roles/monitoring.metricWriter"
gcloud projects add-iam-policy-binding istio-330412 \
    --member="serviceAccount:logging-monitoring-svc-account@istio-330412.iam.gserviceaccount.com" \
    --role="roles/stackdriver.resourceMetadata.writer"
gcloud projects add-iam-policy-binding istio-330412 \
    --member="serviceAccount:logging-monitoring-svc-account@istio-330412.iam.gserviceaccount.com" \
    --role="roles/opsconfigmonitoring.resourceMetadata.writer"
gcloud projects add-iam-policy-binding istio-330412 \
    --member="serviceAccount:logging-monitoring-svc-account@istio-330412.iam.gserviceaccount.com" \
    --role="roles/monitoring.dashboardEditor"
gcloud iam service-accounts keys create cloud-ops.json \
    --iam-account=logging-monitoring-svc-account@istio-330412.iam.gserviceaccount.com \
    --project=istio-330412
```

> https://cloud.google.com/anthos/clusters/docs/bare-metal/latest/quickstart

```
./bmctl create config -c krk-bm-1 \
  --enable-apis --create-service-accounts --project-id=istio-330412
```

> https://cloud.google.com/anthos/multicluster-management/console/logging-in#create_ksa

Upgrade ASM: https://cloud.google.com/service-mesh/docs/unified-install/upgrade

## Hybrid multicluster

- install anthos on bare metal
- install gke with workload identity in gcp
- install ASM with asmcli on gke in gcp with overlay

```yaml
---
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      multiCluster:
        # Provided to ensure a human readable name rather than a UUID.
        # clusterName: "cn-asmtest-331513-europe-central2-a-asm-cluster" # {"$ref":"#/definitions/io.k8s.cli.substitutions.cluster-name"}
        clusterName: cluster1
      meshID: mesh1
      network: network1
```

- install ASM using asmcli on bare metal with overlay

```yaml
---
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      multiCluster:
        # Provided to ensure a human readable name rather than a UUID.
        # clusterName: "cn-asmtest-331513-europe-central2-a-asm-cluster" # {"$ref":"#/definitions/io.k8s.cli.substitutions.cluster-name"}
        clusterName: cluster2
      meshID: mesh1
      network: network2
```

- Use `Set up a multi-cluster mesh outside Google Cloud` to create mesh (use meshID, network and cluster names from overlay)
- Label `istio-system` namespaces accordingly (cluster1=network1)

```bash
kubectl label namespace istio-system topology.istio.io/network=network1 --overwrite
```

- Ensure your LB is configured properly to pass traffic to NodePort/LoadBalancer service dedicated for eastwest gateway
