# 无盘多系统平台 — 技术架构设计

> 版本：v1.1（聚焦版）  
> 当前场景：**1~5 台客户端，千兆网络，单服务端**  
> 扩展研究：见 [`RESEARCH.md`](RESEARCH.md)

---

## 1. 设计约束与目标

| 约束 | 设计影响 |
|------|----------|
| **必须支持 Windows** | 采用 iPXE `sanboot` + iSCSI 方案；镜像需在服务端预装并注入驱动 |
| **当前 1~5 台** | 单服务端即可支撑；存储和网络无需复杂设计 |
| **千兆网络** | 当前场景的默认网络条件；无需额外网络升级 |
| **纯还原 + 数据分离** | 系统盘使用 snapshot 还原；用户数据使用独立持久化卷 |
| **全开源** | 基于 iPXE、LVM、Linux-IO Target、QEMU 等开源组件 |

**核心目标**：
1. 客户端开机 → 选择系统 → 从网络启动进入干净系统盘
2. 重启后系统盘所有变更归零
3. 用户数据（如需要）持久保存在独立数据盘中，重启不丢失

---

## 2. 整体架构

### 2.1 逻辑分层

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                               控制管理层                                       │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────────────────────────┐ │
│  │  镜像仓库     │  │  客户端配置   │  │   Boot API (Python/Shell)           │ │
│  │(Golden Images)│  │(MAC→系统/    │  │   • 按MAC识别客户端                  │ │
│  │              │  │  数据盘映射)  │  │   • 创建/删除系统盘 snapshot         │ │
│  └──────┬───────┘  └──────┬───────┘  │   • 返回 iPXE 启动脚本               │ │
└─────────┼─────────────────┼──────────┘   • 管理用户数据盘生命周期           │ │
          │                 │              └─────────────────────────────────────┘ │
          ▼                 ▼                                                      │
┌─────────────────────────────────────────────────────────────────────────────────┤
│                               启动协议层                                         │
│                                                                                  │
│   ┌─────────────┐      ┌─────────────┐      ┌─────────────────────────────────┐ │
│   │   DHCP      │─────▶│   TFTP      │─────▶│   iPXE 引导脚本                  │ │
│   │ (分配IP+    │      │ (传输iPXE   │      │   • 显示多系统菜单               │ │
│   │  指向iPXE)  │      │  固件)      │      │   • 连接系统盘 iSCSI             │ │
│   └─────────────┘      └─────────────┘      │   • 连接用户数据盘 iSCSI/NFS     │ │
│                                             │   • 加载内核/sanboot 启动        │ │
│                                             └─────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────────┤
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                               存储后端层（单服务端）                             │
│                                                                      │
│   ┌─────────────────────────────────────────────────────────────┐         │
│   │                    LVM Thin Pool                             │         │
│   │                                                              │         │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │         │
│   │   │ ubuntu-gld  │    │ win10-gld   │    │ win11-gld   │     │         │
│   │   │ (thin LV,   │    │ (thin LV,   │    │ (thin LV,   │     │         │
│   │   │  母盘只读)   │    │  母盘只读)   │    │  母盘只读)   │     │         │
│   │   └──────┬──────┘    └──────┬──────┘    └──────┬──────┘     │         │
│   │          │                  │                  │             │         │
│   │          ▼                  ▼                  ▼             │         │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │         │
│   │   │client01-sys │    │client02-sys │    │client03-sys │     │         │
│   │   │(thin snap,  │    │(thin snap,  │    │(thin snap,  │     │         │
│   │   │ 重启丢弃)   │    │ 重启丢弃)   │    │ 重启丢弃)   │     │         │
│   │   └─────────────┘    └─────────────┘    └─────────────┘     │         │
│   │                                                              │         │
│   │   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐     │         │
│   │   │client01-data│    │client02-data│    │client03-data│     │         │
│   │   │(thin LV,    │    │(thin LV,    │    │(thin LV,    │     │         │
│   │   │ 持久保留)   │    │ 持久保留)   │    │ 持久保留)   │     │         │
│   │   └─────────────┘    └─────────────┘    └─────────────┘     │         │
│   │                                                              │         │
│   └─────────────────────────────────────────────────────────────┘         │
│                                │                                           │
│                                ▼                                           │
│   ┌─────────────────────────────────────────────────────────────┐         │
│   │              Linux-IO Target (LIO / targetcli)               │         │
│   │   每个系统 snapshot 和用户数据 LV 导出为独立 iSCSI Target     │         │
│   └─────────────────────────────────────────────────────────────┘         │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼  iSCSI / NFS
┌─────────────────────────────────────────────────────────────────────────────┐
│                               客户端层                                         │
│                                                                      │
│   ┌─────────────────┐    ┌────────────────────────┐    ┌───────────────┐   │
│   │   iPXE 固件      │───▶│   系统盘: iSCSI Boot   │───▶│   OS 运行     │   │
│   │                 │    │   • Linux: root on iSCSI│   │  / C:\        │   │
│   │                 │    │   • Windows: sanboot    │   │               │   │
│   └─────────────────┘    └────────────────────────┘    └───────────────┘   │
│                                      ▲                                     │
│                                      │                                     │
│                           ┌──────────┴──────────┐                         │
│                           │  用户数据盘          │                         │
│                           │  • Linux: /home      │                         │
│                           │  • Windows: D:\      │                         │
│                           │  (iSCSI 或 NFS)      │                         │
│                           └─────────────────────┘                         │
└─────────────────────────────────────────────────────────────────────────────┘
```

### 2.2 数据流（一次完整的启动流程）

```
客户端开机
    │
    ▼
