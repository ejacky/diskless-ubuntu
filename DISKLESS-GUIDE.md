# 无盘安装 Ubuntu 26.04（iSCSI）完整指南

本指南从零开始，在服务端搭建 DHCP + TFTP + NFS + iSCSI 环境，客户端通过 PXE 网络引导，将 Ubuntu 26.04 安装到远端 iSCSI LUN，最终实现无盘运行。

---

## 架构说明

```
┌───────────────────────────────────────────────────────────┐
│                      服务端 (192.168.31.57)                │
│                                                           │
│  DHCP (67/udp)      → 分配 IP、指定引导文件                 │
│  TFTP (69/udp)      → 传输 kernel 和 initrd               │
│  NFS  (2049/tcp)    → 导出 ISO，供客户端读取安装器文件      │
│  iSCSI (3260/tcp)   → 提供远端磁盘 LUN (20G)              │
└───────────────────────────────────────────────────────────┘
         │
         │ PXE 网络引导
         ▼
┌───────────────────────────────────────────────────────────┐
│                 客户端（无盘，空机）                         │
│                                                           │
│  1. PXE ROM → DHCP 获取 IP + bootloader                   │
│  2. TFTP 下载 kernel + initrd                             │
│  3. initrd 通过 NFS 读取 ISO 中的 squashfs（安装器）        │
│  4. 安装器启动，通过 iSCSI 参数发现远端 20G LUN            │
│  5. 将 Ubuntu 26.04 安装到 iSCSI LUN                      │
│  6. 安装完成重启 → 直接从 iSCSI LUN 引导系统（无盘运行）    │
└───────────────────────────────────────────────────────────┘
```

---

## 一、环境准备

### 1.1 硬件要求

- 服务端：已安装 Ubuntu 24.04（本指南以 24.04 为例），至少 30G 空闲磁盘
- 客户端：任意支持 PXE 网络引导的 x86_64 机器
- 网络：服务端与客户端在同一局域网，服务端 IP 固定为 `192.168.31.57`

### 1.2 下载 Ubuntu 26.04 ISO

```bash
wget -P /home/$USER/diskless https://releases.ubuntu.com/26.04/ubuntu-26.04-live-server-amd64.iso
```

---

## 二、安装必需软件包

```bash
sudo apt update
sudo apt install -y isc-dhcp-server tftpd-hpa nfs-kernel-server targetcli-fb
```

---

## 三、配置 iSCSI 存储

创建 20G 的 iSCSI LUN 作为客户端安装目标盘：

```bash
# 创建镜像文件
sudo dd if=/dev/zero of=/var/lib/iscsi_storage.img bs=1M count=20480

# 创建 iSCSI target
sudo targetcli /backstores/fileio create name=ubuntu-install file_or_dev=/var/lib/iscsi_storage.img

# 创建 iSCSI target portal
sudo targetcli /iscsi create iqn.2026-04.com.example:ubuntu-install

# 绑定 LUN
sudo targetcli /iscsi/iqn.2026-04.com.example:ubuntu-install/tpg1/luns create /backstores/fileio/ubuntu-install

# 允许所有 initiator 连接（生产环境应限制 ACL）
sudo targetcli /iscsi/iqn.2026-04.com.example:ubuntu-install/tpg1 set attribute authentication=0 generate_node_acls=1 demo_mode_write_protect=0

# 持久化配置（重启不丢失）
sudo targetcli saveconfig
```

> **说明**：`generate_node_acls=1` 允许任意客户端登录，`demo_mode_write_protect=0` 允许写入，`saveconfig` 将配置写入 `/etc/rtslib-fb-target/saveconfig.json` 保证重启后仍然生效。如果不做 saveconfig，重启服务后配置丢失，客户端会报 `authorization failure (error 24)`。

验证：

```bash
sudo iscsiadm -m discovery -t sendtargets -p 192.168.31.57:3260
# 应输出: 192.168.31.57:3260,1 iqn.2026-04.com.example:ubuntu-install
```

---

## 四、挂载 ISO 并配置 NFS

