# ARM Bare Metal Workload Cluster

本项目用于部署一套 **全 ARM64 的 Bare Metal Provider Workload Cluster**。

> 六台节点运行在 Huawei DCS ARM VM 上，但 Kubernetes 的基础设施 Provider 仍然是 **Cluster API Bare Metal Provider**。本项目不使用 `DCSCluster`、`DCSMachineTemplate`、DCS Provider 或 HCS Provider 的 CAPI 资源。

部署链路沿用参考项目 `dcs-provider/test/mandiri-workloadcluster`：

```text
MachineRegistration
  -> SeedImage
  -> ARM VM 从 ISO 启动并安装 alaudaos.k.25
  -> MachineInventory
  -> MachineInventoryPool
  -> BaremetalMachineTemplate
  -> BaremetalCluster / Cluster / KubeadmControlPlane
  -> MachineDeployment
```

当前项目只保留 `cp/`、`worker/` 两个角色目录和本 README，不使用渲染脚本。YAML 中仍有 `<...>` 时，禁止 apply。

本项目按当前 Global 的 ACP 4.4.1 版本准备，目标 OS 为 **`alaudaos.k.25`**。平台 Registry 为 `10.243.166.5:11443`。Kubernetes、catalog 和 runtime 使用 `arm64`；交付镜像 tag 以实际 K25 ARM64 归档为准，不要猜 preview tag。ACP 4.4.1 OS Support Matrix 的版本为：

| 项目 | 版本 |
|---|---|
| Kubernetes | `v1.35.6-acp.9` |
| CoreDNS | `1.14.7-v4.4.7` |
| etcd | `v3.6.13-acp.2` |
| Kube-OVN chart | `v4.4.12` |
| OS ID | `alaudaos.k.25` |
| Catalog key | `v1.35.6-acp.9_arm64_alaudaos.k.25` |
| Bare Metal Provider | `v1.0.0-feat.5.g920c160c5-feat-cni-network-device` |
| DCS Provider | `v1.0.22`（本项目不使用） |
| 另一个已安装的 CAPI/Kubernetes 插件 | `v1.0.14` |

截图中显示的 Bare Metal Provider 特性版本包含 `feat-cni-network-device`，与本项目使用的 `networkDevice` 字段相匹配。该 build 与 K25 预览文档验证过的 provider 包不同：CRD 能接受 `system-uri` 和 OS 标签，不等于当前 provider 已完整支持 K25 A/B 安装。DCS Provider 虽然安装在 Global 上，但不属于本 Workload Cluster 的基础设施 Provider。

K25 使用两个约 20000 MiB 的只读 A/B 根分区，剩余空间作为 `/persist`。`/usr` 只读；`/var`、`/opt`、`/etc`、`/home`、`/root` 通过 `/persist` 可写。`SeedImage.spec.size: 20Gi` 是构建 ISO 的存储请求，不是目标系统盘容量。系统盘建议至少 100 GiB。不要配置旧的 `COS_STATE` 或 `btrfs snapshotter`。

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

CP 和 Worker 必须使用不同的 Registration 和 SeedImage 资源；两者可以使用同一个已确认的 K25 ARM64 ISO/rootfs tag，但不能混用生成后的 ISO 文件或 Registration 配置。最终 Registry 不要继续使用 `:baremetal` 作为 K25 交付 tag。

同一个 Global 中可以同时存在多个 SeedImage 和 MachineRegistration，包括不同集群、不同架构和不同角色的资源。每套资源必须使用唯一名称，并通过独立的 `registrationRef`、Inventory Pool 和 CAPI 模板隔离。旧的 x86 SeedImage 不要直接用于 ARM 节点；ARM SeedImage 的 `targetPlatform` 必须是 `linux/arm64`。

本项目的所有 Kubernetes 资源已经使用管理命名空间 `cpaas-system`。如果复用到其他 Global 或其他管理命名空间，需要在所有 YAML 的 `metadata.namespace`、`registrationRef.namespace` 和模板引用 namespace 中统一替换，不能只修改一处。

## 3. 必须先确认的现场值

以下现场值没有从 Mandiri 环境复制，必须在目标 Global / ARM VM 环境中确认后替换；ACP 4.4.1 的 Kubernetes、CoreDNS、etcd 和 Kube-OVN 版本已按上面的版本矩阵固定：

```text
<K25_ARM64_IMAGE_TAG>
<KUBEADM_IMAGE_REPOSITORY>
<SSH_AUTHORIZED_KEY>
<CP_INVENTORY_1/2/3>
<WORKER_INVENTORY_1/2/3>
<CP_NODE_1/2/3_NETWORK_DEVICE>
<WORKER_NODE_1/2/3_NETWORK_DEVICE>
<CLUSTER_DEFAULT_NETWORK_DEVICE>
```