[1] iPXE ROM 发起 DHCP 请求
    │
    ▼
[2] DHCP 返回：IP 地址 + iPXE 引导脚本 URL
    │        （如：http://192.168.31.57/boot?mac=aa:bb:cc:dd:ee:ff）
    │
    ▼
[3] iPXE HTTP 请求 Boot API
    │
    ▼
[4] Boot API 服务端逻辑：
    │    a. 识别客户端 MAC
    │    b. 查询配置：默认系统、是否挂载用户数据盘
    │    c. 删除该客户端旧的系统盘 snapshot（如存在）
    │    d. 从对应 golden image 创建新的系统盘 snapshot
    │    e. 检查用户数据盘是否存在，不存在则创建
    │    f. 返回 iPXE 脚本：系统盘 iSCSI 参数 + 用户数据盘参数 + 启动命令
    │
    ▼
[5] iPXE 执行返回的脚本：
    │    • 连接系统盘 iSCSI target
    │    • 连接用户数据盘 iSCSI target（如配置）
    │    • Linux：加载 kernel + initrd → root=系统盘, 用户数据盘单独挂载
    │    • Windows：sanboot 系统盘，用户数据盘由 OS 内 iSCSI 连接
    │
    ▼
[6] 客户端启动进入操作系统
    │
    ▼
[7] 用户运行系统：
    │    • 写入系统分区 → 落在系统盘 snapshot（重启丢弃）
    │    • 写入用户数据分区 → 落在用户数据盘 thin LV（永久保留）
    │
    ▼
[8] 重启/关机 → Boot API 再次收到请求 → 删除旧系统 snapshot，创建新的
    │               用户数据盘保持不变 ✅
    ▼
