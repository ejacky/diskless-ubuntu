# 知识储备分析：从 DISKLESS-GUIDE 到 ARCHITECTURE 实施

> 基于已有基础：`DISKLESS-GUIDE.md` 已完整跑通（单 Ubuntu + PXELINUX/GRUB + fileio iSCSI）  
> 目标：`ARCHITECTURE.md` 完整实施（多系统 + iPXE + LVM Thin Snapshot + Windows Boot）

---

## 1. 已有知识盘点（来自 DISKLESS-GUIDE）

| 领域 | 已掌握内容 | 熟练度参考 |
|------|-----------|-----------|
| **网络引导** | DHCP 分配 IP + 指向 PXELINUX/GRUB；TFTP 传输内核/initrd | ✅ 已实操 |
| **iSCSI Target** | targetcli-fb 创建 fileio backstore、导出 LUN、ACL 配置、saveconfig | ✅ 已实操 |
| **iSCSI Initiator** | `iscsiadm` 发现/登录/登出；内核 `netroot=iscsi:` 参数 | ✅ 已实操 |
| **NFS** | 挂载 ISO、exportfs 导出、内核 `netboot=nfs nfsroot=` | ✅ 已实操 |
| **PXE 菜单** | PXELINUX `pxelinux.cfg/default`、GRUB `grub.cfg` 语法 | ✅ 已实操 |
| **内核启动参数** | `root=UUID=`、`netroot=iscsi:`、`ip=dhcp`、dracut 处理流程 | ✅ 已理解 |
| **LVM 基础** | `pvs/vgs/lvs` 查看；安装器创建的 LV 结构；`blkid` 获取 UUID | ⚠️ 浅层（安装器自动创建） |
| **Initramfs 概念** | 知道 casper initrd vs initramfs-tools initrd 的区别；dracut emergency shell | ⚠️ 概念级 |

---

## 2. 新增知识缺口总览

将 ARCHITECTURE.md 的实施拆为 7 大技术域，逐项对比已有基础：

```
已有基础 ────────────────────────────────────────► 目标架构
  │                                                    │
  ├── DHCP/TFTP/NFS/iSCSI 基础 ────────────────────────┤ ✅ 可直接复用
  ├── PXELINUX/GRUB 静态菜单 ──────────────────────────┤ ❌ 完全替换为 iPXE
  ├── fileio backstore（磁盘镜像文件）──────────────────┤ ❌ 替换为 block backstore + LVM Thin
  ├── 单系统单客户端 ──────────────────────────────────┤ ❌ 扩展为多系统 + 每客户端独立存储
  ├── Linux only ──────────────────────────────────────┤ ❌ 新增 Windows iSCSI Boot
  ├── 手动配置（逐条命令）─────────────────────────────┤ ❌ 新增 Boot API 自动化
  └── 无持久化数据设计 ────────────────────────────────┤ ❌ 新增 系统盘/数据盘双盘分离
```

---

## 3. 七大新增技术域详解

### 3.1 iPXE — 替换 PXELINUX/GRUB（工作量：中）

**为什么必须换**：ARCHITECTURE 的动态菜单、HTTP 传输、Windows sanboot 都无法用 PXELINUX/GRUB 实现。

**已有基础可复用**：
- DHCP 中 `next-server` / `filename` 的概念（但值要改成指向 iPXE 固件）
- 内核启动参数（`root=`、`netroot=`、`ip=dhcp`）在 iPXE 脚本中同样适用

**需要新学的**：

| 知识点 | 说明 | 学习资源方向 |
|--------|------|-------------|
| iPXE 编译 | 从源码编译 `ipxe.pxe`（BIOS）和 `ipxe.efi`（UEFI），需启用 HTTP、iSCSI 支持 | iPXE 官方 `make` 文档 |
| 链载（Chainloading） | 网卡 ROM → TFTP 下载 `undionly.kpxe` / `ipxe.efi` → iPXE 接管 → 再 HTTP 请求脚本 | iPXE 官方 chainloading 指南 |
| iPXE 脚本语言 | `dhcp`、`set`、`chain`、`kernel`、`initrd`、`boot`、`menu`、`choose`、`goto` | iPXE 脚本参考手册 |
| sanboot / sanhook | Windows iSCSI 启动的核心命令：`sanboot iscsi:<ip>::<port>:<lun>:<iqn>` | iPXE 命令参考 |
| 嵌入脚本（Embedded Script） | 将默认 iPXE 脚本编译进固件，实现开机自动 `dhcp` + `chain http://.../boot` | iPXE `EMBED=` 编译选项 |