`<K25_ARM64_IMAGE_TAG>` 必须从实际 ACP 4.4.1 K25 ARM64 交付包或 Registry 导入结果填写。ISO 和 rootfs 使用同一个 tag。不要使用新页面中的 ACP 4.4.0 preview tag。

### 3.1 复用到其他集群时的修改清单

本目录可以作为其他 ARM Bare Metal Workload Cluster 的模板，但不能只修改集群名后直接 apply。至少需要按照下面的清单检查并修改：

| 修改项 | 需要修改的位置 | 当前项目值/说明 |
|---|---|---|
| 集群名称 | 所有 YAML 的 `metadata.name`、`spec.clusterName`、引用字段、标签和 hostname | 当前为 `arm-workloadcluster` |
| 管理命名空间 | 所有 YAML 的 `metadata.namespace`、引用对象的 `namespace` | 当前为 `cpaas-system`；复用时按目标 Global 实际 namespace 统一修改 |
| ACP/Kubernetes 版本 | `cp/06-control-plane.yaml`、`worker/05-worker-machine-deployment.yaml`、catalog key | 当前为 `v1.35.6-acp.9`，换 ACP 版本时必须重新查 OS Support Matrix |
| CoreDNS / etcd / Kube-OVN 版本 | `cp/06-control-plane.yaml`、`cp/05-cluster.yaml` | 当前分别为 `1.14.7-v4.4.7`、`v3.6.13-acp.2`、`v4.4.12` |
| 平台 Registry | `cp/05-cluster.yaml` 的 `cpaas.io/registry-address`、ISO/rootfs 引用、catalog | 当前为 `10.243.166.5:11443` |
| ARM64 ISO | `cp/01-registration-seedimage.yaml`、`worker/01-registration-seedimage.yaml` 的 `spec.baseImage` | `tkestack/kubeos-iso:<K25_ARM64_IMAGE_TAG>` |
| ARM64 rootfs | Registration `install.system-uri` 和 `elemental-image-catalog` | `tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>`，catalog key 为 `v1.35.6-acp.9_arm64_alaudaos.k.25` |
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
| TPM / 安装盘 | 两个 Registration 的 Elemental 配置及 VM 安装流程 | 根据每台 VM 是否有 TPM、实际系统盘分别确认；保持 `/dev/elemental-install-target` |
| K25 OS 标签 | 两个 Registration 的 `machineInventoryLabels` | `os.baremetal.alauda.io/id=alaudaos.k.25`，架构使用 `${Runtime/Architecture}` |

### 3.2 Mandiri 参考项目哪些可以复用

以下内容可以从 `mandiri-workloadcluster` 参考项目照抄或保持同样结构，因为它们描述的是 ACP Bare Metal Provider 的资源关系，而不是 Mandiri 主机的现场值：

- `MachineRegistration -> SeedImage -> MachineInventory -> MachineInventoryPool` 的资源关系；
- `BaremetalCluster -> Cluster -> KubeadmControlPlane` 的控制面引用关系；
- `MachineDeployment -> KubeadmConfigTemplate + BaremetalMachineTemplate` 的 Worker 引用关系；
- `apiVersion`、`kind`、`spec` 的整体层级，以及资源创建顺序；
- `targetPlatform: linux/arm64`、`device: /dev/elemental-install-target` 等 ARM/Elemental 约束；
- `networkType: kube-ovn`、`cluster-type: ProviderBaremetal`；
- KCP 和 MachineDeployment 的 `maxSurge: 0`、`maxUnavailable: 1` 策略；
- KCP 的审计、TLS、kubelet、PodSecurity 和 kubeadm 配置结构；
- Worker 的 `kube-ovn/role=worker` 标签；
- server-side dry-run、Inventory Ready 检查、安装后弹出 ISO 并 disk-first 启动的操作流程。

以下内容不能直接照抄 Mandiri，必须按目标环境修改：

- 任何 `metadata.name`、`clusterName`、hostname、Inventory 名称和 Registration/SeedImage 名称；
- namespace、Registry、ISO、rootfs、catalog key 和所有 ACP/Kubernetes 版本；
- API VIP、LB 后端、端口、Pod/Service/Join CIDR；
- `networkDevice`、节点 IP、网关、DNS、VLAN、bond 和 MTU；
- SSH 公钥、kubeadm image repository、TPM 设置和安装盘；
- CP/Worker 节点数量及对应 Pool 成员。

特别注意：即使两个环境都使用 Bare Metal Provider，也不能直接复制 Mandiri 的 Inventory 名称、网卡名、IP、CIDR、Registry、版本或镜像 tag。资源结构可以复用，现场值必须重新确认。

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
- 配对的 K25 ARM64 `kubeos-iso:<K25_ARM64_IMAGE_TAG>` 已推入平台 Registry；
- 配对的 K25 ARM64 `kubeos-rootfs:<K25_ARM64_IMAGE_TAG>` 已推入平台 Registry；
- `elemental-image-catalog` 已存在，并且 `v1.35.6-acp.9_arm64_alaudaos.k.25` 指向该 ARM64 rootfs；
- 六台 ARM VM 使用 UEFI，能读取 SMBIOS/DMI，系统盘建议至少 100 GiB；
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