[9] 客户端获得全新的干净系统盘，但用户数据仍在 ✅
```

---

## 3. 启动层：iPXE + 动态 Boot API

### 3.1 为什么必须升级到 iPXE

当前 `DISKLESS-GUIDE.md` 使用传统 PXELINUX/GRUB 通过 TFTP 传输配置文件，无法满足：

| 需求 | PXELINUX/GRUB | iPXE |
|------|---------------|------|
| 按客户端动态生成菜单 | ❌ 静态文件 | ✅ 脚本可执行逻辑 |
| Windows iSCSI SAN boot | ❌ 不支持 | ✅ `sanboot` / `sanhook` |
| HTTP 传输（比 TFTP 快） | ❌ 仅 TFTP | ✅ HTTP/HTTPS |
| 脚本化启动流程 | ❌ 无 | ✅ 完整脚本语言 |

### 3.2 iPXE 部署方式

采用**链载（Chainloading）**：网卡自带 PXE → TFTP 下载 `ipxe.pxe` / `ipxe.efi` → iPXE 接管。零侵入，兼容所有支持 PXE 的网卡。

### 3.3 Boot API 设计

**请求**：
```
GET /boot?mac=aa:bb:cc:dd:ee:ff&arch=x86_64
```

**响应**（iPXE 脚本）：
```ipxe
#!ipxe

# 设置图形背景（可选）
console --x 1024 --y 768

# 显示菜单
menu 选择操作系统
item ubuntu    Ubuntu 26.04 (Diskless)
item win10     Windows 10 (Diskless)
item local     从本地硬盘启动
choose --default ubuntu --timeout 10000 target && goto ${target}

:ubuntu
set sys_iscsi iqn.2026-04.com.diskless:client-aa-bb-cc-dd-ee-ff-ubuntu
set data_iscsi iqn.2026-04.com.diskless:client-aa-bb-cc-dd-ee-ff-data
chain http://192.168.31.57/api/prepare?mac=${mac}&os=ubuntu
kernel http://192.168.31.57/images/ubuntu/vmlinuz \
  root=UUID=<系统盘UUID> netroot=iscsi:192.168.31.57::6:3260:0:${sys_iscsi} \
  userdata=iscsi:192.168.31.57::6:3260:0:${data_iscsi} ip=dhcp rw
initrd http://192.168.31.57/images/ubuntu/initrd.img
boot

