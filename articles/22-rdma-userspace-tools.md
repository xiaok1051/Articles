# RDMA 用户态工具地图：从子网发现到打流抓包

> 把 RDMA 机器“看懂、问清、压满、抓准”的用户态工具箱

---

前面几篇已经把 RoCEv2 报文、PFC/ECN/DCQCN、IB 子网和 verbs 调用栈讲清楚了。真到机器上干活时，问题会变得更朴素：设备认没认到？端口起来没？带宽够不够？计数器有没有异常？这篇文章把常用的用户态工具按类别整理一遍，目标不是替代 man page，而是让你知道每个工具该在什么位置出场。

## 1. 工具全景与安装来源

![RDMA 用户态工具全景](../images/22-rdma-tools-map.png)

RDMA 用户态工具大体来自几个包。不同发行版包名略有差异，但归属关系基本稳定：

| 包/来源 | 提供什么 | 代表工具 |
|---------|----------|----------|
| `rdma-core` / `ibverbs-utils` | verbs 基础查询 | `ibv_devices`、`ibv_devinfo` |
| `infiniband-diags` | IB 原生查询与诊断 | `ibstat`、`ibnetdiscover`、`iblinkinfo`、`ibping`、`ibtracert`、`perfquery`、`sminfo`、`smpquery`、`saquery` |
| `opensm` | IB 子网管理器本体 | `opensm` |
| `ibutils2`（或厂商 OFED） | IB 子网体检 | `ibdiagnet` |
| `perftest` | RDMA 带宽/延迟打流 | `ib_write_bw`、`ib_read_bw`、`ib_send_bw`、`ib_atomic_bw`、对应 `*_lat` |
| `qperf` | 轻量 RDMA/网络性能测试 | `qperf` |
| `tcpdump` / `wireshark` | 以太网抓包与解码 | `tcpdump`、`tshark`、`wireshark` |
| `ethtool` / `iproute2` | 网卡与链路通用观测 | `ethtool -S`、`ip -s link`、`rdma link`、`rdma statistic` |

本文默认命令基线是 Linux + `rdma-core` 生态。需要真实 IB 子网、SM 或物理 RDMA 网卡的例子，会单独标注。

## 2. 设备、端口与拓扑查询工具

先解决最基础的问题：这台机器上有没有 RDMA 设备、端口状态如何、它在子网里是谁。下面几张工具卡片按从通用到 IB 原生的顺序排列。

### rdma link / rdma statistic

- 解决什么问题：在 RoCE/通用 RDMA 视角查看设备、端口状态与统计。
- 来自哪个包：`iproute2`（`rdma` 子命令）。
- 常用命令：

```bash
rdma link show
rdma statistic show
```

- 输出重点看什么：设备名、端口 index、state、physical_state、netdev；统计项是否随流量增长。
- 常见坑：`rdma` 是通用 RDMA 视角，不等同于 IB 子网视图；要看 LID/SM 仍需 IB 工具。

### ibv_devices / ibv_devinfo

- 解决什么问题：确认 verbs 层能枚举到哪些 RDMA 设备，以及设备能力。
- 来自哪个包：`ibverbs-utils`（Debian/Ubuntu）或 `rdma-core` 相关包。
- 常用命令：

```bash
ibv_devices
ibv_devinfo -v
```

- 输出重点看什么：hca_id、node GUID、port state、active_mtu、max_qp、max_cq 等设备能力。
- 常见坑：内核驱动能看到设备，不代表 verbs 用户态一定可用；容器/权限/udev 都可能影响枚举。

### ibstat

- 解决什么问题：快速查看 IB CA 端口状态。
- 来自哪个包：通常来自 `infiniband-diags`。
- 常用命令：

```bash
ibstat
ibstat <ca> <port>
```

- 输出重点看什么：`State: Active`、`Physical state: LinkUp`、rate、Base lid、SM lid、Port GUID。
- 常见坑：RoCE 上也可能有 `ibstat` 输出，但 LID/SM 语义与原生 IB 不同，不要按 IB 子网逻辑硬套。

### ibnetdiscover / iblinkinfo

- 解决什么问题：发现 IB 子网拓扑，或打印更短的链路信息。
- 来自哪个包：通常来自 `infiniband-diags`。
- 常用命令：

```bash
ibnetdiscover
iblinkinfo
```

