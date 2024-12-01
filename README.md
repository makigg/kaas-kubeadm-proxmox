# 誤自宅KaaS (Kubernetes as a Service)

家に転がっていた古いPCにProxmox VEを入れて、その上にKaaSを構築した時のメモ。

K3s上にClusterリソースを作成すると、Proxmox上にK8sクラスタがデプロイされるようにする。

![](images/overview.drawio.svg)

使用技術

- K8sクラスタをオンデマンドでデプロイするというKaaSのコア機能はCluster APIを使って実現している。
- クラスタの構築はkubeadmを使っている。
- VMテンプレートの作成にはPackerを使用した。

注意点

- Packerによるイメージテンプレート作成はDHCPを前提としている。
- K8sクラスタの各ノードは固定IPアドレスを前提として設計してある。

参考資料

- [Cluster API](https://cluster-api.sigs.k8s.io/)
- [Building Images for Proxmox VE](https://image-builder.sigs.k8s.io/capi/providers/proxmox)

## 大まかな手順

1. Proxmox VEそのものの設定
2. ゴールデンイメージの作成(Packer)
3. Cluster APIのインストール
4. ClusterClassの作成
5. Clusterデプロイ

## ハードウェアの構成

CPU: Ryzen 7 5800x
Memory: 64GB
SSD1: 500GB (OS, /dev/nvme1n1)
SSD2: 2TB (/dev/nvme0n1)
GPU: RTX 3060 8GB (今回は使わない)

## Proxmox VEの設定

### LVM Thinプールを作成する

VMやPersistentVolumeのデータ格納先として使うため、LVM Thinプールを作成する。
LVMプールはボリューム作成時点でブロックが割り当てられるが、LVM Thinプールは書き込みが発生したとき初めてブロックが割り当てられるらしい。そのため、容量効率を考えてLVM Thinプールを使うことにした。

作成するLVM Thinプール名: `data`

#### SSD2を初期化する

Proxmoxをデフォルト設定でインストールすると、SSD2(`/dev/nvme0n1`)が何も設定されていない状態だったので、初期化して既存のボリュームグループ(VG)`pve`に追加する。新しく作ったほうがいいのかよくわからなかったので、とりあえず、既存のものを使う。

[この手順](https://static.xtremeownage.com/blog/2023/proxmox---add-disk-to-existing-thin-provisioned-lvm/)が参考にする。ただし、追加先のVGが異なるため注意。

SSD2をフォーマット。ラベルはGPTを選択。あとはデフォルト設定。

```shell
cfdisk /dev/nvme0n1
```

物理ディスク(PV)を作成。

```shell
pvcreate /dev/nvme0n1p1
```

一応、既存VG`pve`が存在するか確認する。

```shell
vgscan
```

VG`pve`にPV`/dev/nvme0n1p1`を追加する。

```shell
vgextend pve /dev/nvme0n1p1
```

#### LVM Thinを作成

Proxmoxの画面(pve1 > Disks > LVM-Thin)からLVM Thinプールを作成する。

## VMのゴールデンイメージを作成する

### packerをインストールする。

```shell
mise use --global packer@latest
```

### ゴールデンイメージの作成

[Proxmox用のイメージ作成手順](https://image-builder.sigs.k8s.io/capi/providers/proxmox)を参考にしてゴールデンイメージを作成していく。

#### 設定の書き換え

[kubernetes-sigs/image-builder](https://github.com/kubernetes-sigs/image-builder)を自分の環境用に書き換えて使う。

LVM Thinプールを使う場合、qcow2が使えないので、formatをrawに設定する。

```diff
--- a/images/capi/packer/proxmox/packer.json.tmpl
+++ b/images/capi/packer/proxmox/packer.json.tmpl
@@ -13,7 +13,7 @@
       "disks": [
         {
           "disk_size": "{{user `disk_size`}}",
-          "format": "qcow2",
+          "format": "raw",
           "storage_pool": "{{user `storage_pool`}}",
           "storage_pool_type": "{{user `storage_pool_type`}}",
           "type": "scsi"
```

IPv6を無効化する。

```diff
--- a/images/capi/packer/proxmox/ubuntu-2404.json
+++ b/images/capi/packer/proxmox/ubuntu-2404.json
@@ -1,5 +1,5 @@
 {
-  "boot_command_prefix": "c<wait>linux /casper/vmlinuz --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/24.04/'<enter><wait10s>initrd /casper/initrd<enter><wait10s>boot<enter><wait10s>",
+  "boot_command_prefix": "c<wait>linux /casper/vmlinuz ipv6.disable=1 --- autoinstall ds='nocloud-net;s=http://{{ .HTTPIP }}:{{ .HTTPPort }}/24.04/'<enter><wait10s>initrd /casper/initrd<enter><wait10s>boot<enter><wait10s>",
   "build_name": "ubuntu-2404",
   "distribution_version": "2404",
   "distro_name": "ubuntu",
```

#### Packerの実行

トークンを作成する

```shell
pveum user add packer@pve
pveum aclmod / -user packer@pve -role PVEVMAdmin
pveum user token add packer@pve packer -privsep 0
```

こんな感じの出力が得られる。トークンIDとトークン(value)をメモする。

```shell
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ packer@pve!packer                    │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
└──────────────┴──────────────────────────────────────┘
```

Packerを実行して、VMのゴールデンイメージを作成する。

```shell
export PROXMOX_URL="https://192.168.10.138:8006/api2/json"
export PROXMOX_USERNAME='packer@pve!packer'
export PROXMOX_TOKEN="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export PROXMOX_NODE="pve1"
export PROXMOX_ISO_POOL="local"
export PROXMOX_BRIDGE="vmbr0"
# 作った覚えがないが、Datacenter > Storageから見えるとあった。
# 先ほど作ったdataという名前のLVM Thinプールを束ねるものっぽい？
export PROXMOX_STORAGE_POOL="local-lvm"

make build-proxmox-ubuntu-2404
```

Proxmox VEの画面で作成されたVMテンプレートのVM IDを控える。ProxmoxMachineTemplateの設定に使う。

## Cluster APIの設定

ほぼ、[公式ドキュメントの手順](https://cluster-api.sigs.k8s.io/user/quick-start)通り。

### clusterctlのインストール

```shell
mise use --global clusterctl@latest
```

### Cluster APIのインストール

トークンを生成(Proxmoxのホスト上で実行)

```shell
pveum user add capmox@pve
pveum aclmod / -user capmox@pve -role PVEVMAdmin
pveum user token add capmox@pve capi -privsep 0
```

こんな感じの出力が得られる。トークンIDとトークン(value)をメモする。

```shell
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ capmox@pve!capi                      │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
└──────────────┴──────────────────────────────────────┘
```

Cluster APIをインストール

```shell
# ProxmoxのAPIへのURL
export PROXMOX_URL="https://192.168.10.138:8006"
# ProxmoxのToken ID
export PROXMOX_TOKEN='capmox@pve!capi'
# ProxmoxのToken
export PROXMOX_SECRET="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"

# Finally, initialize the management cluster
clusterctl init --infrastructure proxmox --ipam in-cluster
```

### 動作確認としてクラスタをデプロイする

動作確認でクラスタをデプロイしてみる。

```shell
export PROXMOX_SOURCENODE="pve1"
# The template VM ID used for cloning VMs
export TEMPLATE_VMID=106
# The ssh authorized keys used to ssh to the machines.
export VM_SSH_KEYS="ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQDhQKh2EmK+5FWCw89zHw1aBewgp8S2L/mNlZr10Dx6Dv7x6HCt/ooLJiwIXz+Lfj9pB2ikLbZ14zOP635dCjnSRFky8yHJwH5RpvuLFFOLmoELRFFNJPq5cuVsDXjMod/1rDSajqXY0Ml0cmQIQKvR1Nr/+8rVWrvVjt0pwsWEPpBNPl0L/xgrHgaQIQj6JAQco14XnY0n6bXEOewswKDWmUxdPYoOgMH+dokjjKuvJAuBDhYiGYrD9XwClduyDORDt3OwmuZYDGEIQLBa4oXLF1J8zF0tIDDwYlN+vh2zx2HxLJ6kIk+2FgW+mnvVi3s3v174GqjeIqoMF8EJtcB+IH4ACqruPdjiAgKR581wStcdfjISjcP0AYe9t6L86uG1aIViWPWL7RPZNY4KR4RsYwk9kXGOA6LVr0VmfJvNslU7+oCb67XlxLRHUNBOMy8EfadIXDoIz3FUch1HW2BNbhkStrujrEqk1B5/zlsFRQ7shmI3ccHr9QByslxcFgs=,ssh-rsa AAAAB3NzaC1yc2EAAAADAQABAAABgQCjp0UvoAQy+wUpfEHr1YVU04htWc0qnVhtSCeXVD1Q8kiz/7itIKKfKGvz+PFU9kNcQlrQaUxQ/lh9SUC0y/epVLcxm3TBI99HTeeWTrO1RLvDIR0CS3iC2USPMEX0jbhF2fUoE+Q+vaZArNsP/zR1fEV2Dc5HW/dgjcKwNN9Xr7p6BujkwOQXM8DdqcTTMIdRoZhisD7LGDfWkro1fcoo0eeMGgdcGx/DJLw0s840t1teVxE0Payt4LGMtg/jOCOCXRTzmvLDSfdxBjxYGhT2IK/GFPoC/L2frFaie0N2dRRGI0j4SBgNQbJudQanJvsuT35UkMUrxgkANyrbFQmVmcJhuwo3mbXRLaBSojpBv3AQ3eF5X1WKvTNcZ3H04E0acReUtQcy00CP+8rI1uZ+bhC6NxgR2g1GPKoJZA2EJp4LF9oqyczQzKz62dXGXV5I9c2Sf2Q+q4bVIQjKc79U5EdQnQkTEIyEfI+ib+XvclrT1c1M95uTw1cogOUBDhk="
# The IP address used for the control plane endpoint
export CONTROL_PLANE_ENDPOINT_IP=192.168.10.210
# The IP ranges for Cluster nodes
export NODE_IP_RANGES="[192.168.10.211-192.168.10.214]"
# The gateway for the machines network-config.
export GATEWAY="192.168.10.1"
# Subnet Mask in CIDR notation for your node IP ranges
export IP_PREFIX=24
# The Proxmox network device for VMs
export BRIDGE="vmbr0"
# The dns nameservers for the machines network-config.
export DNS_SERVERS="[8.8.8.8,8.8.4.4]"
# The Proxmox nodes used for VM deployments
export ALLOWED_NODES="[pve1]"
export BOOT_VOLUME_DEVICE="scsi0"

clusterctl generate cluster cluster01 --kubernetes-version v1.30.5 > generated/cluster01-gen.yaml

kubectl apply -f generated/cluster01-gen.yaml
```

## ClusterClassの設定

### Cluster APIのClusterTopology機能を有効にする

ClusterClassを使うためにClusterTopologyを有効にする。以下のDeploymentを修正した。

- capi-system/capi-controller-manager
- capmox-system/capmox-controller-manager
- capi-kubeadm-control-plane-system/capi-kubeadm-control-plane-controller-manager

clusterctl init時点で有効にすることもできる。

### ClusterClassを書いていく

動作確認の時に精製したcluster01-gen.yamlを参照しつつ、ClusterClassを書いていく。

| コンポーネント              | 説明                                                                                                                    |
| --------------------------- | ----------------------------------------------------------------------------------------------------------------------- |
| ProxmoxMachineTemplate      | 作成するVMのスペック。コントロールプレーン用とワーカー用、それぞれ用意する。                                            |
| KubeadmControlPlaneTemplate | 主にコントロールプレーンのstatic podの設定を行う。他にもkubeadm init/join実行前後で実行したいコマンドを定義したりする。 |
| KubeadmConfigTemplate       | kubeadm init/joinコマンドの設定。今回はワーカーノード用に使っている。                                                   |
| ProxmoxClusterTemplate      | KubernetesのエンドポイントやノードのIPアドレスを設定したりする。                                                        |

上記を参照するClusterClassを作成する。

[作成されたもの](./clusterclass/)

## クラスタのデプロイ

作成済みの[クラスタマニフェスト](cluster/cluster01.yaml)があるのでそれをデプロイする。

```shell
kubectl apply -f cluster/cluster01.yaml
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

## CSIを設定する

### Cluster APIのClusterResourceSet機能を有効にする

capi-system/capi-controller-managerを修正してClusterResourceSetを有効にする。

### ゴールデンイメージを作成しなおす

[github](https://github.com/sergelogvinov/proxmox-csi-plugin?tab=readme-ov-file#proxmox-vm-config)に記載があるように、ディスクコントローラーにはVirtIO SCSI singleを使う必要があるため、ゴールデンイメージの修正が必要になる。

これくらいなら直接VMテンプレートを編集しちゃったほうが早い気もするが、一応、packerから実行しなおした。

ディスクコントローラーにVirtIO SCSI singleが使われるようpackerの設定ファイルを変更。

```diff
--- a/images/capi/packer/proxmox/packer.json.tmpl
+++ b/images/capi/packer/proxmox/packer.json.tmpl
@@ -42,7 +42,8 @@
       "template_name": "{{ user `artifact_name` }}",
       "type": "proxmox-iso",
       "unmount_iso": "{{user `unmount_iso`}}",
-      "vm_id": "{{user `vmid`}}"
+      "vm_id": "{{user `vmid`}}",
+      "scsi_controller": "virtio-scsi-single"
     }
   ],
   "post-processors": [
```

ゴールデンイメージ作成する。

```shell
export PROXMOX_URL="https://192.168.10.138:8006/api2/json"
export PROXMOX_USERNAME='packer@pve!packer'
export PROXMOX_TOKEN="xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
export PROXMOX_NODE="pve1"
export PROXMOX_ISO_POOL="local"
export PROXMOX_BRIDGE="vmbr0"
export PROXMOX_STORAGE_POOL="local-lvm"

make build-proxmox-ubuntu-2404
```

ProxmoxMachineTemplateのtemplateIDを書き換えて、ClusterClassを作成しなおす。

### Proxmoxクラスタ作成

Proxmox CSI PluginはProxmox VEがProxmoxクラスタに参加していることが前提となっているため、Proxmoxクラスタを作成する。

Datacenter > pve1 > Clusterから作成する。

クラスタ名: `region1`

### トークン作成

[この手順](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/install.md)を参考にして、Proxmoxのロール、ユーザー、トークンを作成する。(Proxmoxのホスト上で実行)

ロール作成

```shell
pveum role add CSI -privs "VM.Audit VM.Config.Disk Datastore.Allocate Datastore.AllocateSpace Datastore.Audit"
```

ユーザーを作成してロールを紐づける。

```shell
pveum user add kubernetes-csi@pve
pveum aclmod / -user kubernetes-csi@pve -role CSI
```

トークンを生成

```shell
pveum user token add kubernetes-csi@pve kaas -privsep 0
```

こんな感じの出力が得られるので、full-tokenidとvalueをメモする。

```shell
┌──────────────┬──────────────────────────────────────┐
│ key          │ value                                │
╞══════════════╪══════════════════════════════════════╡
│ full-tokenid │ kubernetes-csi@pve!kaas              │
├──────────────┼──────────────────────────────────────┤
│ info         │ {"privsep":"0"}                      │
├──────────────┼──────────────────────────────────────┤
│ value        │ xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx │
└──────────────┴──────────────────────────────────────┘
```

### Cluster APIのClusterResourceSetを使ってProxmox CSI Pluginの設定を各クラスタに行う

ClusterResourceSetはSecretやConfigMapに書かれたyamlをK8sにデプロイしてくれるもの。

Proxmox CSI Pluginの設定

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: csi-proxmox-config
  namespace: kaas
type: addons.cluster.x-k8s.io/resource-set
stringData:
  config.yaml: |
    apiVersion: v1
    kind: Secret
    metadata:
      name: proxmox-csi-plugin
      namespace: csi-proxmox
    stringData:
      config.yaml: |
        clusters:
          - url: https://192.168.10.138:8006/api2/json
            insecure: true
            token_id: "kubernetes-csi@pve!kaas"
            token_secret: "xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"
            # ProxmoxのClusterを指定する
            region: region1
```

StorageClassを作成する。
指定可能なパラメーターは[ドキュメント](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/options.md)を見る。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: storage-class-proxmox
  namespace: kaas
data:
  storage-class.yaml: |
    apiVersion: storage.k8s.io/v1
    kind: StorageClass
    metadata:
      name: proxmox-data
      annotations:
        storageclass.kubernetes.io/is-default-class: "true"
    provisioner: csi.proxmox.sinextra.dev
    allowVolumeExpansion: true
    volumeBindingMode: WaitForFirstConsumer
    reclaimPolicy: Delete
    parameters:
      csi.storage.k8s.io/fstype: ext4
      storage: local-lvm
```

Proxmox CSI Pluginインストール用の[マニフェスト](https://github.com/sergelogvinov/proxmox-csi-plugin/blob/main/docs/deploy/proxmox-csi-plugin-release.yml)をConfigMapに記載する。

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: proxmox-csi-plugin
  namespace: kaas
data:
  proxmox-csi-plugin-release.yml: |
    <ここにProxmox CSI Pluginのマニフェストを記載>
```

ClusterResourceSetを作成

```yaml
apiVersion: addons.cluster.x-k8s.io/v1beta1
kind: ClusterResourceSet
metadata:
  name: csi-proxmox
  namespace: kaas
spec:
  strategy: Reconcile
  clusterSelector:
    matchLabels:
      csi-proxmox: "true"
  resources:
    - name: proxmox-csi-plugin
      kind: ConfigMap
    - name: csi-proxmox-config
      kind: Secret
    - name: storage-class-proxmox
      kind: ConfigMap
```

### Clusterにラベルを張る

```yaml
apiVersion: cluster.x-k8s.io/v1beta1
kind: Cluster
metadata:
  name: cluster01
  namespace: kaas
  labels:
    csi-proxmox: "true"
spec:
  ...
```

## やれなかったこと

### クラスタをまたいでIPアドレス管理をしたかった

今のProxmoxClusterは、クラスタごとにInClusterIPPoolを作ってそれをProxmoxMachineから使う仕様なので、変更は難しそう。

### 各ノードに自動でラベルを追加したい

CSI用にトポロジーラベルを張りたかったが、今は手動でやっている。

### RKE2とかTalos Linuxを試したい

Packerがうまく使えず挫折。