:win10
set sys_iscsi iqn.2026-04.com.diskless:client-aa-bb-cc-dd-ee-ff-win10
chain http://192.168.31.57/api/prepare?mac=${mac}&os=win10
sanboot iscsi:192.168.31.57::6:3260:0:${sys_iscsi}
```

**Boot API 实现**：Python Flask / FastAPI，单文件即可运行。

---

## 4. 存储层：系统盘还原 + 用户数据持久化

### 4.1 双盘分离设计

这是本架构的核心创新点：将**系统盘**和**用户数据盘**物理分离，实现不同的生命周期策略。

```
┌─────────────────────────────────────────────────────────────────┐
│                         单客户端存储视图                          │
│                                                                  │
│   ┌─────────────────────┐        ┌─────────────────────────┐    │
│   │     系统盘           │        │       用户数据盘         │    │
│   │  ┌───────────────┐  │        │  ┌─────────────────┐    │    │
│   │  │ golden image  │  │        │  │  clientXX-data  │    │    │
│   │  │  (thin LV,    │  │        │  │  (thin LV,       │    │    │
│   │  │   只读母盘)    │  │        │  │   独立存在,      │    │    │
│   │  └───────┬───────┘  │        │  │   不随snapshot   │    │    │
│   │          │           │        │  │   删除)          │    │    │
│   │          ▼ snapshot  │        │  └─────────────────┘    │    │
│   │  ┌───────────────┐  │        │         ▲               │    │
│   │  │ clientXX-sys  │  │        │         │ iSCSI 连接     │    │
│   │  │ (thin snap,   │──┼────────┼─────────┘               │    │
│   │  │  重启丢弃)     │  │        │                         │    │
│   │  └───────────────┘  │        │  Linux: 挂载为 /home    │    │
│   │         ▲           │        │  Windows: 挂载为 D:\    │    │
│   │         │ iSCSI 连接 │        │                         │    │
│   │         │           │        │  生命周期：永久保留       │    │
│   │  Linux: rootfs      │        │  （除非管理员手动删除）   │    │
│   │  Windows: C:\       │        │                         │    │
│   │                      │        │                         │    │
│   │  生命周期：每次启动   │        │                         │    │
│   │  重新创建，重启丢弃   │        │                         │    │
│   └─────────────────────┘        └─────────────────────────┘    │
└─────────────────────────────────────────────────────────────────┘
```

### 4.2 系统盘：LVM Thin Snapshot 还原机制

**为什么选 LVM Thin Provisioning**：

| 方案 | 是否满足纯还原 | 性能 | 空间效率 | 管理复杂度 |
|------|---------------|------|----------|-----------|
| fileio LUN（现有） | ❌ 直接写母盘 | 中 | 低 | 低 |
| qcow2 + backing file | ✅ 支持 | 中 | 高 | 中 |
| **LVM Thin Snapshot** | ✅ 原生支持 | **高**（内核级） | **高** | **低** |
| ZFS snapshot | ✅ 支持 | 高 | 高 | 中 |

**选择 LVM Thin**：内核原生、无守护进程、与 targetcli 无缝集成。

### 4.3 用户数据盘：持久化 thin LV

用户数据盘是**普通的 thin LV**（不是 snapshot），因此：
- 客户端可以正常读写
- 重启后数据保留
- 可在线扩容（`lvextend`）
- 管理员可选择性备份/清空

**挂载方式**：

| 操作系统 | 系统盘 | 用户数据盘 | 挂载点 |
|----------|--------|-----------|--------|
| **Linux** | iSCSI LUN → 根分区 | iSCSI LUN → 独立分区 | `/home` 或自定义路径 |
| **Windows** | iSCSI LUN → C: 盘 | iSCSI LUN → 独立分区 | `D:` 盘或用户文件夹重定向 |

> **Windows 用户数据盘连接**：由于 Windows sanboot 后系统盘已由 iPXE 连接，用户数据盘需要在 Windows 启动后通过内置 iSCSI Initiator 自动连接。这需要在母盘镜像中预配置 iSCSI 持久连接脚本。

### 4.4 LVM 存储布局（1~5台场景）

```bash
# 物理卷（服务端本地磁盘，SATA SSD 或 NVMe 均可，500G+ 即可）
pvcreate /dev/sda3  # 或使用独立数据盘 /dev/nvme0n1

# 卷组
vgcreate diskless-vg /dev/sda3

# Thin Pool（1~5 台场景，100~200G 足够）
# 母盘：Ubuntu 100G + Win10 120G = 220G
# 系统 snapshot：5 台 × 平均 20G COW = 100G
# 用户数据：5 台 × 50G = 250G
# 总计约 600G，thin pool 按实际使用分配
lvcreate -L 500G -T diskless-vg/thin-pool

# ========== 母盘（golden images）==========
# 创建后不再直接修改，通过 snapshot 给客户端使用
lvcreate -V 100G -T diskless-vg/thin-pool -n ubuntu-golden
lvcreate -V 120G -T diskless-vg/thin-pool -n win10-golden

# ========== 客户端系统盘 snapshot（动态创建/删除）==========
# 由 Boot API 自动管理，示例：
# lvcreate -s -n client-aabbccddeeff-ubuntu diskless-vg/ubuntu-golden
# lvremove -f diskless-vg/client-aabbccddeeff-ubuntu

# ========== 客户端用户数据盘（持久化，预创建）==========
# 由管理员或 Boot API 首次启动时创建，之后长期保留
lvcreate -V 50G -T diskless-vg/thin-pool -n client-aabbccddeeff-data
```

### 4.5 targetcli iSCSI 导出策略

**每个客户端两个 iSCSI Target**：

```bash
# 系统盘 target（动态创建/删除）
targetcli /iscsi create iqn.2026-04.com.diskless:client-aa-bb-cc-dd-ee-ff-ubuntu
targetcli /iscsi/iqn.../tpg1/luns create /backstores/block/client-aabbccddeeff-ubuntu
targetcli /iscsi/iqn.../tpg1 set attribute authentication=0 generate_node_acls=1 demo_mode_write_protect=0