**实操建议**：先搭一个最小 iPXE 环境——iPXE 固件链载成功后，HTTP 拉取一个简单的 "hello world" 脚本（显示菜单 + 本地启动），验证整个链路。

---

### 3.2 LVM Thin Provisioning + Snapshot（工作量：高）

这是 ARCHITECTURE 的**存储核心**，与 DISKLESS-GUIDE 的 `fileio` 方案差异最大。

**已有基础可复用**：
- `pvcreate` / `vgcreate` / `lvcreate` 命令格式（但 Thin 参数不同）
- `targetcli` 的基本操作路径

**需要新学的**：

| 知识点 | 说明 | 关键命令/概念 |
|--------|------|--------------|
| Thin Pool 创建与管理 | `lvcreate -L 500G -T vg/thin-pool`，理解 over-provisioning | `lvs -a` 查看 pool 使用率 |
| Thin LV（虚拟大小 vs 实际分配） | `-V 100G` 是虚拟大小，实际只占用写入的数据量 | `lvs` 中 `LSize` vs `Data%` |
| Thin Snapshot 机制 | `lvcreate -s` 从 thin LV 创建 snapshot，COW（写时复制）原理 | 写入时原卷不变，差异数据写到 snapshot 的 COW 区 |
| Snapshot 合并与删除 | 重启后 `lvremove` 旧 snapshot + `lvcreate -s` 新 snapshot 实现"还原" | 这是"纯还原"的技术基础 |
| block backstore | targetcli 不再用 `fileio`，改用 `/backstores/block/` 直接暴露 `/dev/mapper/...` | `targetcli /backstores/block create ... dev=/dev/vg/lv` |
| Thin Pool 耗尽风险 | Thin pool 空间满时，snapshot 会损坏且不可逆 | 监控 `Data%`，预留 20%+ 空间 |

**关键理解**：`fileio` 是一个文件被 iSCSI 暴露为磁盘；`block` 是直接暴露块设备。Thin Snapshot 让"每个客户端有独立的可写系统盘"成为可能——原卷只读，snapshot 可写，删除 snapshot 即回到原卷状态。

**实操建议**：在测试机上先玩一遍：
1. 创建 thin pool → thin LV → 写入数据
2. 创建 snapshot → 在 snapshot 上修改/删除文件 → 对比原卷不变
3. 删除 snapshot → 重新创建 → 验证回到初始状态
4. targetcli 用 block backstore 导出 snapshot，从另一台机器 iSCSI 连接验证

---

### 3.3 Boot API — 自动化控制层（工作量：中）

DISKLESS-GUIDE 是手动敲命令；ARCHITECTURE 要求按客户端 MAC 动态创建 snapshot、管理 target。

**已有基础可复用**：
- 知道 targetcli、LVM 的命令行操作（需要把它们封装进代码）
- 理解启动流程的时序（先 DHCP → 再 HTTP 请求 → 再 iSCSI 连接 → 再启动）

**需要新学的**：

| 知识点 | 说明 | 技术方向 |
|--------|------|----------|
| Python Web 框架 | Flask 或 FastAPI，单文件运行即可 | 选一个熟悉的，推荐 FastAPI（自带 OpenAPI 文档）|
| 子进程调用 | Python 调用 `lvcreate` / `lvremove` / `targetcli` 并捕获输出/错误码 | `subprocess.run()` + 错误处理 |
| MAC 地址解析与归一化 | DHCP 请求中 MAC 格式可能有 `:` / `-` / 无分隔符，需统一处理 | 正则清洗 + 格式化 |
| iPXE 脚本动态生成 | 根据客户端配置拼接字符串返回 `Content-Type: text/plain` | 模板字符串或 Jinja2 |
| 简单的配置持久化 | 客户端 MAC → 默认 OS / 数据盘开关 / 数据盘大小 的映射 | SQLite（最简单）或 JSON 文件 |
| 并发安全 | 多个客户端同时开机，同时请求 Boot API 创建 snapshot | 考虑文件锁或数据库事务 |