`<...>` 出现在某个待 apply 的 YAML 中时，不要继续。Elemental 运行时表达式 `${System Information/UUID}`、`${System Information/Serial Number}` 和 `${Runtime/Architecture}` 是合法内容，不要替换。

## 4. ARM 镜像关系

三类引用不能互换：

| 字段 | 镜像 | 用途 |
|---|---|---|
| `SeedImage.spec.baseImage` | `kubeos-iso:<K25_ARM64_IMAGE_TAG>` | 构建 Live ISO |
| `MachineRegistration.spec.config.elemental.install.system-uri` | `kubeos-rootfs:<K25_ARM64_IMAGE_TAG>` | Live ISO 首次写入 K25 系统 |
| `elemental-image-catalog` | `kubeos-rootfs:<K25_ARM64_IMAGE_TAG>` | CAPI reprovision / upgrade |

不要把 ISO 写进 catalog，也不要把 rootfs 写成 SeedImage 的 `baseImage`。ISO 和 rootfs 必须来自同一 K25 ARM64 交付包。

本集群六台节点全部为 ARM64，不需要 Mandiri 文档中 x86 Master 与 ARM Worker 之间切换 catalog 的步骤。正确顺序是：在创建任何 CAPI Machine 前，确认三段式 catalog key 存在：

```text
v1.35.6-acp.9_arm64_alaudaos.k.25
```

先检查现有 ConfigMap：

```bash
kubectl -n cpaas-system get configmap elemental-image-catalog -o yaml
```

如果对象存在但缺少该 key，只做 merge patch，不要 replace 整个 ConfigMap：

```bash
kubectl -n cpaas-system patch configmap elemental-image-catalog \
  --type merge \
  -p '{"data":{"v1.35.6-acp.9_arm64_alaudaos.k.25":"10.243.166.5:11443/tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>"}}'
```

然后精确读取：

```bash
kubectl -n cpaas-system get configmap elemental-image-catalog \
  -o jsonpath='{.data.v1\.35\.6-acp\.9_arm64_alaudaos\.k\.25}{"\n"}'
```

如果 ConfigMap 不存在，先确认 Bare Metal Provider 是否安装完成；不要自行创建一个看似正确但不符合 Provider 版本的 ConfigMap。不要删除其他版本、架构或 OS 的 catalog 条目。

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

### 6.1 CP-01：创建 Control Plane Registration 和 SeedImage

文件：

```text
cp/01-registration-seedimage.yaml
```

这个文件包含两个资源：

1. `MachineRegistration`：定义 CP 节点注册后的名称、Inventory 标签、安装盘和 TPM 行为；
2. `SeedImage`：基于 ARM64 ISO 生成 CP 注册启动 ISO。

首次部署 `arm-workloadcluster` 时，确认以下内容：

| 字段 | 当前值/要求 |
|---|---|
| 所有 namespace | `cpaas-system` |
| `MachineRegistration.metadata.name` | `arm-workloadcluster-cp-registration`，必须唯一 |
| `SeedImage.metadata.name` | `arm-workloadcluster-cp-registration-iso`，必须唯一 |
| `spec.baseImage` | `10.243.166.5:11443/tkestack/kubeos-iso:<K25_ARM64_IMAGE_TAG>` |
| `install.system-uri` | `10.243.166.5:11443/tkestack/kubeos-rootfs:<K25_ARM64_IMAGE_TAG>` |
| `targetPlatform` | `linux/arm64` |
| 安装设备 | `/dev/elemental-install-target` |
| OS 标签 | `os.baremetal.alauda.io/id=alaudaos.k.25` |
| `machineName` | 保留 `${System Information/UUID}`，不要手工替换 |
| `emulate-tpm` | 根据目标 VM 是否有 `/dev/tpm0` 确定 |
| `cleanupAfterMinutes` | `240`；下载 URL 会过期，不要提交到 Git |

复用到其他集群时，只修改目标环境对应的名称、namespace、Registry/ISO/rootfs tag 和 TPM；不要修改 Elemental 表达式：

```text
${System Information/UUID}
${System Information/Serial Number}
${Runtime/Architecture}
```

先执行 server-side dry-run，再 apply：

```bash
kubectl apply --dry-run=server -f cp/01-registration-seedimage.yaml
kubectl apply -f cp/01-registration-seedimage.yaml
kubectl -n cpaas-system describe seedimage arm-workloadcluster-cp-registration-iso
```