# 用户数据盘 target（长期保留）
targetcli /iscsi create iqn.2026-04.com.diskless:client-aa-bb-cc-dd-ee-ff-data
targetcli /iscsi/iqn.../tpg1/luns create /backstores/block/client-aabbccddeeff-data
targetcli /iscsi/iqn.../tpg1 set attribute authentication=0 generate_node_acls=1 demo_mode_write_protect=0
```

> **命名规范**：`iqn.2026-04.com.diskless:client-<mac>-<os>` 便于脚本自动化管理。

### 4.6 纯还原 + 数据保留的生命周期

```
客户端启动
    │
    ▼
Boot API 收到请求
    │
    ├── 系统盘处理：
    │      检查旧 snapshot → lvremove -f diskless-vg/client-XX-sys-old
    │      创建新 snapshot → lvcreate -s -n client-XX-sys-new diskless-vg/ubuntu-golden
    │
    ├── 用户数据盘处理：
    │      检查是否存在 → 不存在则 lvcreate -V 50G -T ... -n client-XX-data
    │      （已存在则直接使用，不删除）
    │
    └── 返回 iPXE 脚本（含两个 iSCSI target 的连接参数）
    │
    ▼
客户端运行期间
    ├── 系统盘写入 → client-XX-sys snapshot（COW）
    └── 用户数据写入 → client-XX-data thin LV（直接写入）
    │
    ▼
客户端重启
    │
    ▼
Boot API 再次收到请求
    ├── 系统盘：删除旧 snapshot，重新创建 ✅ 纯还原
    └── 用户数据盘：保持不变 ✅ 持久化
```

### 4.7 用户数据盘的配置开关

在 Boot API 中，用户数据持久化是**可选配置**：

```python
# Boot API 配置示例（每个客户端独立配置）
clients = {
    "aa:bb:cc:dd:ee:ff": {
        "default_os": "ubuntu",
        "userdata_enabled": True,      # 是否挂载用户数据盘
        "userdata_size": "50G",        # 用户数据盘大小
        "userdata_mount": "/home",     # Linux 挂载点
    },
    "11:22:33:44:55:66": {
        "default_os": "win10",
        "userdata_enabled": False,     # 不挂载数据盘，全还原模式（网吧模式）
    }
}
```

---

## 5. 操作系统支持

### 5.1 Linux（Ubuntu / Debian）

Linux 从 iSCSI 启动是成熟技术，已有 `DISKLESS-GUIDE.md` 验证。

**与现有指南的差异**：

| 项目 | 现有指南 | 目标架构 |
|------|----------|----------|
| 启动器 | PXELINUX/GRUB | **iPXE** |
| 内核来源 | ISO `/casper/vmlinuz` | **已安装系统的 `/boot/vmlinuz`** |
| initrd 来源 | ISO `/casper/initrd` | **已安装系统的 `/boot/initrd.img`** |
| 多客户端 | 单 LUN | **每个客户端独立系统 snapshot + 用户数据盘** |
| 用户数据 | 无 | **独立 iSCSI 分区挂载 /home** |

**用户数据盘挂载（Linux）**：

在 initrd 阶段或 `/etc/fstab` 中配置：
```bash
# /etc/fstab 示例
UUID=<系统盘根分区UUID>  /          ext4  defaults  0 1
UUID=<用户数据盘UUID>    /home      ext4  defaults  0 2
```

用户数据盘需要在 initramfs 中通过 iSCSI 连接，或在内核参数中指定：
```
kernel ... netroot=iscsi:server::6:3260:0:iqn...:client-sys  \
  userdata=iscsi:server::6:3260:0:iqn...:client-data
