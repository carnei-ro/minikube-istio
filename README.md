# minikube / istio

- 3 MiniKube (v1.7.1) instances
- Istio (v1.9) ["Multi-Primary on different networks"](https://istio.io/latest/docs/setup/install/multicluster/multi-primary_multi-network/)

```bash
export CTX_CLUSTER1=cluster1
export CTX_CLUSTER2=cluster2
export CTX_CLUSTER3=cluster3

minikube start --profile=${CTX_CLUSTER1} --cpus=3 --memory=7g --driver=kvm2 --addons storage-provisioner,default-storageclass,metallb
minikube start --profile=${CTX_CLUSTER2} --cpus=3 --memory=7g --driver=kvm2 --addons storage-provisioner,default-storageclass,metallb
minikube start --profile=${CTX_CLUSTER3} --cpus=3 --memory=7g --driver=kvm2 --addons storage-provisioner,default-storageclass,metallb

kubectl --context ${CTX_CLUSTER1} -n metallb-system patch configmaps config --patch '
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.39.200-192.168.39.214'

kubectl --context ${CTX_CLUSTER2} -n metallb-system patch configmaps config --patch '
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.39.215-192.168.39.230'

kubectl --context ${CTX_CLUSTER3} -n metallb-system patch configmaps config --patch '
data:
  config: |
    address-pools:
    - name: default
      protocol: layer2
      addresses:
      - 192.168.39.231-192.168.39.245'

kubectl --context ${CTX_CLUSTER1} -n metallb-system rollout restart daemonset.apps/speaker
kubectl --context ${CTX_CLUSTER1} -n metallb-system rollout restart deployment controller
kubectl --context ${CTX_CLUSTER2} -n metallb-system rollout restart daemonset.apps/speaker
kubectl --context ${CTX_CLUSTER2} -n metallb-system rollout restart deployment controller
kubectl --context ${CTX_CLUSTER3} -n metallb-system rollout restart daemonset.apps/speaker
kubectl --context ${CTX_CLUSTER3} -n metallb-system rollout restart deployment controller

kubectl --context ${CTX_CLUSTER1} -n metallb-system get all

###

wget https://github.com/istio/istio/releases/download/1.9.0/istio-1.9.0-linux-amd64.tar.gz
tar xzvf istio-1.9.0-linux-amd64.tar.gz
cd istio-1.9.0/

###

mkdir -p certs
pushd certs
make -f ../tools/certs/Makefile.selfsigned.mk root-ca
make -f ../tools/certs/Makefile.selfsigned.mk ${CTX_CLUSTER1}-cacerts
make -f ../tools/certs/Makefile.selfsigned.mk ${CTX_CLUSTER2}-cacerts
make -f ../tools/certs/Makefile.selfsigned.mk ${CTX_CLUSTER3}-cacerts

kubectl --context="${CTX_CLUSTER1}" create ns istio-system
kubectl --context="${CTX_CLUSTER1}" create secret generic cacerts -n istio-system \
      --from-file=${CTX_CLUSTER1}/ca-cert.pem \
      --from-file=${CTX_CLUSTER1}/ca-key.pem \
      --from-file=${CTX_CLUSTER1}/root-cert.pem \
      --from-file=${CTX_CLUSTER1}/cert-chain.pem

kubectl --context="${CTX_CLUSTER2}" create ns istio-system
kubectl --context="${CTX_CLUSTER2}" create secret generic cacerts -n istio-system \
      --from-file=${CTX_CLUSTER2}/ca-cert.pem \
      --from-file=${CTX_CLUSTER2}/ca-key.pem \
      --from-file=${CTX_CLUSTER2}/root-cert.pem \
      --from-file=${CTX_CLUSTER2}/cert-chain.pem

kubectl --context="${CTX_CLUSTER3}" create ns istio-system
kubectl --context="${CTX_CLUSTER3}" create secret generic cacerts -n istio-system \
      --from-file=${CTX_CLUSTER3}/ca-cert.pem \
      --from-file=${CTX_CLUSTER3}/ca-key.pem \
      --from-file=${CTX_CLUSTER3}/root-cert.pem \
      --from-file=${CTX_CLUSTER3}/cert-chain.pem

popd
###

kubectl --context="${CTX_CLUSTER1}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER1}" label namespace istio-system topology.istio.io/network=network1

cat <<EOF > cluster1.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster1
      network: network1
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: network1
        enabled: true
        k8s:
          env:
            # sni-dnat adds the clusters required for AUTO_PASSTHROUGH mode
            - name: ISTIO_META_ROUTER_MODE
              value: "sni-dnat"
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: network1
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
EOF


bin/istioctl install --context="${CTX_CLUSTER1}" -f cluster1.yaml -y

kubectl --context="${CTX_CLUSTER1}" -n istio-system get all

kubectl --context="${CTX_CLUSTER1}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml

###

kubectl --context="${CTX_CLUSTER2}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER2}" label namespace istio-system topology.istio.io/network=network2

cat <<EOF > cluster2.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster2
      network: network2
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: network2
        enabled: true
        k8s:
          env:
            # sni-dnat adds the clusters required for AUTO_PASSTHROUGH mode
            - name: ISTIO_META_ROUTER_MODE
              value: "sni-dnat"
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: network2
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
EOF

bin/istioctl install --context="${CTX_CLUSTER2}" -f cluster2.yaml -y

kubectl --context="${CTX_CLUSTER1}" -n istio-system get all

kubectl --context="${CTX_CLUSTER2}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml

###

kubectl --context="${CTX_CLUSTER3}" get namespace istio-system && \
  kubectl --context="${CTX_CLUSTER3}" label namespace istio-system topology.istio.io/network=network3

cat <<EOF > cluster3.yaml
apiVersion: install.istio.io/v1alpha1
kind: IstioOperator
spec:
  values:
    global:
      meshID: mesh1
      multiCluster:
        clusterName: cluster3
      network: network3
  components:
    ingressGateways:
      - name: istio-eastwestgateway
        label:
          istio: eastwestgateway
          app: istio-eastwestgateway
          topology.istio.io/network: network3
        enabled: true
        k8s:
          env:
            # sni-dnat adds the clusters required for AUTO_PASSTHROUGH mode
            - name: ISTIO_META_ROUTER_MODE
              value: "sni-dnat"
            # traffic through this gateway should be routed inside the network
            - name: ISTIO_META_REQUESTED_NETWORK_VIEW
              value: network3
          service:
            ports:
              - name: status-port
                port: 15021
                targetPort: 15021
              - name: tls
                port: 15443
                targetPort: 15443
              - name: tls-istiod
                port: 15012
                targetPort: 15012
              - name: tls-webhook
                port: 15017
                targetPort: 15017
EOF

bin/istioctl install --context="${CTX_CLUSTER3}" -f cluster3.yaml -y

kubectl --context="${CTX_CLUSTER3}" -n istio-system get all

kubectl --context="${CTX_CLUSTER3}" apply -n istio-system -f \
    samples/multicluster/expose-services.yaml

###

bin/istioctl x create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"

bin/istioctl x create-remote-secret \
  --context="${CTX_CLUSTER3}" \
  --name=cluster3 | \
  kubectl apply -f - --context="${CTX_CLUSTER1}"

bin/istioctl x create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=cluster1 | \
  kubectl apply -f - --context="${CTX_CLUSTER2}"

bin/istioctl x create-remote-secret \
  --context="${CTX_CLUSTER3}" \
  --name=cluster3 | \
  kubectl apply -f - --context="${CTX_CLUSTER2}"

bin/istioctl x create-remote-secret \
  --context="${CTX_CLUSTER1}" \
  --name=cluster1 | \
  kubectl apply -f - --context="${CTX_CLUSTER3}"

bin/istioctl x create-remote-secret \
  --context="${CTX_CLUSTER2}" \
  --name=cluster2 | \
  kubectl apply -f - --context="${CTX_CLUSTER3}"

### verify

kubectl create --context="${CTX_CLUSTER1}" namespace sample
kubectl create --context="${CTX_CLUSTER2}" namespace sample
kubectl create --context="${CTX_CLUSTER3}" namespace sample

kubectl label --context="${CTX_CLUSTER1}" namespace sample \
    istio-injection=enabled
kubectl label --context="${CTX_CLUSTER2}" namespace sample \
    istio-injection=enabled
kubectl label --context="${CTX_CLUSTER3}" namespace sample \
    istio-injection=enabled

kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample

# Atencao a essa parte, estamos criando so o service, nao o deployment...    
kubectl apply --context="${CTX_CLUSTER3}" \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample

kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v1 -n sample
kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l app=helloworld

kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/helloworld/helloworld.yaml \
    -l version=v2 -n sample
kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l app=helloworld

kubectl apply --context="${CTX_CLUSTER1}" \
    -f samples/sleep/sleep.yaml -n sample
kubectl apply --context="${CTX_CLUSTER2}" \
    -f samples/sleep/sleep.yaml -n sample
kubectl apply --context="${CTX_CLUSTER3}" \
    -f samples/sleep/sleep.yaml -n sample

kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l app=sleep

kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l app=sleep

kubectl exec --context="${CTX_CLUSTER1}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_CLUSTER1}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- sh -c "while true; do curl -sS helloworld.sample:5000/hello; done"
    
kubectl exec --context="${CTX_CLUSTER2}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_CLUSTER2}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- sh -c "while true; do curl -sS helloworld.sample:5000/hello; done"

# Isso so ta funcionando pq criei o namespace "sample" e o service "helloworld" no cluster3.
kubectl exec --context="${CTX_CLUSTER3}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_CLUSTER3}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- sh -c "while true; do curl -sS helloworld.sample:5000/hello; done"
```