`describe` 只是查看状态。实际挂载 ISO 时，需要从 SeedImage 的 status 取出下载地址和校验地址：

```bash
kubectl -n cpaas-system get seedimage arm-workloadcluster-cp-registration-iso \
  -o jsonpath='{.status.downloadURL}{"\\n"}'

kubectl -n cpaas-system get seedimage arm-workloadcluster-cp-registration-iso \
  -o jsonpath='{.status.checksumURL}{"\\n"}'
```

也可以一次查看完整 status：

```bash
kubectl -n cpaas-system get seedimage arm-workloadcluster-cp-registration-iso \
  -o jsonpath='{.status}{"\\n"}'
```

将输出的 `downloadURL` 用于下载或在 DCS 虚拟介质界面挂载生成的 ISO；不要把 `spec.baseImage` 的 Registry 镜像地址误当成生成后的 SeedImage 下载地址。

也可以列出当前 SeedImage 状态：

```bash
kubectl -n cpaas-system get seedimage \
  -o custom-columns='NAME:.metadata.name,READY:.status.conditions[?(@.type=="SeedImageReady")].status,REASON:.status.conditions[?(@.type=="SeedImageReady")].reason,URL:.status.downloadURL'
```

继续下一步前必须确认：

- `SeedImageReady=True`；
- reason 表示构建成功，而不是过期或清理；
- `status.downloadURL` 非空；
- `status.checksumURL` 非空；
- 生成的是 ARM64 ISO；
- 下载地址尚未因 `cleanupAfterMinutes: 240` 被清理；
- 还没有把同一批 CP Inventory 分配给其他 Pool。

### 6.2 CP-02：使用生成的 ISO 启动三台 CP VM，并指定安装盘

对 `cp01`、`cp02`、`cp03` 分别执行以下流程。每台 VM 都必须单独确认磁盘，不能因为三台 VM 的磁盘名称看起来相同就直接复制。

#### 6.2.1 挂载和启动 SeedImage ISO

在 DCS VM 控制台中：

1. 将 `arm-workloadcluster-cp-registration-iso` 的 `status.downloadURL` 对应 ISO 挂载到 VM 的虚拟光驱；
2. 设置该 VM 从虚拟光驱启动；
3. 启动后进入 ARM64 Live OS shell；
4. 确认当前确实是该 CP Registration 生成的 ISO。

#### 6.2.2 找到真实系统盘

通用 SeedImage 不应该把 `/dev/sda`、`/dev/vda` 或 `/dev/nvme0n1` 写死，因为不同 VM 的设备枚举顺序可能不同。进入 Live OS 后执行：

```bash
lsblk -d -e7 -o NAME,PATH,TYPE,SIZE,MODEL,SERIAL,WWN,HCTL
ls -l /dev/disk/by-id/
findmnt
```

根据容量、MODEL、SERIAL/WWN 以及 DCS 控制台的磁盘配置，确认哪一个是该 VM 的系统盘。不要选择：

- `sr*` 或虚拟光驱；
- `loop*`；
- Live ISO 对应设备；
- 已知的数据盘；
- 仅凭设备名排序猜出的磁盘。

#### 6.2.3 在安装前准备磁盘和网络

这里不是“创建软链接后立即重启”这么简单。产品流程是：SeedImage 启动后先运行注册流程，Elemental Operator 创建 `MachineInventory`，随后 Elemental Installer 才会把 OS 写入目标磁盘。软链接只是让 `install.device` 找到目标盘，不能代替注册、安装、磁盘标签检查和安装完成确认。

先确认目标盘上没有残留的 Elemental 分区标签：

```bash
lsblk -o NAME,PATH,TYPE,SIZE,FSTYPE,LABEL,PARTLABEL,MOUNTPOINTS
blkid
```

重点检查目标盘及其他磁盘上是否存在旧安装残留的：

```text
COS_STATE
COS_PERSISTENT
COS_OEM
COS_RECOVERY
```

这些是旧 Elemental 安装遗留标签，不是 K25 要创建的分区。K25 安装后应看到 A/B 根分区和 `/persist`，不要期待出现 `COS_STATE`。如果发现旧标签，先停止操作，确认目标盘和数据盘后，按照产品的固定系统盘/磁盘清理流程处理。不要对不确定的设备执行 `wipefs`、`sgdisk --zap-all` 或其他破坏性命令。

Live ISO 中还需确认：

```bash
ls /sys/firmware/dmi/tables/
dmidecode -s system-uuid
dmidecode -s system-serial-number
```

ARM 注册依赖 SMBIOS。UUID 必须在六台节点间唯一。系统必须以 UEFI 启动。

同时确认 Live OS 已经能访问 Global、Registry 和 DNS：

```bash
ip addr
ip route
getent hosts <GLOBAL_HOSTNAME>
```