```

> 如果 initrd 不处理 `userdata=` 参数，可以写一个自定义 initramfs hook，在 `local-top` 阶段连接用户数据盘 iSCSI。

### 5.2 Windows（关键设计）

Windows 无盘是工作量最大的部分，走 **iPXE sanboot + iSCSI** 路线。

#### 5.2.1 技术路线

```
服务端：Windows 母盘镜像 → iSCSI Target → 客户端系统盘 thin snapshot
客户端：iPXE ──sanboot──▶ iSCSI 系统盘 ──▶ Windows 启动
        Windows 内 ──iSCSI Initiator──▶ 连接用户数据盘（如配置）
```

#### 5.2.2 Windows 母盘镜像准备

```bash
# 1. 创建虚拟磁盘
qemu-img create -f raw /tmp/win10-golden-tmp.raw 120G

# 2. QEMU 安装 Windows（带 VirtIO 驱动）
qemu-system-x86_64 \
  -name win10-golden -machine type=q35,accel=kvm -cpu host -smp 4 -m 8192 \
  -drive file=/tmp/win10-golden-tmp.raw,format=raw,if=virtio,cache=none \
  -cdrom /path/to/Win10.iso \
  -drive file=/path/to/virtio-win.iso,media=cdrom \
  -netdev bridge,id=net0,br=br0 -device virtio-net-pci,netdev=net0 \
  -vnc :1
```

Windows 内配置：
```powershell
# 确保 iSCSI Initiator 自动启动
Set-Service -Name msiscsi -StartupType Automatic
Start-Service msiscsi

# 集成常见物理机网卡驱动（关键！）
# 下载 Intel、Realtek、Broadcom 驱动，在 QEMU 内安装
pnputil -i -a C:\Drivers\*.inf

# 可选：配置用户数据盘自动连接脚本
# 写一个 PowerShell 脚本，在系统启动时自动连接 iSCSI 用户数据 target
# 并挂载为 D: 盘

# Sysprep 通用化（推荐）
C:\Windows\System32\Sysprep\Sysprep.exe /generalize /oobe /shutdown
```

导入 LVM：
```bash
lvcreate -V 120G -T diskless-vg/thin-pool -n win10-golden
dd if=/tmp/win10-golden-tmp.raw of=/dev/diskless-vg/win10-golden bs=4M status=progress
rm /tmp/win10-golden-tmp.raw
```

#### 5.2.3 Windows 用户数据盘

由于 iPXE `sanboot` 只连接一个 iSCSI target（系统盘），用户数据盘需要在 Windows 启动后**二次连接**。

**方案：Windows 启动脚本自动连接**

在母盘镜像中预置：
```powershell
# C:\Scripts\Connect-UserData.ps1
# 系统启动时自动执行

$targetIP = "192.168.31.57"
$targetIQN = "iqn.2026-04.com.diskless:client-$env:COMPUTERNAME-data"

# 连接 iSCSI target
New-IscsiTargetPortal -TargetPortalAddress $targetIP
Connect-IscsiTarget -NodeAddress $targetIQN -IsPersistent $true