```bash
# 创建 NFS 导出目录
sudo mkdir -p /srv/nfs/ubuntu-26.04

# 挂载 ISO
sudo mount -o loop /home/$USER/diskless/ubuntu-26.04-live-server-amd64.iso /srv/nfs/ubuntu-26.04

# 配置 NFS 导出
echo '/srv/nfs/ubuntu-26.04 *(ro,no_subtree_check,no_root_squash)' | sudo tee /etc/exports

# 应用导出
sudo exportfs -ra

# 开机自动挂载 ISO
echo "/home/$USER/diskless/ubuntu-26.04-live-server-amd64.iso /srv/nfs/ubuntu-26.04 iso9660 loop,ro 0 0" | sudo tee -a /etc/fstab
```

验证：

```bash
showmount -e 127.0.0.1
# 应输出: /srv/nfs/ubuntu-26.04 *
```

---

## 五、配置 TFTP 引导文件

### 5.1 从 ISO 提取 kernel 和 initrd

```bash
sudo cp /srv/nfs/ubuntu-26.04/casper/vmlinuz /var/lib/tftpboot/amd64/linux
sudo cp /srv/nfs/ubuntu-26.04/casper/initrd  /var/lib/tftpboot/amd64/initrd
```

### 5.2 创建 BIOS 引导菜单

```bash
sudo mkdir -p /var/lib/tftpboot/amd64/pxelinux.cfg
sudo tee /var/lib/tftpboot/amd64/pxelinux.cfg/default << 'EOF'
DEFAULT ubuntu-nodisk-install
PROMPT 1
TIMEOUT 60

MENU TITLE PXE Boot Menu

LABEL ubuntu-nodisk-install
    MENU LABEL ^Install Ubuntu 26.04 to iSCSI (diskless boot)
    KERNEL linux
    APPEND initrd=initrd  boot=casper netboot=nfs nfsroot=192.168.31.57:/srv/nfs/ubuntu-26.04 ip=dhcp
EOF
```

> **说明**：
> - `KERNEL linux` 是相对路径，PXELINUX 会从自身所在目录（`amd64/`）下查找 `linux` 文件。
> - 这里**不配置 iSCSI 参数**。`boot=casper` 流程中 iSCSI 登录脚本不会执行，这些参数无效。iSCSI 磁盘由安装阶段手动连接（见 7.3 节）。

### 5.3 创建 UEFI 引导菜单

```bash
sudo mkdir -p /var/lib/tftpboot/amd64/grub
sudo tee /var/lib/tftpboot/amd64/grub/grub.cfg << 'EOF'
set timeout=60
set default=0

menuentry "Install Ubuntu 26.04 to iSCSI (diskless boot)" {
    set gfxpayload=keep
    linux   linux boot=casper netboot=nfs nfsroot=192.168.31.57:/srv/nfs/ubuntu-26.04 ip=dhcp ---
    initrd  initrd
}
EOF
```

> **说明**：这里同样**不配置 iSCSI 参数**。`boot=casper` 流程中 iSCSI 登录脚本不会执行，这些参数无效。iSCSI 磁盘由安装阶段手动连接（见 7.3 节）。

### 5.4 TFTP 最终目录结构

```
/var/lib/tftpboot/
└── amd64/
    ├── bootx64.efi          # UEFI 引导器（来自 GRUB）
    ├── grubx64.efi          # GRUB UEFI 二进制
    ├── grub/
    │   └── grub.cfg         # UEFI 引导菜单
    ├── initrd               # 安装器 initrd（来自 ISO /casper/initrd）
    ├── ldlinux.c32          # PXELINUX 库文件
    ├── linux                # 安装内核（来自 ISO /casper/vmlinuz）
    ├── pxelinux.0           # BIOS 引导器
    └── pxelinux.cfg/
        └── default          # BIOS 引导菜单
```

---

## 六、配置 DHCP

```bash
sudo tee /etc/dhcp/dhcpd.conf << 'EOF'
option domain-name "pxe.local";
option domain-name-servers 8.8.8.8, 8.8.4.4;

default-lease-time 600;
max-lease-time 7200;

# 指定 TFTP 服务器地址
next-server 192.168.31.57;

# 子网配置
subnet 192.168.31.0 netmask 255.255.255.0 {
    range 192.168.31.150 192.168.31.200;
    option routers 192.168.31.1;
    option subnet-mask 255.255.255.0;
    # 兜底：未匹配到 efi/bios class 的客户端默认使用 PXELINUX
    filename "amd64/pxelinux.0";
}

# UEFI 客户端 → 使用 GRUB
class "efi" {
    match if substring(option vendor-class-identifier, 0, 20) = "PXEClient:Arch:00007";
    filename "amd64/bootx64.efi";
}

# BIOS 客户端 → 使用 PXELINUX
class "bios" {
    match if substring(option vendor-class-identifier, 0, 9) = "PXEClient" and
         not (substring(option vendor-class-identifier, 9, 2) = ":A");
    filename "amd64/pxelinux.0";
}
EOF

sudo systemctl restart isc-dhcp-server
```

