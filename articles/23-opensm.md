# OpenSM 源码解析：子网管理器如何发现、路由并统治一个子网

> 从启动到下发 LFT：读透 OpenSM 的核心闭环

---

## 1. 引言与模块全景

### 从两篇前文到本文

「IB 协议那篇」第一次把 SM 这个角色摆上台面：IB 交换机不跑任何分布式路由协议，转发表不是各台交换机自己收敛出来的，而是由 SM 集中算好、统一下发。那篇讲清了「SM 是谁、SMA/SA 如何配合、子网发现大致分几步」，但只停留在概念与报文层面——SM 究竟是一个怎样的程序、内部如何组织、那条「发现 → 分配 LID → 算路径 → 下发 LFT」的闭环具体落在哪些代码里，都没有展开。

「用户态工具那篇」则把 OpenSM 当成一条命令来介绍：它来自 `opensm` 包，负责发现拓扑、分配 LID、计算并下发转发表，一条 `opensm -g <port_guid>` 就能拉起一个子网管理器。那张卡片解决的是「怎么用」，至于这条命令背后守护进程发生了什么，同样点到为止。

本文补上最后一块拼图：钻进 OpenSM 源码，把 SM 的核心闭环逐段跟读一遍。读完能回答三个问题——OpenSM 启动后如何一步步接管子网、拓扑数据在内存里长什么样、LFT 最终是怎么算出来又怎么写进交换机的。

### OpenSM 是什么

OpenSM 是 IB 生态里最主流的开源子网管理器（Subnet Manager），随 OFED 与 `rdma-core` 生态一起分发。它是一个运行在用户态的守护进程，通常部署在子网中某台 HCA 所在主机（或交换机上的管理 CPU）上，启动后以 master 身份接管整个子网。

它工作的方式可以一句话概括：通过 MAD 与设备通信。OpenSM 并不直接操作硬件寄存器，而是构造 256 字节的 MAD 管理报文，经 QP0（SMP，子网管理报文）与 QP1（GMP，通用管理报文）和每个节点的 SMA 交互——`Get(NodeInfo)` 查询节点信息、`Set(LinearForwardingTable)` 下发配置。这一层收发由 `libvendor` 封装（默认走 `osm_vendor_ibumad.c`），上层逻辑不必关心每个字节如何落到链路上。

为什么 IB 离不开 SM？因为 IB 天生没有分布式路由：交换机不会交换路由信息，也不做最短路径计算，只是一台「查表转发」的执行者。没有 SM，子网里的设备连 LID 都没有，更谈不上互通——这正是 IB 与以太网最根本的分野。本文涉及具体实现处均以 OpenSM 3.3.24 为准。

### 模块全景

![OpenSM 模块全景](../images/23-opensm-modules.png)

上图是 OpenSM 核心代码的模块地图。中心是 `osm_opensm` 主循环，驱动一条清晰的流水线：`osm_sm` / `osm_sm_state_mgr` 管状态机，`osm_subnet` / `osm_node` / `osm_port` / `osm_switch` 承载拓扑数据模型，`osm_lid_mgr` / `osm_ucast_mgr` / `osm_lin_fwd_rcv` 依次完成 LID 分配、单播路由计算与 LFT 下发。`libvendor` 在底层负责 MAD 收发，SA 对外提供查询服务；灰色部分是 QoS、分区、组播与拥塞控制，本文只做职责概览、不展开。

各模块与源码文件的对应关系如下（文件均来自 OpenSM 3.3.24）：

| 模块 | 职责 | 关键文件 | 本文是否深入 |
|------|------|----------|--------------|
| 状态机 | 驱动 SM 从发现到稳态的重配置循环 | `osm_sm.c`、`osm_sm_state_mgr.c` | 本文深入 |
| 拓扑数据模型 | 子网、节点、端口、交换机的内存表示 | `osm_subnet.c`、`osm_node.c`、`osm_port.c`、`osm_switch.c` | 本文深入 |
| LID 分配 | 为每个端口分配唯一 16 位 LID | `osm_lid_mgr.c` | 本文深入 |
| 单播路由(minhop) | 计算任意两点间最短路径 | `osm_ucast_mgr.c` | 本文深入 |
| LFT 下发 | 把算好的线性转发表写入交换机 | `osm_lin_fwd_rcv.c` | 本文深入 |
| MAD 收发(libvendor) | 构造并收发 MAD，屏蔽 QP0/QP1 底层细节 | `libvendor/`（`osm_vendor_ibumad.c`） | 只做职责概览 |
| SA | 子网管理服务，响应 PathRecord 等查询 | `osm_sa.c`（+ `osm_sa_*_record.c`） | 只做职责概览 |
| QoS / 分区 / 组播 / 拥塞控制 | P_Key 分区、组播组、拥塞控制等扩展能力 | `osm_qos.c` / `osm_pkey.c` / `osm_mcast_mgr.c` / `osm_congestion_control.c` | 只做职责概览 |
| 性能监控 | 周期性采集子网性能计数器 | `osm_perfmgr.c` | 只做职责概览 |
| 事件插件 | 将 SM 事件上报给外部订阅者 | `osm_event_plugin.c` | 只做职责概览 |

后续章节的编排就沿着这条闭环走：第 2 章讲原理——SM 如何完成一条主流程；第 3 章逐段跟读源码；第 4 章收尾。

## 2. 原理篇：SM 如何完成一条主流程

### 2.1 启动与状态机

### 2.2 拓扑发现

### 2.3 LID 与路由

### 2.4 LFT 下发与稳态

## 3. 源码篇：逐段跟读实现

### 3.1 启动与状态机

### 3.2 拓扑发现

### 3.3 LID 与路由

### 3.4 LFT 下发与稳态

## 4. 收尾