- 输出重点看什么：switch/CA GUID、端口连接关系、速率/宽度。
- 常见坑：需要 SM 已经把子网组建起来；大型子网输出很长，建议配合 `grep` 定位 GUID 或端口。

## 3. IB 子网管理工具

设备看见了，接下来要问：这个子网是谁管起来的、管子网的人状态如何、我能从它那里问到什么。下面几张卡片回答这三个问题。

### opensm

- 解决什么问题：IB 子网的中心管理器，负责发现拓扑、分配 LID、计算并下发转发表。
- 来自哪个包：`opensm`。
- 常用命令：

```bash
opensm -g <port_guid> -f /var/log/opensm.log
ps aux | grep opensm
```

- 输出重点看什么：日志中是否完成 subnet discovery；是否有 master SM 竞选；是否报错无法绑定端口。
- 常见坑：同一子网只能有一个 master SM；多 SM 时需要理解 priority/standby。RoCE 没有这一层，不要在纯 RoCE 环境里找 SM。

### sminfo

- 解决什么问题：快速确认当前子网 SM 是谁、状态如何。
- 来自哪个包：`infiniband-diags`。
- 常用命令：

```bash
sminfo
```

- 输出重点看什么：`sm lid`、`sm guid`、`activity count`、`priority`、`state 3 SMINFO_MASTER`。
- 常见坑：`sminfo` 失败常见于没有 SM、权限不足或指定端口错误。

### smpquery / saquery

- 解决什么问题：用 SMP/SA 查询子网管理对象，例如 node、port、P_Key、PathRecord。
- 来自哪个包：`infiniband-diags`。
- 常用命令：

```bash
smpquery nodeinfo <lid>
smpquery portinfo <lid> <port>
smpquery pkeys <lid> <port>
saquery -N
saquery -p
```

- 输出重点看什么：node/port 属性、P_Key 成员关系、PathRecord 关键字段。
- 常见坑：SA 查询比 SMP 更偏“目录服务”，理解 P_Key/PathRecord 才能用好 `saquery`。

## 4. IB 诊断工具

SM 把子网建起来之后，真正的问题是：链路健康吗、两个节点之间通不通、交换机转发表长什么样。诊断工具按“体检—抽查—深扒”的思路来用。

### ibdiagnet

- 解决什么问题：IB 子网级体检，汇总拓扑、链路质量、异常端口。
- 来自哪个包：`ibutils2`（或厂商 OFED 包）。
- 常用命令：

```bash
ibdiagnet
ibdiagnet -lw <width> -ls <speed>
ibdiagnet -o <output_dir>
```

- 输出重点看什么：发现的 switch/CA 数量、链路 speed/width 是否符合预期、Symbol Error/BER 相关告警。
- 常见坑：大规模子网上执行有开销；只扫一部分端口时要明确范围，避免误判。

### ibcheck* / ibping / ibtracert

- 解决什么问题：快速做健康检查、节点连通性检查和路径追踪。
- 来自哪个包：通常来自 `infiniband-diags`。
- 常用命令：

```bash
ibchecknet
ibcheckerrors
ibping <lid>
ibtracert <src_lid> <dst_lid>
```

- 输出重点看什么：`ibcheckerrors` 中的 error counter；`ibtracert` 逐跳端口路径。
- 常见坑：`ibping` 需要先有对端以 server 模式运行（`ibping -S`）；LID 会变，子网重建后旧 LID 不能直接再用。

### perfquery / dump_lfts / dump_mfts

- 解决什么问题：查询端口性能计数器，导出交换机线性/组播转发表。
- 来自哪个包：通常来自 `infiniband-diags`。
- 常用命令：

```bash
perfquery <lid> <port>
perfquery -R <lid> <port>
dump_lfts
dump_mfts
ibroute <switch_lid>
```

- 输出重点看什么：PortXmitData/PortRcvData、PortXmitDiscards、SymbolErrorCounter；`ibroute` 输出单台交换机 LID 到 egress port 的映射。
- 常见坑：`perfquery -R` 会 reset 计数器；生产环境执行前要确认是否允许清零。

## 5. 性能打流工具

链路看起来正常，不代表性能达标。打流工具的任务是把带宽压满、把延迟测准，用数字回答“这条路到底能跑多快、有多稳”。

### perftest：带宽打流

