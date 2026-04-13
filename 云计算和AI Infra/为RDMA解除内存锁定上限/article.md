K8s-containerd 环境下为 RDMA 修改 Pod 内存锁定上限

# 最终解决方案

为了省流和节约时间，首先直接给出最终使用且有效的解决方案。

该方案的核心在于修改 **OCI Runtime Specification (OCI 运行时规范)**。在 containerd 的语境下，通过 `base_runtime_spec` 参数指定的 JSON 文件被视为容器的**基础规范模板**。它是定义容器资源边界（如 `memlock` 限制）的最终权威来源（详见 [OCI 配置规范](https://github.com/opencontainers/runtime-spec/blob/main/config.md#posix-process)），底层的 OCI 运行时（如 runc）会严格按照该蓝图的定义，通过 `setrlimit` 系统调用来初始化容器进程的资源限额。

## 导出现有配置并修改

由于 containerd 并不支持将该参数与系统默认配置进行自动合并（Merge），而是采取完全替换（Replace）策略，因此我们必须先导出一份包含当前系统完整定义的配置模板，再在其基础上进行微调。在节点上运行以下命令。

```shell
ctr oci spec > /etc/containerd/rdma-spec.json
vim /etc/containerd/rdma-spec.json
```

修改配置文件的以下部分（在 `rlimits` 数组中追加 `RLIMIT_MEMLOCK`）：

```json
        "rlimits": [
            {
                "type": "RLIMIT_NOFILE",
                "hard": 1024,
                "soft": 1024
            },
            {
                "type": "RLIMIT_MEMLOCK",
                "hard": 18446744073709551615,
                "soft": 18446744073709551615
            }
        ],
```

## 引用修改后的配置

修改 containerd 的配置。

```toml
      [plugins."io.containerd.grpc.v1.cri".containerd.runtimes]

        [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.nvidia]
          # 修改为指向上述 JSON 模板的绝对路径
          base_runtime_spec = "/etc/containerd/rdma-spec.json"
          cni_conf_dir = ""
          cni_max_conf_num = 0
          container_annotations = []
          pod_annotations = []
          privileged_without_host_devices = false
          privileged_without_host_devices_all_devices_allowed = false
          runtime_engine = ""
          runtime_path = ""
          runtime_root = ""
          runtime_type = "io.containerd.runc.v2"
          sandbox_mode = "podsandbox"
          snapshotter = ""
```

需要根据具体情况修改，目标是让对应的 Pod 成功应用对应的配置。

> 修改容器运行时配置并重启服务后，**不会影响已经在运行的 Pod**。必须手动删除旧的 Pod，由 Kubernetes 重新调度并触发 containerd 使用新的 OCI 蓝图进行初始化，配置才会真正生效。

## 解除 containerd 自身的限制（可选）

对于使用 systemd 管理的 containerd，可以通过如下操作来实现这个修改。

```shell
EDITOR=vim systemctl edit containerd
```

在守护进程配置中添加以下内容。

```toml
[Service]
LimitMEMLOCK=infinity
```

然后使用 `systemctl restart containerd` 重启服务。

虽然实测显示由于 containerd 拥有 `CAP_SYS_RESOURCE` 特权，仅修改 OCI 蓝图即可解除 Pod 内存锁定上限，但补全此项是为了确保守护进程自身的鲁棒性，作为生产环境的最佳实践推荐开启。

## 测试

至此，Pod 中的内存锁定上限应该已被解除，我们可以再次运行 RDMA 带宽测试命令，分别能够在主机和从机上看到以下输出。

**Master:**

```
root@pyt-***-260206-2a3fe-master-0:/# ib_write_bw -s 1M -d mlx5_0 -F
************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x2c QPN 0x0087 PSN 0xcc6a4c RKey 0x1fcc bd VAddr 0x037f67d3b5030
 remote address: LID 0x2c QPN 0x0088 PSN 0x6dc93a RKey 0x1fcf be VAddr 0x037f026db4030
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 1048576    5000             11133.60            11129.09           0.011123
---------------------------------------------------------------------------------------
```

**Worker:**

```
root@pyt-***-260206-2a3fe-worker-0:/# ib_write_bw -s 1M -d mlx5_0 -F 10.244.44.139
---------------------------------------------------------------------------------------
                    RDMA_Write BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 1            Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 TX depth        : 128
 CQ Moderation   : 1
 Mtu             : 4096[B]
 Link type       : IB
 Max inline data : 0[B]
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
 local address: LID 0x2c QPN 0x0088 PSN 0x6dc93a RKey 0x1fcf be VAddr 0x037f026db4030
 remote address: LID 0x2c QPN 0x0087 PSN 0xcc6a4c RKey 0x1fcc bd VAddr 0x037f67d3b5030
---------------------------------------------------------------------------------------
 #bytes     #iterations    BW peak[MB/sec]    BW average[MB/sec]   MsgRate[Mpps]
 1048576    5000             11133.60            11129.09           0.011123
---------------------------------------------------------------------------------------

root@pyt-***-260206-2a3fe-worker-0:/# ulimit -l
unlimited
```

可以看到限制已经解除，且 RDMA 已经可以在 Pod 间进行较大数据量的通信。

## NOTES

这是针对单个节点的操作，如果有大量需要这样配置的节点，那么最好使用脚本等方法来自动化完成，而不是手动配置。

重装 containerd 等操作可能导致上述配置丢失，需要重新走一遍这个流程。

---

# 问题

## 环境

集群与对应节点软件环境如下。

| 项目 | 版本 |
| --- | --- |
| 集群 Server Version | v1.34.3 |
| 节点 Kubelet 版本 | v1.34.3 |
| 操作系统 | Ubuntu 22.04.5 LTS |
| 内核版本 | 5.15.0-170-generic |
| containerd | 1.7.28 |

需要注意的是，在实际操作之后，项目组对集群 Kubernetes 和节点内核版本进行过一次升级，在升级后我又重新做了一次上面的操作，这里给出的是升级后的版本。

也就是说，对于稍旧的版本，这套方案也是有效的；此外，升级内核后可能会丢失相关的配置，推测是 containerd 也被重装了，因此需要重新在节点上进行相关操作。

## 问题现象

实验室有做 RDMA 相关的项目组，在使用我们内部的 Crater 集群管理平台时，发现以下问题。

```
[host2] $ ib_read_bw -q 30 10.244.46.50
Couldn't allocate MR
failed to create mr
Failed to create MR
 Couldn't create IB resources
```

RDMA 进行带宽测试时，小数据量（如 1024 字节）传输正常，但 1M 及以上数据量传输失败，具体的报错信息与上面类似，详见 [Issue #339](https://github.com/raids-lab/crater/issues/339)。

直接使用 `ulimit -l` 查看限制，得到的是 64KB，即系统默认的内存锁定上限。

此时在容器中使用 `ulimit -l unlimited` 或者修改 `/etc/security/limits.conf` 均不能成功修改内存锁定上限。

此外，连通性测试时产生的以下报错，也是同样的问题导致的。

```
[host1] $ ib_read_bw -q 30

************************************
* Waiting for client to connect... *
************************************
---------------------------------------------------------------------------------------
                    RDMA_Read BW Test
 Dual-port       : OFF          Device         : mlx5_0
 Number of qps   : 30           Transport type : IB
 Connection type : RC           Using SRQ      : OFF
 PCIe relax order: ON
 ibv_wr* API     : ON
 CQ Moderation   : 1
 Mtu             : 4096[B]
 Link type       : IB
 Outstand reads  : 16
 rdma_cm QPs     : OFF
 Data ex. method : Ethernet
---------------------------------------------------------------------------------------
ethernet_read_keys: Couldn't read remote address
 Unable to read to socket/rdma_cm
Failed to exchange data between server and clients
```

## 问题分析

RDMA 的核心是让 hardware（网卡 HCA）绕过 CPU 直接访问远程内存。在普通操作中，操作系统为了优化内存，会随时进行“换页” (Paging) 或移动数据，导致虚拟地址对应的物理地址发生变化。如果网卡正在传输数据时，内核将该页内存换出或移动，会导致传输崩溃甚至系统错误。

所以，为了保证地址绝对稳定，RDMA 在传输前必须执行“内存注册” (Memory Registration, MR)。这一动作在内核层面将指定的虚拟内存页“锁死”在物理内存中，禁止内核对其进行移动或交换到磁盘。

Crater 已经通过给作业设置 `CAP_IPC_LOCK` 允许其执行锁定内存的操作，但是没有修改允许锁定的量的限制。同时，由于没有给予容器 `CAP_SYS_RESOURCE` 的权限，容器内部的操作也无法修改这个上限。

目前 Kubernetes 官方在 Pod Spec 中并未提供直接设置 rlimit 的配置项，详见 [Kubernetes Issue #3595](https://github.com/kubernetes/kubernetes/issues/3595)，因此需要从容器运行时的角度下手。

> 实际上用户是需要这个功能的，但是 Kubernetes 和 containerd 似乎都不认为这在自己的职责范围内，导致目前没有一个优雅的方法能够解决这个问题。
> 按理来说，我们希望的是，可以直接在创建 Pod 时传递一个参数，来解决这个 rlimit 的问题。
> 另外，值得期待的是，Kubernetes 社区已经在 v1.36 周期内正式启动了关于原生支持 Pod 级 rlimit 配置的讨论，详见 [KEP-5758](https://github.com/kubernetes/enhancements/pull/5762)。

dockerd 运行时提供了相应的配置项 `default-ulimits`，可以比较方便的在节点层面配置这个限制，详见[配置文件说明](https://docs.docker.com/reference/cli/dockerd/#daemon-configuration-file)和[资源限制指南](https://docs.docker.com/engine/containers/resource_constraints/#set-default-ulimits-for-a-container)。

但目前集群中使用的是 containerd，它并不提供类似 dockerd 的配置项，详见[配置文件说明](https://github.com/containerd/containerd/blob/main/docs/cri/config.md)，同时开发者在 [Issue #3150](https://github.com/containerd/containerd/issues/3150) 中明确拒绝提供守护进程级别的配置。因此需要另辟蹊径以解决这个问题。

这也就引出了我们最终通过 OCI 来解决这个问题的想法。

---

# 踩坑与探索

经过验证，以下方案在我们的环境中是无效的。

1. 直接在 Pod 中通过使用 `ulimit -l unlimited` 或者修改 `/etc/security/limits.conf` 后重新登录，均无法直接修改内存锁定上限，原因推测为 Pod 没有 `CAP_SYS_RESOURCE` 的权限，无法突破预设的上限。
2. 直接修改 Pod 将要调度到的节点宿主机的 `/etc/security/limits.conf`，或是仅为宿主机容器运行时守护进程设置 `LimitMEMLOCK=infinity`，原因推测为容器执行器 runc 会严格按照 OCI 规范蓝图中的定义，在启动时为容器执行 `setrlimit` 系统调用，这会将容器的限制强制覆盖（通常是拉低）为蓝图中的默认值，比如这里的 64KB。
3. 现版本的 Kubernetes 在 Pod（或容器）的 API 里没有提供可直接设置 rlimit 的字段，无法在平台创建对应 Pod 时直接放开这个限制。

---

本文中的 rlimit 指操作系统对每个进程施加的一类资源上限，在系统接口与容器运行时配置里常以 `RLIMIT_*` 与 `setrlimit` 等形式出现；`ulimit` 则是 Shell 内建命令，用于查看或尝试调整当前 shell 进程上的这些上限。二者对应同一套机制，但严格来说并非同一概念。本文所调整的内存锁定上限对应其中的 `RLIMIT_MEMLOCK`，是这一类资源上限中的一个。

除了尝试了许多网络上无效的方法之外，现如今的 AI 在解决这个问题时也表现出了非常严重的幻觉，比如认为 containerd 也有类似 dockerd 那样的配置项，导致我踩了很多坑。

参考的链接直接附在了文中的对应地方，此外，关于 RDMA 的前期配置，可以参看[师兄的博客](https://zhuanlan.zhihu.com/p/1897271506645530541)和[我们平台的文档](https://raids-lab.github.io/crater/zh/docs/admin/more/rdma/)。