---

## 七、客户端安装

### 7.1 确认服务端就绪

```bash
# 所有服务均为 active (running)
sudo systemctl status isc-dhcp-server tftpd-hpa nfs-server targetclid

# 验证 iSCSI target 可发现
sudo iscsiadm -m discovery -t sendtargets -p 192.168.31.57:3260
```

### 7.2 PXE 启动

1. 客户端进入 BIOS/UEFI，将启动顺序设为 Network / PXE Boot 优先
2. 启动后自动获取 IP 地址，显示 PXE Boot Menu
3. 选择 `Install Ubuntu 26.04 to iSCSI (diskless boot)`，回车

### 7.3 连接 iSCSI 磁盘

Ubuntu 安装器启动后，默认只显示本地硬盘。需要手动连接远端 iSCSI LUN：

1. 等待安装器进入第一个交互页面（语言选择或键盘布局）
2. 按 `Ctrl + Alt + F2` 切换到终端
3. 执行：

```bash
sudo iscsiadm -m discovery -t sendtargets -p 192.168.31.57:3260
sudo iscsiadm -m node --login
```

4. 输出 `successful` 即登录成功
5. 按 `Ctrl + Alt + F1` 切回安装界面，继续安装

> **注意**：如果登录时报 `authorization failure (error 24)`，回到服务端执行 `sudo targetcli saveconfig` 并确认 `generate_node_acls=1`。

### 7.4 选择安装目标磁盘

1. 安装器进入磁盘选择界面
2. 如果 iSCSI LUN 没有立即出现，选择 **Back** 退回上一步再进来即可刷新
3. 会看到一个约 20G 的新磁盘——这就是远端 iSCSI LUN
4. **只勾选这个 iSCSI 磁盘**，取消本地硬盘的勾选
5. 继续完成安装（分区、用户、SSH 等按需选择）

### 7.5 安装完成后的引导方式

系统安装完成重启后，有两种引导方式。

> **⚠️ 关键前提**：无论选哪种方式，都必须先**替换 TFTP 中的内核和 initrd**。ISO 自带的 `vmlinuz` / `initrd` 是 casper live 环境，只能启动安装器，不处理 `root=` / `netroot=` 参数。必须换用 iSCSI LUN 上已安装系统自己构建的内核和 initrd。

#### 7.5.1 提取已安装系统的内核和 initrd

```bash
# 1. 服务端连接自己导出的 iSCSI LUN
sudo iscsiadm -m discovery -t sendtargets -p 192.168.31.57:3260
sudo iscsiadm -m node --login

# 2. 确认 iSCSI 磁盘（20G 且有分区的那个）
lsblk

# 3. 挂载 /boot 分区（通常是 sdb2 或 sdc2，1.8G 那个）
sudo mount /dev/sdb2 /mnt

# 4. 查看文件
ls /mnt/

# 5. 用已安装系统的内核和 initrd 替换 TFTP 文件
sudo cp /mnt/vmlinuz-*    /var/lib/tftpboot/amd64/linux
sudo cp /mnt/initrd.img-* /var/lib/tftpboot/amd64/initrd

# 6. 清理
sudo umount /mnt
sudo iscsiadm -m node --logout
```

> **说明**：ISO 的 `/casper/initrd` 由 casper-live 构建，主脚本是 casper 的 `/init`，只走 live-boot 流程。已安装系统的 `initrd.img-*` 由 initramfs-tools 构建，其 `/init` 会处理 `root=` / `netroot=` 参数，并通过 `/scripts/local-top/iscsi` 自动建立 iSCSI 连接。

#### 7.5.2 确定根分区参数

