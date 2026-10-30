# ARM Bare Metal Workload Cluster

本项目用于部署一套 **全 ARM64 的 Bare Metal Provider Workload Cluster**。

> 六台节点运行在 Huawei DCS ARM VM 上，但 Kubernetes 的基础设施 Provider 仍然是 **Cluster API Bare Metal Provider**。本项目不使用 `DCSCluster`、`DCSMachineTemplate`、DCS Provider 或 HCS Provider 的 CAPI 资源。

部署链路沿用参考项目 `dcs-provider/test/mandiri-workloadcluster`：

```text
MachineRegistration
  -> SeedImage
  -> ARM VM 从 ISO 启动并安装 Alauda OS
  -> MachineInventory
  -> MachineInventoryPool
  -> BaremetalMachineTemplate
  -> BaremetalCluster / Cluster / KubeadmControlPlane
  -> MachineDeployment
```

当前项目只保留 `cp/`、`worker/` 两个角色目录和本 README，不使用渲染脚本。YAML 中仍有 `<...>` 时，禁止 apply。

本项目按当前 Global 的 ACP 4.4.1 版本准备，OS 镜像已确认为 ARM64。平台 Registry 为 `10.243.166.5:11443`。ACP 4.4.1 OS Support Matrix 的版本为：

| 项目 | 版本 |
|---|---|
| Kubernetes | `v1.35.6-acp.9` |
| CoreDNS | `1.14.7-v4.4.7` |
| etcd | `v3.6.13-acp.2` |
| Kube-OVN chart | `v4.4.12` |
| Bare Metal Provider | `v1.0.0-feat.5.g920c160c5-feat-cni-network-device` |
| DCS Provider | `v1.0.22`（本项目不使用） |
| 另一个已安装的 CAPI/Kubernetes 插件 | `v1.0.14` |

截图中显示的 Bare Metal Provider 特性版本包含 `feat-cni-network-device`，与本项目使用的 `networkDevice` 字段相匹配。DCS Provider 虽然安装在 Global 上，但不属于本 Workload Cluster 的基础设施 Provider。

## 1. 目标拓扑

集群名固定为：

```text
arm-workloadcluster
```

| 角色 | 主机名 | 节点 IP |
|---|---|---|
| Control Plane 1 | `arm-workloadcluster-cp01` | `10.243.166.130` |
| Control Plane 2 | `arm-workloadcluster-cp02` | `10.243.166.131` |
| Control Plane 3 | `arm-workloadcluster-cp03` | `10.243.166.132` |
| Worker 1 | `arm-workloadcluster-worker01` | `10.243.166.133` |
| Worker 2 | `arm-workloadcluster-worker02` | `10.243.166.134` |
| Worker 3 | `arm-workloadcluster-worker03` | `10.243.166.135` |
| API VIP | — | `10.243.166.138:6443` |

`10.243.166.138` 是 Kubernetes API 的 External Load Balancer VIP，不是任何节点地址。

节点 IP 只在本 README 中记录。参考项目使用的 `MachineInventoryPool` API 只承载 Inventory 名、hostname 和 networkDevice，不应向 Pool YAML 添加没有依据的静态 IP 字段。

## 2. 项目结构

```text
baremetal-workloadcluster-arm/
├── README.md
├── cp/
│   ├── 01-registration-seedimage.yaml
│   ├── 02-control-plane-pool.yaml
│   ├── 03-baremetal-cluster.yaml
│   ├── 04-control-plane-machine-template.yaml
│   ├── 05-cluster.yaml
│   └── 06-control-plane.yaml
└── worker/
    ├── 01-registration-seedimage.yaml
    ├── 02-worker-pool.yaml
    ├── 03-worker-machine-template.yaml
    ├── 04-worker-kubeadm-config-template.yaml
    └── 05-worker-machine-deployment.yaml
```

### Control Plane

| 文件 | 作用 |
|---|---|
| [`cp/01-registration-seedimage.yaml`](cp/01-registration-seedimage.yaml) | ARM64 Control Plane Registration 和 SeedImage |
| [`cp/02-control-plane-pool.yaml`](cp/02-control-plane-pool.yaml) | 三台 CP Inventory Pool |
| [`cp/03-baremetal-cluster.yaml`](cp/03-baremetal-cluster.yaml) | BaremetalCluster、API VIP、CNI 网卡 |
| [`cp/04-control-plane-machine-template.yaml`](cp/04-control-plane-machine-template.yaml) | CP BaremetalMachineTemplate |
| [`cp/05-cluster.yaml`](cp/05-cluster.yaml) | CAPI Cluster |
| [`cp/06-control-plane.yaml`](cp/06-control-plane.yaml) | 三副本 KubeadmControlPlane |