# 初始化磁盘（首次）并挂载为 D:
$disk = Get-Disk | Where-Object { $_.BusType -eq 'iSCSI' -and $_.IsOffline }
if ($disk) {
    Initialize-Disk -Number $disk.Number -PartitionStyle GPT
    New-Partition -DiskNumber $disk.Number -UseMaximumSize -DriveLetter D
    Format-Volume -DriveLetter D -FileSystem NTFS -NewFileSystemLabel "Data"
}
```

通过组策略或计划任务，让该脚本在系统启动时运行。

#### 5.2.4 已知风险与缓解

| 风险 | 影响 | 缓解措施 |
|------|------|----------|
| 驱动兼容性 | 物理机网卡/芯片组与 QEMU 不同，可能蓝屏 | Sysprep 通用化；注入主流驱动 |
| Windows 激活 | 硬件变更导致激活失效 | 使用 KMS 批量激活 |
| iSCSI 连接移交 | iPXE sanboot 后 Windows 需自己维持连接 | 测试 `keep-san` 选项；准备网卡驱动 |
| 用户数据盘连接失败 | Windows 内脚本执行失败 | 添加重试逻辑和错误日志 |

---

## 6. 多系统切换机制

### 6.1 切换方式

| 方式 | 场景 | 实现 |
|------|------|------|
| **手动选择**（默认） | 客户端开机显示菜单，用户选择 | iPXE `menu` / `choose` |
| **MAC 绑定默认** | 某台机器固定进某系统 | Boot API 查询数据库，跳过菜单 |
| **上次选择记忆** | 记住用户上次选的系统 | Boot API 记录到数据库 |

### 6.2 iPXE 菜单示例

```ipxe
#!ipxe

dhcp
set mac ${net0/mac}

chain http://192.168.31.57/boot?mac=${mac}&arch=${arch}

menu 请选择要启动的操作系统 (MAC: ${mac})
item ubuntu     Ubuntu 26.04
item win10      Windows 10
item local      从本地硬盘启动
choose --default ubuntu --timeout 15000 os
chain http://192.168.31.57/boot?mac=${mac}&os=${os}
```

---

## 7. 网络设计（1~5台，千兆）

### 7.1 容量评估

| 场景 | 带宽需求 | 千兆能力 |
|------|----------|----------|
| 单客户端顺序读 | ~80-100 MB/s | ✅ 接近饱和 |
| 2~3 台并发启动 | ~200-300 MB/s = 1.6-2.4 Gbps | ✅ 可接受 |
| 5 台并发启动 | ~400-500 MB/s = 3.2-4 Gbps | ⚠️ 变慢但可用（错峰启动） |
| 日常运行（已缓存） | ~5-20 MB/s 每台 | ✅ 无压力 |

**结论**：千兆网络在 1~5 台场景下**完全可行**。并发启动时会变慢，建议错峰开机。

### 7.2 网络拓扑

```
┌─────────────────────────────────────────┐
│              千兆交换机                  │
│         （普通 8/16 口交换机）            │
└─────────────────┬───────────────────────┘
                  │
    ┌─────────────┼─────────────┐
    │             │             │