安装时如果选了 LVM（Ubuntu 默认），根分区在 LVM 逻辑卷上。需要获取其 UUID：

在客户端 dracut shell 或已挂载系统中执行：

```bash
# 激活 LVM（如果是 LVM）并查看 LV
lvm pvs && lvm vgs && lvm lvs
ls /dev/mapper/
blkid /dev/ubuntu-vg-1/ubuntu-lv   # 按实际 LV 路径调整
```

记下 UUID 和 LV 路径。如果是普通 ext4 分区，直接用分区的 UUID 即可。

#### 7.5.3 配置 PXE 菜单

根据根分区类型选择对应的内核参数：

**情况 A：根分区在 LVM 逻辑卷（最常见）**

```bash
# BIOS 菜单 (/var/lib/tftpboot/amd64/pxelinux.cfg/default)：
LABEL boot-from-iscsi
    MENU LABEL ^Boot Ubuntu from iSCSI
    KERNEL linux
    APPEND initrd=initrd root=UUID=<LV的UUID> netroot=iscsi:192.168.31.57:6:3260:0:iqn.2026-04.com.example:ubuntu-install ip=dhcp
```

```bash
# UEFI 菜单 (/var/lib/tftpboot/amd64/grub/grub.cfg)：
menuentry "Boot Ubuntu from iSCSI" {
    set gfxpayload=keep
    linux   linux root=UUID=<LV的UUID> netroot=iscsi:192.168.31.57:6:3260:0:iqn.2026-04.com.example:ubuntu-install ip=dhcp ---
    initrd  initrd
}
```

**情况 B：根分区是普通 ext4/xfs 分区（非 LVM）**

```bash
# BIOS 菜单：
LABEL boot-from-iscsi
    MENU LABEL ^Boot Ubuntu from iSCSI
    KERNEL linux
    APPEND initrd=initrd  root=iscsi:192.168.31.57:6:3260:0:iqn.2026-04.com.example:ubuntu-install ip=dhcp
```

#### 7.5.4 参数说明

| 参数 | 作用 | 示例 |
|------|------|------|
| `root=UUID=<uuid>` | 指定真正的根文件系统（LVM LV 或分区的 UUID） | `root=UUID=16225cea-23fe-452b-ba35-6116864050e1` |
| `netroot=iscsi:<ip>:6:<port>:<lun>:<iqn>` | 告诉 dracut 通过 iSCSI 建立网络存储连接 | `netroot=iscsi:192.168.31.57:6:3260:0:iqn.2026-04.com.example:ubuntu-install` |
| `root=iscsi:<ip>:6:<port>:<lun>:<iqn>` | 直接把整个 iSCSI LUN 作为根设备（仅当根在整盘或非 LVM 分区时用） | （见情况 B） |
| `ip=dhcp` | DHCP 获取客户端 IP | 必选 |

> **关键区别**：`netroot=` 负责网络存储连接建立，`root=` 负责指定挂载哪个文件系统。LVM 场景下根分区不在 iSCSI LUN 的第一层，必须拆开写——`netroot=iscsi:` 建连接 + `root=UUID=` 指 LV。

**方式一：客户端直接 iSCSI 引导（推荐，需网卡支持 iBFT）**

1. 重启时进入 BIOS/UEFI 设置
2. 启动顺序中添加 iSCSI / Network Boot 条目
3. 配置 iSCSI 参数：Target IP `192.168.31.57`，Port `3260`，LUN `0`，IQN `iqn.2026-04.com.example:ubuntu-install`
4. 保存退出，客户端直接从 iSCSI LUN 加载系统

**方式二：PXE 二次引导 iSCSI（通用，任何支持 PXE 的网卡都行）**

按 7.5.1→7.5.3 步骤配置 PXE 菜单。客户端 PXE 启动后选择 "Boot Ubuntu from iSCSI" 即可。

---

## 八、常见问题