### Worker

| 文件 | 作用 |
|---|---|
| [`worker/01-registration-seedimage.yaml`](worker/01-registration-seedimage.yaml) | ARM64 Worker Registration 和独立 SeedImage |
| [`worker/02-worker-pool.yaml`](worker/02-worker-pool.yaml) | 三台 Worker Inventory Pool |
| [`worker/03-worker-machine-template.yaml`](worker/03-worker-machine-template.yaml) | Worker BaremetalMachineTemplate |
| [`worker/04-worker-kubeadm-config-template.yaml`](worker/04-worker-kubeadm-config-template.yaml) | Worker kubeadm bootstrap |
| [`worker/05-worker-machine-deployment.yaml`](worker/05-worker-machine-deployment.yaml) | 三副本 Worker MachineDeployment |

CP 和 Worker 必须使用不同的 Registration 和 SeedImage 资源；两者可以使用同一个已确认的 ARM64 `kubeos-iso:baremetal` 作为 `baseImage`。不要混用生成后的 ISO 文件或 Registration 配置。

## 3. 必须先确认的现场值

以下现场值没有从 Mandiri 环境复制，必须在目标 Global / ARM VM 环境中确认后替换；ACP 4.4.1 的 Kubernetes、CoreDNS、etcd 和 Kube-OVN 版本已按上面的版本矩阵固定：

```text
<MANAGEMENT_NAMESPACE>
<KUBEADM_IMAGE_REPOSITORY>
<SSH_AUTHORIZED_KEY>
<CP_INVENTORY_1/2/3>
<WORKER_INVENTORY_1/2/3>
<CP_NODE_1/2/3_NETWORK_DEVICE>
<WORKER_NODE_1/2/3_NETWORK_DEVICE>
<CLUSTER_DEFAULT_NETWORK_DEVICE>
```

### 3.1 复用到其他集群时的修改清单

本目录可以作为其他 ARM Bare Metal Workload Cluster 的模板，但不能只修改集群名后直接 apply。至少需要按照下面的清单检查并修改：

| 修改项 | 需要修改的位置 | 当前项目值/说明 |
|---|---|---|
| 集群名称 | 所有 YAML 的 `metadata.name`、`spec.clusterName`、引用字段、标签和 hostname | 当前为 `arm-workloadcluster` |
| 管理命名空间 | 所有 YAML 的 `metadata.namespace`、引用对象的 `namespace` | 当前为 `<MANAGEMENT_NAMESPACE>`，通常是 `cpaas-system` |
| ACP/Kubernetes 版本 | `cp/06-control-plane.yaml`、`worker/05-worker-machine-deployment.yaml`、catalog key | 当前为 `v1.35.6-acp.9`，换 ACP 版本时必须重新查 OS Support Matrix |
| CoreDNS / etcd / Kube-OVN 版本 | `cp/06-control-plane.yaml`、`cp/05-cluster.yaml` | 当前分别为 `1.14.7-v4.4.7`、`v3.6.13-acp.2`、`v4.4.12` |
| 平台 Registry | `cp/05-cluster.yaml` 的 `cpaas.io/registry-address`、ISO/rootfs 引用、catalog | 当前为 `10.243.166.5:11443` |
| ARM64 ISO | `cp/01-registration-seedimage.yaml`、`worker/01-registration-seedimage.yaml` 的 `spec.baseImage` | 当前为 `tkestack/kubeos-iso:baremetal` |
| ARM64 rootfs | `elemental-image-catalog`，不是 SeedImage YAML | 当前为 `tkestack/kubeos-rootfs:baremetal`，并映射到目标 Kubernetes version key |
| Control Plane Registration | `cp/01-registration-seedimage.yaml` 的 Registration、SeedImage 名称和 `registrationRef` | 必须使用新集群独立的资源名 |
| Worker Registration | `worker/01-registration-seedimage.yaml` 的 Registration、SeedImage 名称和 `registrationRef` | 必须与 CP Registration 分开 |
| API VIP 和端口 | `cp/03-baremetal-cluster.yaml`、LB 配置和 README 拓扑 | 当前为 `10.243.166.138:6443` |
| LB 后端 | 外部 LB 配置，不在 CAPI YAML 中自动创建 | 只转发到 3 台 CP，不要加入 Worker |
| Pod / Service / Join CIDR | `cp/05-cluster.yaml` | 当前分别为 Pod `100.150.0.0/16`、Service `100.151.0.0/16`、Join `100.68.0.0/16`；复用到其他集群时必须重新确认不冲突 |
| CP/Worker 节点数量 | CP/Worker Pool 成员、`cp/06-control-plane.yaml`、`worker/05-worker-machine-deployment.yaml` | 当前为 3 CP + 3 Worker；改变数量时同步修改 Pool 和 replicas |
| MachineInventory 名称 | `cp/02-control-plane-pool.yaml`、`worker/02-worker-pool.yaml` | 必须使用新 ISO 注册后实际发现的 Inventory 名称 |
| hostname | 两个 Pool 文件及 README | 必须与目标节点规划一致，不能重复 |
| `networkDevice` | 两个 Pool 文件和 `cp/03-baremetal-cluster.yaml` | 使用 Inventory 的实际网卡名，不要直接复制 `eth0` 或旧环境值 |
| 节点 IP / 网关 / DNS / VLAN / MTU | VM 网络配置和 README | 不应虚构写入 `MachineInventoryPool`；按现场网络表配置 |
| SSH 公钥 | `cp/06-control-plane.yaml`、`worker/04-worker-kubeadm-config-template.yaml` | 替换为目标环境公钥；私钥不能进入项目 |
| kubeadm 镜像仓库 | `cp/06-control-plane.yaml` 的 `imageRepository` | 按目标 ACP 版本和产品文档确认，不要默认沿用其他环境 |
| TPM / 安装盘 | 两个 Registration 的 Elemental 配置及 VM 安装流程 | 根据每台 VM 是否有 TPM、实际系统盘分别确认 |