**关键设计决策**：
- Boot API 是否只在启动时被调用一次？还是分两次（第一次返回菜单，第二次准备存储）？ARCHITECTURE 中设计了 `/boot?mac=` 返回菜单 + `/api/prepare?mac=&os=` 准备存储，这是合理的分离。
- 如果 Boot API 宕机，客户端是否有降级方案？建议考虑本地缓存脚本或默认菜单。

**实操建议**：
1. 先用 Flask 写一个 `GET /boot?mac=...` 返回固定 iPXE 脚本的 API
2. 把脚本中的 `lvcreate` / `targetcli` 命令替换为 Python `subprocess` 调用
3. 加入 SQLite 存储客户端配置
4. 测试多 MAC 并发请求

---

### 3.4 Linux 多盘启动 — rootfs + /home 分离（工作量：中）

DISKLESS-GUIDE 只挂了一个 iSCSI LUN 作为根分区。ARCHITECTURE 要求同时挂两个：系统盘（snapshot）+ 数据盘（持久化 thin LV）。

**已有基础可复用**：
- `netroot=iscsi:` 参数（系统盘仍用它）
- `/etc/fstab` 的基本语法

**需要新学的**：

| 知识点 | 说明 | 技术方向 |
|--------|------|----------|
| 多 iSCSI Target 连接 | 同一客户端连接两个 target：系统盘 + 数据盘 | iPXE 脚本中分别 `set` 两个 IQN |
| initramfs 阶段挂载第二块盘 | 数据盘需在 initramfs 中连接并挂载到 `/home`，否则 systemd 启动后用户目录已初始化 | 自定义 initramfs hook（`local-top` 阶段）|
| dracut 自定义 hook | 在 `/usr/lib/dracut/modules.d/` 添加模块，让 initrd 处理自定义参数（如 `userdata=iscsi:...`） | dracut 模块开发文档 |
| 内核参数传递第二块盘 | iPXE 脚本中把数据盘参数写进内核命令行，initramfs hook 解析 | 类似 `netroot=` 的处理方式 |

**关键难点**：
- dracut 默认只处理 `netroot=`（系统盘）。数据盘需要你自己写一个 hook，在 `local-top` 阶段执行 `iscsiadm` 连接数据盘 target。
- 如果 initramfs 不处理数据盘，也可以让 systemd 启动后再 mount，但此时 `/home` 可能已有内容（从系统盘 snapshot 来的），mount 上去会覆盖，导致用户无法登录（如果系统盘 `/home` 是空的）。
- **推荐方案**：在 initramfs 阶段连接数据盘并挂载 `/home`，确保系统启动时 `/home` 已经是持久化盘。

**实操建议**：
1. 先不用 Boot API，手动创建两个 block backstore target
2. 手动修改 iPXE 脚本连接两个 target
3. 手动修改 initramfs 或 `/etc/fstab` 挂载 `/home`
4. 验证：在 `/home` 写文件 → 重启 → 文件仍在，系统盘已还原

---

### 3.5 Windows iSCSI Boot — 最大知识缺口（工作量：很高）

这是 ARCHITECTURE 中风险最高（R1）、工作量最大（3~5 天验证）的部分。DISKLESS-GUIDE 完全没有涉及。

**已有基础**：无直接复用。间接相关的是 iSCSI Target 配置经验。

**需要新学的**：