- 解决什么问题：验证 RDMA Write/Read/Send/Atomic 路径的带宽和消息率。
- 来自哪个包：`perftest`。
- 常用命令：

```bash
ib_write_bw -d <device> -p <port> -s <msg_size> -n <iters> -F --report_gbits
ib_read_bw  -d <device> -p <port> -s <msg_size> -n <iters> -F --report_gbits
ib_send_bw  -d <device> -p <port> -s <msg_size> -n <iters> -F --report_gbits
ib_atomic_bw -d <device> -p <port> -s <msg_size> -n <iters> -F
```

- 输出重点看什么：`BW average[Gb/s]`、`MsgRate[Mpps]`、是否有 `completed with error`、两端是否都正常退出。
- 常见坑：perftest 是双端工具——一端先跑同一条命令做 server，另一端在命令末尾加对端 IP/主机名做 client；RoCE 环境通常还要 `-x <gid_index>`（或 `-R` 走 rdma_cm）才能建立连接；`-F` 是忽略 CPU 频率告警。只跑通小包不代表大包达标；MTU、GID index、RoCE 版本、PFC/ECN 都会影响结果。

### perftest：延迟打流

- 解决什么问题：测量 RDMA 操作延迟，尤其是 Write/Read/Atomic 的 round-trip。
- 来自哪个包：`perftest`。
- 常用命令：

```bash
ib_write_lat -d <device> -p <port> -s <msg_size> -n <iters> -F
ib_read_lat  -d <device> -p <port> -s <msg_size> -n <iters> -F
ib_send_lat  -d <device> -p <port> -s <msg_size> -n <iters> -F
```

- 输出重点看什么：median/average latency、尾部延迟、是否有异常抖动。
- 常见坑：双端要求和带宽卡片相同（server/client、RoCE 下 `-x`/`-R`）。延迟测试对 CPU 亲和性、中断、频率更敏感；不要只看平均延迟。

### qperf

- 解决什么问题：用更轻量的方式跑 RDMA 或 TCP/UDP 性能基线。
- 来自哪个包：`qperf`。
- 常用命令：

```bash
qperf -v <server_ip> rc_bw rc_lat
qperf -v <server_ip> uc_bw uc_lat
```

- 输出重点看什么：bw/lat 结果是否稳定；连接建立是否成功。
- 常见坑：先在服务端裸跑 `qperf`，再用客户端命令连；`qperf` 结果适合做快速对照，不适合替代 perftest 的细参数验证。

## 6. RoCE 与通用 Linux 网络工具

RoCE 跑在以太网上，排查问题时 RDMA 工具之外还得有通用网络视角：链路协商对不对、丢包丢在哪、报文到底长什么样。下面几张卡片补上这个视角。

### ethtool / ip -s link

- 解决什么问题：看 RoCE 网卡链路、协商速率、丢包/暂停帧/错误计数。
- 来自哪个包：`ethtool`、`iproute2`。
- 常用命令：

```bash
ethtool <ifname>
ethtool -S <ifname>
ip -s link show dev <ifname>
```

- 输出重点看什么：Speed、Duplex、Link detected；`rx_pause`、`tx_pause`、discard/error/drop 相关计数是否增长。
- 常见坑：不同网卡的 `-S` 计数器名字差异很大；不要只凭名字猜含义，结合厂商文档。

### rdma statistic / sysfs counters

- 解决什么问题：从 RDMA 子系统视角看计数器，补充 `ethtool -S`（`rdma statistic` 的基础用法见第 2 节卡片）。
- 来自哪个包：`iproute2`（`rdma` 子命令），以及内核 sysfs。
- 常用命令：

```bash
rdma statistic show
ls /sys/class/infiniband/<device>/ports/<port>/counters
cat /sys/class/infiniband/<device>/ports/<port>/counters/*
```

- 输出重点看什么：xmit/rcv data/packets、error/discard 类计数。
- 常见坑：需要内核启用并绑定计数器（`rdma statistic mode`），部分内核默认看不到数据；sysfs 计数器单位可能是“包”也可能是“字节”，看名字和文档确认。

### tcpdump / tshark / wireshark

- 解决什么问题：抓 RoCEv2 报文，确认 BTH/GRH、QP、PSN、重传迹象。
- 来自哪个包：`tcpdump`、`wireshark`。
- 常用命令：