如果只创建 1 台 CP 或 1 台 Worker，不能只把 `replicas` 改成 `1`，还必须同步修改对应的 `MachineInventoryPool` 成员数量、LB 后端和高可用规划。

以下内容通常保持不变，除非目标产品版本或网络方案不同：

- `targetPlatform: linux/arm64`；
- `device: /dev/elemental-install-target`；
- `cluster-type: ProviderBaremetal`；
- `networkType: kube-ovn`；
- CP 使用 `KubeadmControlPlane`，Worker 使用 `MachineDeployment`；
- Bare Metal Provider 资源链：`MachineInventoryPool -> BaremetalMachineTemplate -> CAPI Machine`。

还需要确认：

- Bare Metal Provider、Kubeadm Provider 及对应 CRD 已 Ready；
- 平台 Registry 地址为 `10.243.166.5:11443`；
- ARM64 `kubeos-iso:baremetal` 已推入平台 Registry；
- ARM64 `kubeos-rootfs:baremetal` 已推入平台 Registry；
- `elemental-image-catalog` 已存在，并且 `v1.35.6-acp.9` key 指向 ARM64 rootfs；
- 当前 Pod CIDR 为 `100.150.0.0/16`、Service CIDR 为 `100.151.0.0/16`、Kube-OVN Join CIDR 为 `100.68.0.0/16`，且不与 Global、其他 Workload 或物理网络冲突；
- 六台 ARM VM 的实际网卡名、VLAN/bond、前缀、网关、DNS 和 MTU；
- DCS 控制台能够为 ARM VM 挂载并启动生成的 ARM64 ISO；
- VM 的安装盘可以在 Live ISO 内被准确识别；
- API VIP 已配置 External LB；
- 同一把 SSH 公钥会写入 KCP 和 Worker KubeadmConfigTemplate。

检查占位符：

```bash
grep -RInE '<[A-Z0-9_]+>' README.md cp worker
```

`<...>` 出现在某个待 apply 的 YAML 中时，不要继续。Elemental 运行时表达式 `${System Information/UUID}` 和 `${System Information/Serial Number}` 是合法内容，不要替换。

## 4. ARM 镜像关系

两类镜像用途不同：

| 用途 | 使用镜像 |
|---|---|
| Live ISO 启动和安装 | SeedImage 的 ARM64 `kubeos-iso` |
| CAPI Machine reprovision / join | `elemental-image-catalog` 中对应版本的 ARM64 `kubeos-rootfs` |

不要把 ISO 写进 catalog，也不要把 rootfs 写成 SeedImage 的 `baseImage`。

本集群六台节点全部为 ARM64，不需要 Mandiri 文档中 x86 Master 与 ARM Worker 之间切换 catalog 的步骤。正确顺序是：在创建任何 CAPI Machine 前，确认 `v1.35.6-acp.9` catalog key 指向 ARM64 rootfs。

示意检查：

```bash
kubectl -n <MANAGEMENT_NAMESPACE> get configmap elemental-image-catalog -o yaml
```

如果 ConfigMap 不存在，先确认 Bare Metal Provider 是否安装完成；不要自行创建一个看似正确但不符合 Provider 版本的 ConfigMap。

## 5. External Load Balancer