| 知识点 | 说明 | 技术方向 |
|--------|------|----------|
| QEMU/KVM 安装 Windows | 创建 raw 虚拟磁盘、QEMU 参数（`-machine q35`、`-accel kvm`、VirtIO 驱动）| QEMU 文档、VirtIO-win ISO |
| VirtIO 驱动 | QEMU 默认磁盘/网卡是 VirtIO，Windows 安装盘不自带驱动，需额外加载 | 下载 `virtio-win.iso` |
| Windows iSCSI Initiator 服务 | `msiscsi` 服务设置为自动启动；iSCSI 发现/登录/持久化连接 | PowerShell `Get-Service`、`New-IscsiTargetPortal`、`Connect-IscsiTarget` |
| Sysprep 通用化 | `sysprep /generalize /oobe /shutdown` 移除硬件绑定，为分发到不同物理机做准备 | Windows 部署文档 |
| 驱动注入（Driver Injection） | 在母盘中预装 Intel/Realtek/Broadcom 等物理机网卡驱动，防止蓝屏 | `pnputil -i -a`、DISM |
| iPXE sanboot | `sanboot iscsi:<ip>::<port>:<lun>:<iqn>` 命令 | iPXE 文档 |
| Windows 内二次 iSCSI 连接 | sanboot 只连接系统盘，数据盘需在 Windows 启动后用内置 iSCSI Initiator 连接 | PowerShell iSCSI cmdlet |
| Windows 磁盘初始化/分区/格式化 | 首次连接数据盘时需要初始化 GPT、创建分区、格式化为 NTFS | `Initialize-Disk`、`New-Partition`、`Format-Volume` |
| Windows 激活（KMS） | 硬件通用化后激活可能失效，需 KMS 批量激活 | 如适用 |

**关键风险与应对**：

| 风险 | 知识要求 |
|------|----------|
| 物理机网卡驱动缺失导致 7B 蓝屏 | 必须掌握驱动注入或 Sysprep 后的 PnP 驱动自动安装 |
| iSCSI 连接从 iPXE 移交给 Windows 失败 | 理解 iPXE `keep-san` 选项；测试不同网卡的兼容性 |
| Windows 更新导致母盘膨胀 | 掌握离线维护 snapshot / 重建 golden image 的流程 |

**实操建议**：
1. **先在 QEMU 内验证 Windows iSCSI Boot**：
   - 服务端创建 iSCSI target → QEMU 用 iSCSI 直接启动 Windows（不经过 iPXE，验证 Windows 能否从 iSCSI 正常启动）
2. **再叠加 iPXE 链路**：
   - iPXE → sanboot → Windows 从 iSCSI 启动
3. **最后加入数据盘**：
   - Windows 启动后自动连接第二个 iSCSI target 并挂载为 D:
4. **最后迁移到物理机**：
   - 这是最大风险点，蓝屏概率高，准备好驱动包和调试手段

---

### 3.6 多系统切换与客户端管理（工作量：中）

**已有基础**：无。DISKLESS-GUIDE 只有一个系统。

**需要新学的**：

| 知识点 | 说明 |
|--------|------|
| 多个 Golden Image 管理 | Ubuntu golden、Win10 golden、Win11 golden 的命名、版本控制、空间规划 |
| 每客户端独立 snapshot | MAC 地址归一化后作为 LV 名的一部分（如 `client-aabbccddeeff-ubuntu`）|
| 菜单动态生成 | 根据客户端配置决定显示哪些系统选项、默认选项、超时时间 |
| "上次选择记忆"（可选） | Boot API 中记录客户端上次选择的 OS，下次默认选中 |

---

### 3.7 运维与故障排查（工作量：中，但长期重要）

**需要新学的**：

| 知识点 | 说明 | 技术方向 |
|--------|------|----------|
| Thin Pool 监控 | `lvs -a` 中 `Data%` 字段，超过阈值告警 | 简单脚本 + cron 或 systemd timer |
| Boot API 日志 | snapshot 创建/删除、target 绑定/解绑的操作日志 | Python `logging` 模块 |
| iPXE 调试 | iPXE 内置 `imgstat`、`route`、`ifstat` 等命令排查网络/脚本问题 | iPXE 调试模式（Ctrl+B 进入 shell）|
| dracut emergency shell 进阶 | 当 Linux 无法挂载 root 时，如何手动连接 iSCSI、激活 LVM、继续启动 | 经验积累 |
| Windows 蓝屏调试 | 蓝屏代码（如 0x7B INACCESSIBLE_BOOT_DEVICE）、启用内核调试 | 物理机调试或 QEMU 串口日志 |

---

## 4. 知识储备优先级矩阵

按「实施阻塞程度 × 学习曲线陡峭度」排序：

```
                    学习曲线陡峭
                         ▲
                         │
    Windows iSCSI Boot   │   ████████████████████  ← 最高优先级+最陡峭
    LVM Thin Snapshot    │   ██████████████
    iPXE 编译与脚本      │   ██████████
    Linux 多盘 initramfs │   ████████
    Boot API 开发        │   ██████
    多系统管理           │   ████
    运维监控             │   ██
                         │
    ─────────────────────┼────────────────────────► 实施阻塞程度
                         │
```