如果网络未起来，先使用目标 VM 的实际网卡、VLAN、网关和 DNS 配置使网络正常，再继续注册和安装。

#### 6.2.4 创建通用安装目标软链接

优先使用稳定的 `/dev/disk/by-id/` 路径，而不是 `/dev/sdX` 或 `/dev/vdX`。下面的 `<DISK_BY_ID>` 必须替换为刚才确认的实际路径：

```bash
DISK_BY_ID=/dev/disk/by-id/<CONFIRMED_SYSTEM_DISK_ID>
test -b "$DISK_BY_ID"
readlink -f "$DISK_BY_ID"

sudo rm -f /dev/elemental-install-target
sudo ln -s "$DISK_BY_ID" /dev/elemental-install-target

ls -l /dev/elemental-install-target
readlink -f /dev/elemental-install-target
```

如果该 VM 没有可用的 `/dev/disk/by-id/` 稳定链接，才可以临时使用根据 `lsblk` 确认的实际设备路径：

```bash
sudo rm -f /dev/elemental-install-target
sudo ln -s /dev/<CONFIRMED_DEVICE_FROM_LSBLK> /dev/elemental-install-target
readlink -f /dev/elemental-install-target
```

这个实际设备路径只用于当前 Live OS 的软链接，不能写回 Registration YAML，也不能复制到其他 VM。Registration YAML 始终保持：

```yaml
device: /dev/elemental-install-target
```

软链接必须在 Elemental 安装/注册流程开始前建立。确认目标正确后，再让 ISO 继续执行安装；不要在目标磁盘不明确时继续。

软链接不会跨重启持久化，因为 `/dev` 通常是运行时设备树。这里不要求它在安装后的系统中永久存在；它的作用是在当前 Live OS 安装阶段让 Elemental 找到目标盘。安装器必须在软链接存在时完成写盘和 EFI bootloader 安装。

#### 6.2.5 确认注册和安装真正完成

创建软链接后不要立即手工重启或立即卸载 ISO。先让 `elemental-register`、Elemental Operator 和安装流程完成。可以同时从 Global 和 VM 控制台检查：

```bash
# Global 上检查 Inventory
kubectl -n cpaas-system get machineinventories.elemental.cattle.io -o yaml
```

计划使用的 Inventory 至少应逐步出现：

- `Ready=True`；
- `os.baremetal.alauda.io/id=alaudaos.k.25`；
- `os.baremetal.alauda.io/arch=arm64`；
- `status.plan.secretRef.name` 非空；
- 安装完成后 `status.plan.state=Applied`；
- `status.observedNetwork` 已发现；
- 目标主机的 UUID/MAC 与该 Inventory 对应。

在 VM 控制台检查安装相关日志：

```bash
journalctl -b --no-pager | grep -Ei 'elemental|register|install|error|failed'
```

安装完成后再确认磁盘：

```bash
lsblk -o NAME,PATH,TYPE,SIZE,FSTYPE,LABEL,PARTLABEL,MOUNTPOINTS
findmnt /
df -h /
```

如果目标盘仍然没有 A/B 根分区、EFI 分区、`/persist` 或 root 文件系统，不要卸载 ISO 重启；先回到安装日志和 Inventory plan secret 排查。`eject-cd: true` 和 `reboot: true` 应由安装流程在成功后执行。

#### 6.2.6 安装完成后的启动顺序

确认安装成功后，对每台 VM 分别执行：

1. 让安装流程按 `eject-cd: true` 弹出 SeedImage ISO；如果未自动弹出，再在 DCS 控制台手动卸载；
2. 将启动顺序改为 disk-first；
3. 从刚才确认的系统盘启动；
4. 确认系统不再从 Live ISO 启动；
5. 确认根文件系统来自安装盘，而不是 Live ISO；
6. 确认节点网络和 DNS 可访问 Global。

如果卸载 ISO 后直接进入 PXE，先重新挂载 ISO 检查硬盘是否有 GPT、EFI 分区、ARM64 EFI 文件和 root filesystem；不要反复切换启动顺序掩盖安装失败。

不要把 `/dev/sda`、`/dev/vda` 或 `/dev/nvme0n1` 写入 YAML；它们最多只能作为现场排查时根据 `lsblk` 确认出的临时设备名。

### 6.3 CP-03：发现并核对三台 MachineInventory

安装完成并从磁盘启动后，在 Global 上执行：

```bash
kubectl -n cpaas-system \
  get machineinventories.elemental.cattle.io -o wide
```

需要进一步查看每台 Inventory：

```bash
kubectl -n cpaas-system get machineinventory <INVENTORY_NAME> -o yaml
```

列出 Inventory 的 K25 标签和 plan 状态：

