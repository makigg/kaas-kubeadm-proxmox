# クラスタのデプロイ方法

## cluster01をデプロイ

クラスタのマニフェストを作成する

```shell
kubectl apply -f cluster01.yaml
```

kubeconfigの取得

```shell
clusterctl get kubeconfig cluster01 -n kaas > ~/.kube/cluster01.config

if [ -z "$KUBECONFIG" ]; then
  export KUBECONFIG=${HOME}/.kube/config
fi
export KUBECONFIG=${KUBECONFIG}:${HOME}/.kube/cluster01.config
```

Ciliumのインストール

```shell
cilium install --context=cluster01-admin@cluster01
```

各ノードにCSIのラベルを付ける。

```shell
kubectl label node --all topology.kubernetes.io/region=region1 topology.kubernetes.io/zone=pve1 --context=cluster01-admin@cluster01
```

## cluster02をデプロイ

クラスタのマニフェストを作成する

```shell
kubectl apply -f cluster02.yaml
```

kubeconfigの取得

```shell
clusterctl get kubeconfig cluster02 -n kaas > ~/.kube/cluster02.config

if [ -z "$KUBECONFIG" ]; then
  export KUBECONFIG=${HOME}/.kube/config
fi
export KUBECONFIG=${KUBECONFIG}:${HOME}/.kube/cluster02.config
```

Ciliumのインストール

```shell
cilium install --context=cluster02-admin@cluster02
```

各ノードにCSIのラベルを付ける。

```shell
kubectl label node --all topology.kubernetes.io/region=region1 topology.kubernetes.io/zone=pve1 --context=cluster02-admin@cluster02
```

## clusterXXのデプロイ

cluster01を例にとる

```shell
export CLUSTER=cluster01
```

マニフェストを作成して、クラスタをデプロイ

```shell
# cluster01.yamlを作成
cat <<EOF > ${CLUSTER}.yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: ${CLUSTER}
  namespace: kaas
  labels:
    csi-proxmox: "true"
spec:
  topology:
    class: kubeadm-v1.30.5
    version: v1.30.5
    controlPlane:
      replicas: 1
    workers:
      machineDeployments:
      - class: default-worker
        name: cpu-node
        replicas: 1
    variables:
    - name: controlPlaneHost
      value: "192.168.10.210"
    - name: ipV4Addresses
      value:
      - "192.168.10.211-192.168.10.214"
EOF

# クラスタのデプロイ
kubectl apply -f ${CLUSTER}.yaml
```

kubeconfigの取得

```shell
clusterctl get kubeconfig ${CLUSTER} -n kaas > ~/.kube/${CLUSTER}.config

if [ -z "$KUBECONFIG" ]; then
  export KUBECONFIG=${HOME}/.kube/config
fi
export KUBECONFIG=${KUBECONFIG}:${HOME}/.kube/${CLUSTER}.config
```

Ciliumのインストール

```shell
cilium install --context=${CLUSTER}-admin@${CLUSTER}
```

各ノードにCSIのラベルを付ける。

```shell
kubectl label node --all topology.kubernetes.io/region=region1 topology.kubernetes.io/zone=pve1 --context=${CLUSTER}-admin@${CLUSTER}
```
