# Istio

- https://istiobyexample.dev/

```bash
echo "Cleanup"

kind delete clusters --all

KUBE_API_EXPOSE_ADDRESS=192.168.231.241

echo "KIND"

for CTX in 1 2
do
[ "${CTX}" == "1" ] && podSubnet="10.244.0.0/16" && serviceSubnet="10.96.0.0/12"
echo '[ "${CTX}" == "2" ] && podSubnet="10.245.0.0/16" && serviceSubnet="10.112.0.0/12"'
[ "${CTX}" == "2" ] && podSubnet="10.244.0.0/16" && serviceSubnet="10.96.0.0/12"
cat <<EOF > bm-krk-${CTX}-admin@bm-krk-${CTX}-cluster.yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
networking:
  apiServerAddress: "${KUBE_API_EXPOSE_ADDRESS}"
  apiServerPort: 6443${CTX}
  podSubnet: "${podSubnet}"
  serviceSubnet: "${serviceSubnet}"
EOF

kind create cluster --name c${CTX} --config ./bm-krk-${CTX}-admin@bm-krk-${CTX}-cluster.yaml
done

echo "MetalLB"

for CTX in kind-c1 kind-c2
do

kubectl --context ${CTX} \
    apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/namespace.yaml
kubectl --context ${CTX} \
    create secret generic -n metallb-system memberlist --from-literal=secretkey="$(openssl rand -base64 128)"
kubectl --context ${CTX} \
    apply -f https://raw.githubusercontent.com/metallb/metallb/master/manifests/metallb.yaml
done

for CTX in 1 2
do
[ "${CTX}" == "1" ] && POOL=172.24.255.200-172.24.255.250
[ "${CTX}" == "2" ] && POOL=172.24.255.150-172.24.255.199
cat <<EOF > metallb-c${CTX}.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  namespace: metallb-system
  name: config
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - ${POOL}
EOF

kubectl apply -f metallb-c${CTX}.yaml --context bm-krk-${CTX}-admin@bm-krk-${CTX}
done


echo "Certs"
echo "https://istio.io/latest/docs/tasks/security/cert-management/plugin-ca-cert/"

git clone https://github.com/istio/istio.git && cd istio

mkdir -p certs
pushd certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca

for CTX in 1 2
    do

        make -f ../tools/certs/Makefile.selfsigned.mk bm-krk-${CTX}-cacerts

        kubectl create namespace istio-system --context bm-krk-${CTX}-admin@bm-krk-${CTX}
        kubectl create secret generic cacerts -n istio-system --context bm-krk-${CTX}-admin@bm-krk-${CTX} \
            --from-file=bm-krk-${CTX}/ca-cert.pem \
            --from-file=bm-krk-${CTX}/ca-key.pem \
            --from-file=bm-krk-${CTX}/root-cert.pem \
            --from-file=bm-krk-${CTX}/cert-chain.pem

done

popd

ISTIOCTL_BIN_PATH=../istioctl

for CTX in 1 2
do
kubectl --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" get namespace istio-system && \
  kubectl --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" label namespace istio-system topology.istio.io/network=network${CTX}

cat <<EOF > cluster${CTX}.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster${CTX}
      network: network${CTX}
EOF

${ISTIOCTL_BIN_PATH} install --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" -f cluster${CTX}.yaml -y
samples/multicluster/gen-eastwest-gateway.sh \
    --mesh mesh1 --cluster cluster${CTX} --network network${CTX} | \
    ${ISTIOCTL_BIN_PATH} --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" install -y -f -
kubectl --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" get svc istio-eastwestgateway -n istio-system
kubectl apply --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" -n istio-system -f \
    samples/multicluster/expose-istiod.yaml
kubectl --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml
[ "${CTX}" == "1" ] && CTX_ANOTHER=2
[ "${CTX}" == "2" ] && CTX_ANOTHER=1
${ISTIOCTL_BIN_PATH} x create-remote-secret \
    --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" \
    --name=cluster${CTX} | \
    kubectl apply -f - --context="bm-krk-${CTX_ANOTHER}-admin@bm-krk-${CTX_ANOTHER}"
done

echo "Verify"
echo "https://istio.io/latest/docs/setup/install/multicluster/verify/"

for CTX in 1 2
do
kubectl create --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" namespace sample
kubectl label --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" namespace sample \
    istio-injection=enabled
kubectl apply --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
kubectl apply --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v${CTX} -n sample
kubectl apply --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" \
    -f samples/sleep/sleep.yaml -n sample
done

for CTX in 1 2
do
kubectl wait --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" \
    --for=condition=available --timeout=600s deployment/sleep -n sample
done

for CTX in 1 2
do
for i in $(seq 1 5)
do
kubectl exec --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" -n sample -c sleep \
    "$(kubectl get pod --context="bm-krk-${CTX}-admin@bm-krk-${CTX}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- curl -sS helloworld.sample:5000/hello
done
done

cd ..
```

## ASM bare-metal

```bash
asm/istio/expansion/gen-eastwest-gateway.sh     --mesh ${MESH_ID}     --cluster ${CLUSTER_2}     --network network2     --revision asm-1112-17 |     ./istioctl --kubeconfig=/home/aokhotnikov/baremetal/bmctl-workspace/bm-krk-2/bm-krk-2-kubeconfig install -y --set spec.values.global.pilotCertProvider=kubernetes -f -


kubectl label namespace istio-system topology.istio.io/network=network1 --overwrite
```

## TODO

- [x] Try setup with east-west GW only (without ingress GW) - Works
- [x] Try same pod/svc k8s networks - Works
- [x] Try exposing k8s API (?) via east-west GW - Didn't find docs (https://istio.io/latest/docs/setup/install/multicluster/before-you-begin/#api-server-access) 
- [ ] Verify using Ingress