**建议学习顺序**：

| 阶段 | 学习内容 | 预计时间 | 产出 |
|------|----------|----------|------|
| **Phase 1** | LVM Thin Pool + Snapshot 原理与实操 | 1 天 | 能在测试机上手动完成"创建→修改→删除→重建"snapshot 全流程 |
| **Phase 2** | iPXE 编译 + 链载 + 脚本基础 | 0.5~1 天 | 客户端能链载 iPXE，HTTP 拉取菜单脚本，成功启动本地/网络系统 |
| **Phase 3** | Boot API MVP（Flask + LVM/targetcli 封装）| 1~2 天 | 按 MAC 自动创建 snapshot、返回 iPXE 脚本 |
| **Phase 4** | Linux 多盘启动（系统盘 + /home 数据盘）| 1~2 天 | Ubuntu 无盘运行，/home 持久化，重启系统盘还原 |
| **Phase 5** | Windows 母盘制作（QEMU + Sysprep + 驱动注入）| 2~3 天 | 得到一个可启动的 Windows iSCSI LUN |
| **Phase 6** | Windows iSCSI Boot 验证（iPXE sanboot）| 3~5 天 | 物理机能从 iPXE sanboot 进入 Windows 桌面 |
| **Phase 7** | Windows 数据盘自动连接 + 联调 | 1~2 天 | D: 盘持久化，多系统切换正常 |
| **Phase 8** | 监控、日志、文档收尾 | 1 天 | thin pool 告警、Boot API 日志、故障排查手册 |

---

## 5. 每个阶段的"最小验证标准"

建议每完成一个 Phase，就用以下 checklist 验证，不通过不进入下一阶段：

- [ ] **Phase 1**：`lvs` 能看到 thin pool 和 snapshot；snapshot 上写 1GB 文件后 `Data%` 增长；删除 snapshot 后重新创建，文件消失。
- [ ] **Phase 2**：客户端 iPXE 固件链载成功；HTTP 请求返回的脚本能被 iPXE 正确解析并执行到 `boot` 命令。
- [ ] **Phase 3**：改一个客户端的 MAC，Boot API 能返回不同的 iPXE 脚本；重启两次，snapshot 被正确删除重建。
- [ ] **Phase 4**：`/home` 下 `echo "test" > marker.txt` → 重启 → `cat /home/marker.txt` 输出 `test`；`/` 下创建文件 → 重启 → 文件消失。
- [ ] **Phase 5**：QEMU 从 iSCSI 启动 Windows 成功（不经过 iPXE，先验证 Windows 本身）。
- [ ] **Phase 6**：物理机从 iPXE → sanboot → Windows 桌面，无蓝屏，网卡/显卡驱动正常。
- [ ] **Phase 7**：D: 盘创建文件 → 重启 → 文件仍在；切换到 Ubuntu → 再切回 Windows → D: 文件仍在。

---

## 6. 总结

从 DISKLESS-GUIDE 到 ARCHITECTURE 的跨越，本质上是三个维度的升级：

| 维度 | DISKLESS-GUIDE | ARCHITECTURE | 核心新知识 |
|------|---------------|--------------|-----------|
| **引导器** | PXELINUX/GRUB 静态菜单 | iPXE 动态脚本 | iPXE 编译、脚本语言、链载 |
| **存储** | fileio 单 LUN | LVM Thin Pool + per-client snapshot | Thin Provisioning、COW、snapshot 生命周期 |
| **系统** | 单 Ubuntu | Ubuntu + Windows 多系统 | Windows iSCSI Boot、sanboot、驱动注入、Sysprep |
| **自动化** | 手动命令 | Boot API 动态编排 | Python Web 开发、子进程管理、MAC 识别 |
| **数据** | 无分离 | 系统盘/数据盘分离 | initramfs hook、多 iSCSI target、Windows 二次连接 |

**最大风险点**：Windows iSCSI Boot 在物理机上的稳定性（R1）。建议 Phase 5~6 尽早启动，不要在 Linux 侧 polish 完才开始碰 Windows——如果物理机 Windows iSCSI Boot 走不通，整个架构需要重新评估。