```bash
kubectl -n cpaas-system get machineinventories.elemental.cattle.io \
  -o custom-columns='NAME:.metadata.name,ARCH:.metadata.labels.os\.baremetal\.alauda\.io/arch,OS:.metadata.labels.os\.baremetal\.alauda\.io/id,READY:.status.conditions[?(@.type=="Ready")].status,PLAN:.status.plan.state,NIC:.spec.observedNetwork.interfaces[0].name'
```

只把以下状态均满足的 Inventory 放入 CP Pool：

- `Ready=True`；
- `os.baremetal.alauda.io/id=alaudaos.k.25`；
- `os.baremetal.alauda.io/arch=arm64`；
- allocation 为空或显示可用；
- owner 为空；
- 已从安装后的磁盘启动，不是 Live ISO；
- UUID、MAC、hostname 和规划的 CP 节点一一对应；
- `observedNetwork` 中的网卡名已记录；
- 安装完成后 plan state 为 `Applied`；
- 节点 IP 与 `10.243.166.130`、`10.243.166.131`、`10.243.166.132` 的规划对应。

然后只修改 `cp/02-control-plane-pool.yaml` 中的这些现场字段：

```yaml
machineInventories:
  - name: "<实际 CP01 Inventory 名称>"
    hostname: arm-workloadcluster-cp01
    networkDevice: "<CP01 实际 CNI 网卡名>"
```

CP02、CP03 按同样方式填写。不要修改 Pool 的 `clusterName` 和资源名，除非这是另一个集群模板。

### 6.4 CP-04：创建 Control Plane MachineInventoryPool

文件：

```text
cp/02-control-plane-pool.yaml
```

在 apply 前最后检查三件事：

- 三个 Inventory 名称是真实存在且 Ready 的名称；
- 三个 `networkDevice` 是每台 CP 的实际 CNI 网卡名；
- 三个 Inventory 没有被其他 Pool 或 Machine 占用。

```bash
kubectl apply --dry-run=server -f cp/02-control-plane-pool.yaml
kubectl apply -f cp/02-control-plane-pool.yaml
kubectl -n cpaas-system get machineinventorypool arm-workloadcluster-control-plane-pool -o yaml
```

确认 Pool 已接受三台 Inventory 后再继续。

### 6.5 CP-05：创建 BaremetalCluster

文件：

```text
cp/03-baremetal-cluster.yaml
```

部署前确认：

- `controlPlaneLoadBalancer.host` 是已配置好的 `10.243.166.138`；
- 外部 LB 已 TCP passthrough 到三台 CP 的 `6443`；
- `networkDevice` 已替换为集群默认 CNI 网卡名；
- `networkType` 与目标环境的 Kube-OVN 配置一致。

```bash
kubectl apply --dry-run=server -f cp/03-baremetal-cluster.yaml
kubectl apply -f cp/03-baremetal-cluster.yaml
kubectl -n cpaas-system get baremetalcluster arm-workloadcluster -o yaml
```

### 6.6 CP-06：创建 Control Plane BaremetalMachineTemplate

文件：

```text
cp/04-control-plane-machine-template.yaml
```

该文件通常只需要确认 namespace、模板名称和 Pool 引用，不需要填写节点 IP：

```yaml
machineInventoryPoolRef:
  name: arm-workloadcluster-control-plane-pool
```

执行：

```bash
kubectl apply --dry-run=server -f cp/04-control-plane-machine-template.yaml
kubectl apply -f cp/04-control-plane-machine-template.yaml
```

### 6.7 CP-07：创建 CAPI Cluster

文件：

```text
cp/05-cluster.yaml
```

部署前检查：

- `infrastructureRef` 指向 `BaremetalCluster/arm-workloadcluster`；
- `controlPlaneRef` 指向 `KubeadmControlPlane/arm-workloadcluster-control-plane`；
- Pod CIDR 为 `100.150.0.0/16`；
- Service CIDR 为 `100.151.0.0/16`；
- Kube-OVN Join CIDR 为 `100.68.0.0/16`；
- Registry 和 Kube-OVN 版本与目标 ACP 4.4.1 一致。

```bash
kubectl apply --dry-run=server -f cp/05-cluster.yaml
kubectl apply -f cp/05-cluster.yaml
kubectl -n cpaas-system get cluster arm-workloadcluster -o yaml
```

### 6.8 CP-08：创建 KubeadmControlPlane

文件：

```text
cp/06-control-plane.yaml
```

部署前检查：

- `replicas: 3`；
- `version: v1.35.6-acp.9`；
- `machineTemplate.infrastructureRef` 指向 CP BaremetalMachineTemplate；
- SSH 公钥已替换；
- CoreDNS、etcd、imageRepository 与目标 ACP 版本一致；
- kubelet patch 含 `imagePullCredentialsVerificationPolicy: NeverVerify`；
- controller-manager 和 kubelet 使用 `/opt/libexec/kubernetes/kubelet-plugins/volume/exec/`；
- 不要把 `preKubeadmCommands` 中的现场私有命令从其他环境直接复制过来。