API VIP 必须提供 TCP passthrough：

```text
10.243.166.138:6443
  -> 10.243.166.130:6443
  -> 10.243.166.131:6443
  -> 10.243.166.132:6443
```

要求：

- 四层 TCP 转发；
- 不在 LB 上终止 TLS；
- 不把 Worker 加入 API LB；
- VIP 不配置到任何节点；
- Global 到 VIP、VIP 到三台 CP 的路由和安全策略已放通。

在 apply `cp/03-baremetal-cluster.yaml` 和 `cp/06-control-plane.yaml` 前，VIP 必须已经准备好。

## 6. Control Plane 部署顺序

所有命令默认在已连接目标 Global 的管理节点上执行。一次只 apply 一个 YAML，不要执行 `kubectl apply -f cp/` 一把梭。

### 6.1 创建 ARM64 CP Registration / SeedImage

确认 `cp/01-registration-seedimage.yaml` 中的：

- `<MANAGEMENT_NAMESPACE>`；
- ARM64 ISO 已固定为 `10.243.166.5:11443/tkestack/kubeos-iso:baremetal`；
- 必要的 COS_STATE 容量和 TPM 设置。

执行：

```bash
kubectl apply --dry-run=server -f cp/01-registration-seedimage.yaml
kubectl apply -f cp/01-registration-seedimage.yaml
kubectl -n <MANAGEMENT_NAMESPACE> describe seedimage arm-workloadcluster-cp-registration-iso
```

必须先看到 SeedImage 已生成并具有可下载地址，再给三台 CP VM 挂载 ISO。

### 6.2 在三台 ARM CP VM 上安装

每台 VM 都单独确认系统盘：

```bash
lsblk -d -e7 -o NAME,PATH,SIZE,MODEL,SERIAL,WWN,HCTL
ln -s <CONFIRMED_OS_DISK> /dev/elemental-install-target
readlink -f /dev/elemental-install-target
```

注意：

- `<CONFIRMED_OS_DISK>` 只能根据该台 VM 的 `lsblk` 结果填写；
- 不要把 `/dev/sda`、`/dev/vda` 或 `/dev/nvme0n1` 写入 YAML；
- 不要选择 `sr*`、`loop*` 或数据盘；
- 安装触发重启后，必须弹出 ISO，并把启动顺序改为 disk-first；
- 加入 Pool 前必须确认根文件系统不是 Live ISO。

### 6.3 发现 CP Inventory

```bash
kubectl -n <MANAGEMENT_NAMESPACE> \
  get machineinventories.elemental.cattle.io -o wide
```

三台 CP Inventory 必须满足：

- `Ready=True`；
- allocation 为空或显示可用；
- owner 为空；
- 已从安装后的磁盘启动；
- 架构为 ARM64；
- `observedNetwork` 中的网卡名已记录。

把实际 Inventory 名和实际网卡名填入 `cp/02-control-plane-pool.yaml`。

### 6.4 创建 CP CAPI 资源

按以下顺序执行，每一步先 dry-run：

```bash
kubectl apply --dry-run=server -f cp/02-control-plane-pool.yaml
kubectl apply -f cp/02-control-plane-pool.yaml

kubectl apply --dry-run=server -f cp/03-baremetal-cluster.yaml
kubectl apply -f cp/03-baremetal-cluster.yaml

kubectl apply --dry-run=server -f cp/04-control-plane-machine-template.yaml
kubectl apply -f cp/04-control-plane-machine-template.yaml

kubectl apply --dry-run=server -f cp/05-cluster.yaml
kubectl apply -f cp/05-cluster.yaml

kubectl apply --dry-run=server -f cp/06-control-plane.yaml
kubectl apply -f cp/06-control-plane.yaml
```

控制面资源检查：

```bash
kubectl -n <MANAGEMENT_NAMESPACE> \
  get baremetalcluster,cluster,kubeadmcontrolplane,machine,baremetalmachine
```

必须等到：

- BaremetalCluster Ready；
- KCP replicas 为 3/3；
- 三台 CP Machine/BaremetalMachine 正常；
- 三台 ARM64 CP Node Ready；
- API 可通过 `10.243.166.138:6443` 访问。

在控制面未 Ready 前，不要创建 Worker MachineDeployment。

## 7. Worker 部署顺序

### 7.1 创建独立 ARM64 Worker Registration / SeedImage

确认 `worker/01-registration-seedimage.yaml` 中的：

- `<MANAGEMENT_NAMESPACE>`；
- ARM64 ISO 与 CP 使用同一个已确认的 `10.243.166.5:11443/tkestack/kubeos-iso:baremetal`；
- 必要的 COS_STATE 容量和 TPM 设置。