```bash
tcpdump -i <ifname> udp port 4791 -nn -vv -w roce.pcap
tshark -i <ifname> -f "udp port 4791" -w roce.pcap
tshark -r roce.pcap -Y 'roce' -V
```

- 输出重点看什么：UDP 4791 是否出现；Wireshark/tshark 是否解出 RoCE/IB BTH；PSN 是否连续。
- 常见坑：抓包点不对会什么都看不到；交换机镜像、网卡 offload、加密/隧道都会影响解码。IB 原生报文不是以太网帧，不能直接用这套抓包思路。

## 7. 工具汇总表

| 工具 | 类别 | 安装包 | 常用命令 | 关键输出 | 适用场景 |
|------|------|--------|----------|----------|----------|
| `rdma link` | RoCE/通用 | `iproute2` | `rdma link show` | link 状态、physical_state、netdev | 确认 RDMA 链路视角 |
| `rdma statistic` | RoCE/通用 | `iproute2` | `rdma statistic show` | RDMA 计数器 | RDMA 子系统统计 |
| `ibv_devinfo` | RoCE/通用 | `ibverbs-utils` | `ibv_devinfo -v` | port state、MTU、设备能力 | 确认 verbs 可用性 |
| `ibstat` | IB 原生 | `infiniband-diags` | `ibstat` | State、rate、Base lid、SM lid | 快速看 CA 端口 |
| `ibnetdiscover` | IB 原生 | `infiniband-diags` | `ibnetdiscover` | GUID、连接关系 | 拓扑发现 |
| `iblinkinfo` | IB 原生 | `infiniband-diags` | `iblinkinfo` | 链路状态、速率/宽度 | 快速看链路 |
| `opensm` | IB 子网管理 | `opensm` | `opensm -g <guid>` | discovery、master 状态 | 组建/维护 IB 子网 |
| `sminfo` | IB 子网管理 | `infiniband-diags` | `sminfo` | sm lid、state SMINFO_MASTER | 确认 SM 状态 |
| `smpquery` / `saquery` | IB 子网管理 | `infiniband-diags` | `smpquery nodeinfo <lid>`、`saquery -p` | node/port、P_Key、PathRecord | 查 SMP/SA 对象 |
| `ibdiagnet` | IB 诊断 | `ibutils2` | `ibdiagnet` | 链路速率/宽度、Symbol Error/BER | 子网体检 |
| `ibping` / `ibtracert` | IB 诊断 | `infiniband-diags` | `ibping <lid>`、`ibtracert <src> <dst>` | 连通性、逐跳路径 | 连通性与路径 |
| `perfquery` | IB 诊断 | `infiniband-diags` | `perfquery <lid> <port>` | Xmit/Rcv、discard、symbol error | 端口计数器 |
| `dump_lfts` / `dump_mfts` / `ibroute` | IB 诊断 | `infiniband-diags` | `dump_lfts`、`dump_mfts`、`ibroute <switch_lid>` | 交换机线性/组播转发表 | 转发表导出 |
| `ib_write_bw` | 打流 | `perftest` | `ib_write_bw ... --report_gbits` | BW average、MsgRate | RDMA Write 带宽 |
| `ib_write_lat` | 打流 | `perftest` | `ib_write_lat ...` | latency 分布 | RDMA Write 延迟 |
| `qperf` | 打流 | `qperf` | `qperf -v <ip> rc_bw rc_lat` | bw/lat | 快速性能基线 |
| `ethtool -S` | RoCE/通用 | `ethtool` | `ethtool -S <ifname>` | pause/error/discard 计数 | RoCE 网卡计数器 |
| `tcpdump` | 抓包 | `tcpdump` | `tcpdump -i <ifname> udp port 4791` | UDP 4791 报文 | RoCEv2 抓包 |
| `tshark` | 抓包 | `wireshark` | `tshark -r roce.pcap -Y 'roce'` | RoCE/BTH 解码 | 报文分析 |

## 结尾

这套工具可以分成四层：先看设备/端口是否就绪，再看 IB 子网或 RoCE 链路是否健康，然后用 perftest 把性能打满，最后用计数器和抓包解释“为什么”。前面 RNIC 架构和 UEC 的文章里已经出现过其中一些输出字段，后续做具体网卡排障时还会继续用到它们。