```bash
kubectl apply --dry-run=server -f cp/06-control-plane.yaml
kubectl apply -f cp/06-control-plane.yaml
```

控制面资源检查：

```bash
kubectl -n cpaas-system \
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

### 7.1 Worker-01：创建 Worker Registration 和 SeedImage

文件：

```text
worker/01-registration-seedimage.yaml
```

确认以下内容：

- namespace 为 `cpaas-system`；
- Worker Registration 和 SeedImage 名称与 CP 不重复；
- `baseImage` 为 ARM64 `10.243.166.5:11443/tkestack/kubeos-iso:<K25_ARM64_IMAGE_TAG>`；
- `install.system-uri` 为配对的 `kubeos-rootfs:<K25_ARM64_IMAGE_TAG>`；
- `targetPlatform: linux/arm64`；
- `device: /dev/elemental-install-target`；
- `machineName` 和 UUID/Serial Number 表达式保持不变；
- OS 标签为 `alaudaos.k.25`，架构使用 `${Runtime/Architecture}`；
- TPM 设置符合 Worker VM；不要再配置 COS_STATE。

```bash
kubectl apply --dry-run=server -f worker/01-registration-seedimage.yaml
kubectl apply -f worker/01-registration-seedimage.yaml
kubectl -n cpaas-system describe seedimage arm-workloadcluster-worker-registration-iso
```

取出 Worker 生成 ISO 的下载地址和校验地址：

```bash
kubectl -n cpaas-system get seedimage arm-workloadcluster-worker-registration-iso \
  -o jsonpath='{.status.downloadURL}{"\\n"}'

kubectl -n cpaas-system get seedimage arm-workloadcluster-worker-registration-iso \
  -o jsonpath='{.status.checksumURL}{"\\n"}'
```

将 `downloadURL` 对应的生成 ISO 挂载到 Worker VM。不要使用 CP SeedImage 的下载地址。继续前确认 `SeedImageReady=True`、`downloadURL` 和 `checksumURL` 非空。CP 和 Worker 可以使用同一个 ARM64 ISO 基础镜像，但必须是两个独立的 SeedImage 资源。

### 7.2 Worker-02：启动 ISO、选择系统盘并安装三台 Worker

对 `worker01`、`worker02`、`worker03` 分别执行与 CP 相同的磁盘流程：

1. 在 DCS 控制台挂载 Worker SeedImage ISO 并从虚拟光驱启动；
2. 在 ARM64 Live OS 中执行 `lsblk -d -e7 -o NAME,PATH,TYPE,SIZE,MODEL,SERIAL,WWN,HCTL`；
3. 根据磁盘 ID、容量、MODEL、SERIAL/WWN 确认系统盘；
4. 优先选择 `/dev/disk/by-id/<稳定磁盘 ID>`；
5. 创建软链接：

```bash
DISK_BY_ID=/dev/disk/by-id/<CONFIRMED_WORKER_SYSTEM_DISK_ID>
test -b "$DISK_BY_ID"
sudo rm -f /dev/elemental-install-target
sudo ln -s "$DISK_BY_ID" /dev/elemental-install-target
readlink -f /dev/elemental-install-target
```

6. 让 Elemental 使用 `/dev/elemental-install-target` 完成安装；
7. 重启后弹出 ISO，改为 disk-first；
8. 确认系统从安装盘启动，而不是继续从 Live ISO 启动。

软链接只在当前 VM 的 Live OS 环境中根据实际磁盘创建，不能把某台 VM 的 `/dev/sdX`、`/dev/vdX` 或 by-id 路径复制到其他节点的 YAML 中。

安装后再配置并持久化节点 IP：

```text
worker01 -> 10.243.166.133
worker02 -> 10.243.166.134
worker03 -> 10.243.166.135
```

具体接口、前缀、网关、DNS、VLAN/bond 配置必须以现场网络表和 VM 实际网卡为准，不能直接复制 Mandiri 的 `eth0` 或 `eth0.329`。

### 7.3 Worker-03：发现 Inventory 并填写 Worker Pool

```bash
kubectl -n cpaas-system \
  get machineinventories.elemental.cattle.io -o wide
kubectl -n cpaas-system get machineinventory <WORKER_INVENTORY_NAME> -o yaml
```

确认三台 Worker Inventory：

- `Ready=True`；
- `os.baremetal.alauda.io/id=alaudaos.k.25`；
- `os.baremetal.alauda.io/arch=arm64`；
- allocation 为空或可用；
- owner 为空；
- 已从安装后的磁盘启动；
- UUID、MAC、hostname 和 Worker IP 一一对应；
- 安装完成后 plan state 为 `Applied`；
- `observedNetwork` 中的网卡名已记录。

只修改 `worker/02-worker-pool.yaml` 中的实际 Inventory 名和实际网卡名：

```yaml
machineInventories:
  - name: "<实际 Worker01 Inventory 名称>"
    hostname: arm-workloadcluster-worker01
    networkDevice: "<Worker01 实际 CNI 网卡名>"