```bash
kubectl apply --dry-run=server -f worker/01-registration-seedimage.yaml
kubectl apply -f worker/01-registration-seedimage.yaml
kubectl -n <MANAGEMENT_NAMESPACE> describe seedimage arm-workloadcluster-worker-registration-iso
```

不要使用 CP 的 ISO。

### 7.2 安装三台 ARM Worker

Worker 使用自己的 ARM64 ISO。每台 VM 仍然单独执行认盘、建立 `/dev/elemental-install-target`、安装、弹出 ISO、disk-first 启动。

安装后再配置并持久化节点 IP：

```text
worker01 -> 10.243.166.133
worker02 -> 10.243.166.134
worker03 -> 10.243.166.135
```

具体接口、前缀、网关、DNS、VLAN/bond 配置必须以现场网络表和 VM 实际网卡为准，不能直接复制 Mandiri 的 `eth0` 或 `eth0.329`。

### 7.3 发现 Worker Inventory 并创建 Pool

```bash
kubectl -n <MANAGEMENT_NAMESPACE> \
  get machineinventories.elemental.cattle.io -o wide
```

将三台 Worker 的实际 Inventory 名和实际网卡名填入 `worker/02-worker-pool.yaml`，确认三台都是 `Ready=True` 且可用。

### 7.4 创建 Worker CAPI 资源

```bash
kubectl apply --dry-run=server -f worker/02-worker-pool.yaml
kubectl apply -f worker/02-worker-pool.yaml

kubectl apply --dry-run=server -f worker/03-worker-machine-template.yaml
kubectl apply -f worker/03-worker-machine-template.yaml

kubectl apply --dry-run=server -f worker/04-worker-kubeadm-config-template.yaml
kubectl apply -f worker/04-worker-kubeadm-config-template.yaml

kubectl apply --dry-run=server -f worker/05-worker-machine-deployment.yaml
kubectl apply -f worker/05-worker-machine-deployment.yaml
```

Worker 检查：

```bash
kubectl -n <MANAGEMENT_NAMESPACE> \
  get machinedeployment,machine,baremetalmachine
```

最终要求：

- MachineDeployment READY 为 3/3；
- 3 台 Worker Machine/BaremetalMachine 正常；
- 6 台节点全部 Ready；
- 所有节点报告架构为 `arm64`；
- Worker 带有 `kube-ovn/role=worker`。

## 8. 最终验收

```bash
kubectl --kubeconfig <WORKLOAD_KUBECONFIG> \
  get nodes -o custom-columns=NAME:.metadata.name,ARCH:.status.nodeInfo.architecture,IP:.status.addresses
```

验证项目：

- CP 节点名称和 Worker 节点名称符合本 README；
- 六个节点的 `ARCH` 都是 `arm64`；
- CP IP 为 `.130-.132`，Worker IP 为 `.133-.135`；
- API endpoint 使用 `10.243.166.138:6443`；
- CP 标签为 `kube-ovn/role=master`；
- Worker 标签为 `kube-ovn/role=worker`；
- kube-ovn、CoreDNS、etcd 和系统 Pod 正常；
- Pod-to-Pod、Pod-to-Service、Node-to-Node 和 API 访问正常；
- 没有 `ImageCatalogMiss`、Inventory 未就绪、错误网卡或重复 IP。

## 9. 常见故障边界

- **SeedImage 没有可下载地址**：先检查 ARM ISO、默认 StorageClass 和 Bare Metal Provider，不要继续挂 ISO。
- **Inventory 未 Ready**：检查 VM 是否仍从 Live ISO 启动、网卡/网络是否可达、时间同步和 TPM 设置。
- **反复启动 ISO**：弹出虚拟介质并将 BMC/DCS 启动顺序改为 disk-first。
- **ImageCatalogMiss**：确认 catalog 的目标版本 key 指向 ARM64 rootfs，而不是 ISO，并且 key 与 Machine 的 Kubernetes version 完全一致。
- **Pool 不可用**：检查 Inventory 名、owner、Ready 状态和 `observedNetwork`，不要猜 Inventory 名。
- **API 不通**：检查 `10.243.166.138:6443` 是否 TCP 透传到三台 CP，Worker 不应进入 API LB。
- **CIDR 冲突**：停止部署，重新选择未被 Global、其他 Workload 或物理网络使用的 Pod/Service/Join CIDR。

## 10. 安全注意事项

密码、Registry 凭据、kubeconfig、私钥、Registration URL 和临时令牌不得写入本项目。SSH 公钥可以写入 KCP/Worker 清单，但私钥不能进入仓库。