┌───▼───┐   ┌───▼───┐   ┌─────▼─────┐
│ 服务端 │   │ 客户1 │   │ 客户2~5   │
│(DHCP+  │   │       │   │           │
│ iSCSI) │   │       │   │           │
└────────┘   └───────┘   └───────────┘
```

### 7.3 服务端网卡建议

- 最低：单千兆网口（可用）
- 推荐：双千兆网口做 **LACP 链路聚合**（bonding），提供 2Gbps 带宽
- 存储与服务共用同一网卡即可（1~5 台场景流量不大）

---

## 8. 实施路线图（MVP 阶段）

**目标**：1~5 台客户端，支持 Ubuntu + Windows 切换，系统盘还原 + 用户数据持久化。

| 任务 | 说明 | 预计工作量 |
|------|------|-----------|
| 8.1 部署 iPXE | 编译带 HTTP/iSCSI 支持的 iPXE，配置链载 | 半天 |
| 8.2 搭建 LVM Thin Pool | 创建 thin pool + golden LV + 用户数据 LV | 半天 |
| 8.3 开发 Boot API | Python Flask：MAC 识别、snapshot 管理、数据盘管理 | 1~2 天 |
| 8.4 Linux 无盘验证 | 1 台客户端，验证 Ubuntu 从 snapshot 启动 + /home 持久化 | 1 天 |
| 8.5 Windows 镜像准备 | QEMU 安装 Windows → 注入驱动 → Sysprep → 导入 LVM | 2~3 天 |
| 8.6 Windows iSCSI Boot 验证 | 最关键的攻关点。验证 sanboot → Windows 正常启动 | 3~5 天 |
| 8.7 Windows 用户数据盘验证 | 验证 Windows 内自动连接 iSCSI 数据盘并挂载为 D: | 1 天 |
| 8.8 多系统切换联调 | Linux / Windows 切换，验证 snapshot 还原 + 数据保留 | 1 天 |

**验收标准**：
- [ ] 客户端开机看到菜单，可选 Ubuntu 或 Windows
- [ ] 选择 Ubuntu → 正常启动，/home 下创建文件 → 重启 → 系统干净但 /home 文件仍在
- [ ] 选择 Windows → 正常启动到桌面，D: 盘创建文件 → 重启 → 系统干净但 D: 文件仍在
- [ ] 关闭用户数据功能的客户端 → 全还原模式，重启后无任何残留

---

## 9. 项目文件规划

```
diskless-ubuntu/
├── README.md                    # 项目概述
├── ARCHITECTURE.md              # 本文件：1~5台架构设计
├── RESEARCH.md                  # 扩展性研究：10~50台、万兆、100台
├── DISKLESS-GUIDE.md            # 单系统 Ubuntu iSCSI 安装指南（基础参考）
│
├── scripts/                     # 自动化脚本
│   ├── install-deps.sh          # 服务端依赖安装
│   ├── build-ipxe.sh            # 编译 iPXE 固件
│   ├── create-golden.sh         # 创建 golden image thin LV
│   ├── boot-api.py              # Boot API 服务（MVP 版）
│   ├── snapshot-manager.sh      # 系统 snapshot 创建/删除
│   └── userdata-manager.sh      # 用户数据盘创建/扩容/备份
│
├── configs/                     # 配置文件模板
│   ├── dhcpd.conf               # DHCP 配置（指向 iPXE）
│   ├── ipxe/                    # iPXE 脚本模板
│   │   ├── menu.ipxe
│   │   ├── boot-linux.ipxe
│   │   └── boot-windows.ipxe
│   └── targetcli/               # iSCSI 配置模板
│
└── docs/                        # 详细操作文档
    ├── linux-golden-setup.md    # Linux 母盘制作
    ├── windows-golden-setup.md  # Windows 母盘制作（含驱动注入、数据盘脚本）
    └── troubleshooting.md       # 常见问题排查
```

---

## 10. 风险登记册

| 编号 | 风险 | 可能性 | 影响 | 应对策略 |
|------|------|--------|------|----------|
| R1 | Windows iSCSI Boot 在目标硬件上不稳定 | 中 | **高** | 提前用实际硬件做 PoC；准备 VDI 备选 |
| R2 | Windows 内用户数据盘自动连接失败 | 中 | 中 | 脚本添加重试和日志；首次启动手动引导 |
| R3 | LVM thin pool 空间不足 | 低 | 中 | 监控使用率；设置告警；预留 20% 空间 |
| R4 | 5 台同时启动时千兆网络拥塞 | 中 | 低 | 错峰启动；或双千兆聚合 |
| R5 | 客户端网卡不支持 iPXE 链载 | 低 | 中 | 测试所有目标机型；极少数可用 USB-iPXE |

---

## 附录 A：用户数据分离配置速查

| 配置项 | Linux | Windows |
|--------|-------|---------|
| 数据盘类型 | thin LV（非 snapshot） | thin LV（非 snapshot） |
| 连接时机 | initramfs 阶段 | 系统启动后（PowerShell 脚本） |
| 挂载点 | `/home` | `D:` |
| 文件系统 | ext4/xfs | NTFS |
| 是否持久 | ✅ 是 | ✅ 是 |
| 可选关闭 | ✅ Boot API 配置 | ✅ Boot API 配置 + 脚本不执行 |
| 扩容方式 | `lvextend` + `resize2fs` | `lvextend` + Windows 磁盘管理 |