```

Worker02、Worker03 按相同方式填写，然后执行：

```bash
kubectl apply --dry-run=server -f worker/02-worker-pool.yaml
kubectl apply -f worker/02-worker-pool.yaml
kubectl -n cpaas-system get machineinventorypool arm-workloadcluster-worker-pool -o yaml
```

### 7.4 Worker-04：创建 Worker BaremetalMachineTemplate

文件：

```text
worker/03-worker-machine-template.yaml
```

确认 `machineInventoryPoolRef.name` 为：

```text
arm-workloadcluster-worker-pool
```

```bash
kubectl apply --dry-run=server -f worker/03-worker-machine-template.yaml
kubectl apply -f worker/03-worker-machine-template.yaml
```

### 7.5 Worker-05：创建 Worker KubeadmConfigTemplate

文件：

```text
worker/04-worker-kubeadm-config-template.yaml
```

部署前修改：

- SSH 公钥；
- namespace（当前为 `cpaas-system`）；
- kubelet patch 含 `NeverVerify`；
- join 配置含 `/opt/libexec/kubernetes/kubelet-plugins/volume/exec/`；
- Worker 标签和目标环境需要的 bootstrap 配置。

保留：

```yaml
node-labels: "kube-ovn/role=worker"
```

```bash
kubectl apply --dry-run=server -f worker/04-worker-kubeadm-config-template.yaml
kubectl apply -f worker/04-worker-kubeadm-config-template.yaml
```

### 7.6 Worker-06：创建 MachineDeployment

文件：

```text
worker/05-worker-machine-deployment.yaml
```

部署前确认：

- `replicas: 3`；
- `version: v1.35.6-acp.9`；
- `clusterName: arm-workloadcluster`；
- `bootstrap.configRef` 指向 Worker KubeadmConfigTemplate；
- `infrastructureRef` 指向 Worker BaremetalMachineTemplate；
- `maxSurge: 0`，避免裸金属滚动更新时临时超配。

```bash
kubectl apply --dry-run=server -f worker/05-worker-machine-deployment.yaml
kubectl apply -f worker/05-worker-machine-deployment.yaml
```

Worker 检查：

```bash
kubectl -n cpaas-system \
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
- 没有 `ImageCatalogMiss`、`OSFamilyUnknown`、Inventory 未就绪、错误网卡或重复 IP。

建议先用一台可重装 ARM VM 试点：SeedImage Ready、Inventory OS/arch 正确、A/B 和 `/persist` 正确、plan `Applied`、磁盘启动正常。试点失败时停止扩展到其余五台，先确认当前 Bare Metal Provider 是否真正支持 K25。

## 9. 常见故障边界

- **SeedImage 没有可下载地址**：先检查 ARM ISO、默认 StorageClass、PVC 和 Bare Metal Provider，不要继续挂 ISO。`cleanupAfterMinutes: 240` 后 URL 可能被清理。
- **Inventory 未 Ready**：检查 VM 是否仍从 Live ISO 启动、网卡/网络是否可达、时间同步、TPM、SMBIOS/DMI。
- **反复启动 ISO**：弹出虚拟介质并将 BMC/DCS 启动顺序改为 disk-first。
- **ImageCatalogMiss**：确认三段式 key `v1.35.6-acp.9_arm64_alaudaos.k.25` 指向 ARM64 rootfs，而不是 ISO。
- **OSFamilyUnknown**：检查 Inventory 的 `os.baremetal.alauda.io/id` 是否为 `alaudaos.k.25`。
- **rootfs 拉取失败**：核对 `system-uri`、Live ISO registry 配置和交付包导入方式。
- **kubeadm 写只读路径失败**：检查三处 `/opt/libexec/kubernetes/kubelet-plugins/volume/exec/`。
- **persist-preflight**：检查 `/persist` 可用空间是否大于一个根分区再加约 2 GiB。
- **Pool 不可用**：检查 Inventory 名、owner、Ready 状态、OS/arch 标签和 `observedNetwork`，不要猜 Inventory 名。
- **API 不通**：检查 `10.243.166.138:6443` 是否 TCP 透传到三台 CP，Worker 不应进入 API LB。
- **CIDR 冲突**：停止部署，重新选择未被 Global、其他 Workload 或物理网络使用的 Pod/Service/Join CIDR。

## 10. 安全注意事项

密码、Registry 凭据、kubeconfig、私钥、Registration URL 和临时令牌不得写入本项目。SSH 公钥可以写入 KCP/Worker 清单，但私钥不能进入仓库。