| 现象 | 原因 | 解决 |
|------|------|------|
| 客户端停在 `PXE-E53: No boot filename received` | DHCP 未响应或 class 匹配失败 | 检查 `isc-dhcp-server` 状态和 `dhcpd.conf` 子网配置 |
| `Loading linux... failed: No such file or directory` | TFTP 路径错误 | PXELINUX 中 `KERNEL` 路径是相对于 `pxelinux.0` 所在目录，不要加 `amd64/` 前缀 |
| `Loading linux... ok` 后立刻 `kernel panic` | initrd 不完整、参数错误、或 NFS 不通 | 检查 initrd 是否从 ISO `/casper/initrd` 直接复制；检查 NFS 导出；检查 `boot=casper netboot=nfs nfsroot=` 参数 |
| iSCSI 登录报 `authorization failure (error 24)` | target ACL 未配置或重启后丢失 | 服务端执行 `sudo targetcli saveconfig`，确认 `generate_node_acls=1` |
| 安装器里看不到 iSCSI 磁盘 | 未手动登录 iSCSI | 按 `Ctrl+Alt+F2` 进入终端，执行 `sudo iscsiadm -m node --login`，切回后刷新磁盘列表 |
| `tftp: client does not accept options` | 正常告警，不影响传输 | 忽略 |
| 客户端报 `Unable to find a medium containing a live file system` | 使用了 ISO 的 casper initrd，它只认识 live boot 流程 | 替换为已安装系统的 initrd（见 7.5.1） |
| 客户端进入 dracut emergency shell，`Can't mount root file` | ① root 在 LVM 上但只用 `root=iscsi:`；② iSCSI 连接慢，initrd 等不及 | ① 改为 `root=UUID=<LV_UUID> netroot=iscsi:...`（见 7.5.3）；② 无需操作，手动连接后 `exit` 即可 |
| iSCSI 登录报 `authorization failure (error 24)` | target ACL 未配置或重启后丢失 | 服务端执行 `sudo targetcli saveconfig`，确认 `generate_node_acls=1` |
| dracut shell 中 `mount: unknown filesystem type 'LVM2_member'` | 直接 mount 了 LVM PV，需要先激活 LVM | 执行 `lvm pvs && lvm vgs && lvm lvs` 确认结构，然后 mount LV 路径 |

---

## 九、参数速查

### 内核命令行参数

| 参数 | 含义 | 示例 |
|------|------|------|
| `boot=casper` | 使用 casper live 引导系统 | 必选（安装阶段） |
| `netboot=nfs` | 从 NFS 获取 squashfs | 必选（安装阶段） |
| `nfsroot=<ip>:<path>` | NFS 服务器地址和路径 | `192.168.31.57:/srv/nfs/ubuntu-26.04` |
| `ip=dhcp` | DHCP 获取客户端 IP | 必选 |
| `netroot=iscsi:<ip>:<proto>:<port>:<lun>:<iqn>` | 通过 iSCSI 建立网络存储连接（告诉 dracut） | `netroot=iscsi:192.168.31.57:6:3260:0:iqn.2026-04.com.example:ubuntu-install` |
| `root=UUID=<uuid>` | 指定根文件系统 UUID（与 `netroot=iscsi:` 配合使用，支持 LVM LV） | `root=UUID=16225cea-23fe-452b-ba35-6116864050e1` |
| `root=iscsi:<ip>:<proto>:<port>:<lun>:<iqn>` | 直接把整个 iSCSI LUN 作为根设备（仅非 LVM 场景） | `root=iscsi:192.168.31.57:6:3260:0:iqn.2026-04.com.example:ubuntu-install` |

> **关于 iSCSI 参数**：内核命令行支持 `iscsi_target_ip`、`iscsi_target_port`、`iscsi_target_name` 等参数，但这些参数由 `/scripts/local-top/iscsi` 脚本处理。在 `boot=casper` 流程中，该脚本**不会被执行**，所以这些参数放在这里不会生效。iSCSI 磁盘连接在安装阶段手动完成（见 7.3 节）。

### 关键路径

| 路径 | 说明 |
|------|------|
| `/srv/nfs/ubuntu-26.04/` | ISO 挂载点 + NFS 导出目录 |
| `/var/lib/tftpboot/amd64/linux` | 安装内核（从 ISO `/casper/vmlinuz` 复制） |
| `/var/lib/tftpboot/amd64/initrd` | 安装器 initrd（从 ISO `/casper/initrd` 直接复制） |
| `/var/lib/iscsi_storage.img` | iSCSI 存储镜像（20G） |
| `/etc/dhcp/dhcpd.conf` | DHCP 配置 |
| `/etc/exports` | NFS 导出配置 |
