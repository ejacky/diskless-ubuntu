# rCore x86_64 无盘 PXE 启动 Demo 指南

本指南把 [rcore-os/rCore](https://github.com/rcore-os/rCore) 的 `x86_64/pc` 版本通过 **DHCP + TFTP** 网络启动跑起来。与 `DISKLESS-GUIDE.md` 中 Ubuntu 那套相比，rCore 的 PC 版会把用户程序镜像直接链进内核 ELF，因此不需要 NFS/iSCSI，只需要把 **一个自包含的 EFI 引导文件** 下发给客户端即可。

---

## 一、整体思路

```
┌─────────────────────────────────────┐
│  服务端 (192.168.31.57)              │
│  DHCP → 分配 IP + 指定 BootX64.efi  │
│  TFTP → 下发 BootX64.efi            │
└─────────────────────────────────────┘
           │
           │ UEFI PXE 启动
           ▼
┌─────────────────────────────────────┐
│  客户端（无盘，空机）                 │
│  1. UEFI PXE ROM 通过 DHCP 拿到 IP   │
│  2. 通过 TFTP 下载 BootX64.efi       │
│  3. 运行 rboot（内核 ELF 已内嵌）    │
│  4. rboot 建立页表并跳入 rCore 内核  │
│  5. 内核使用编译期链入的用户镜像运行  │
└─────────────────────────────────────┘
```

关键点：

- rCore `x86_64` 的 UEFI 引导器是 `rboot`。
- 正常 `rboot.efi` 启动时会从 FAT 文件系统读取 `\EFI\rCore\kernel.elf`。
- UEFI PXE 下载下来的 `BootX64.efi` 通常**没有附带可访问的文件系统**，所以必须把 `kernel.elf` 和 `rboot.conf` 在编译期嵌入到 `rboot.efi` 内部，变成单一文件。
- `BOARD=pc` 会启用 `link_user` feature，用户程序镜像会被编进内核 ELF，运行时不再需要额外磁盘。

---

## 二、前置条件

| 工具/环境 | 说明 |
|---|---|
| Rust `nightly-2020-06-04` | rCore 仓库根目录的 `rust-toolchain` 已指定该版本 |
| `rust-src`、`llvm-tools-preview` | 编译 no_std 内核需要 |
| target `x86_64-unknown-uefi` | 编译 rboot 需要 |
| musl GCC（或直接用 `PREBUILT=1`） | 构建用户态程序 |
| QEMU + `OVMF_CODE.fd` | 验证 UEFI 网络启动 |
| `isc-dhcp-server`、`tftpd-hpa` | 服务端网络服务 |

安装 Rust 组件：

```bash
rustup component add rust-src llvm-tools-preview
rustup target add x86_64-unknown-uefi
```

---

## 三、拉取源码

`rboot` 和 `user` 是子模块，必须 `--recursive`：

```bash
git clone --recursive https://github.com/rcore-os/rCore.git
cd rCore
```

---

## 四、构建带用户镜像的内核 ELF

### 4.1 生成用户镜像

进入 `user` 目录，用预编译包生成 `x86_64.img`：

```bash
cd user
make sfsimg PREBUILT=1 ARCH=x86_64
```

产物：

```text
user/build/x86_64.img
user/build/x86_64.qcow2
```

### 4.2 编译内核

`BOARD=pc` 会启用 `link_user` feature，把 `user/build/x86_64.img` 直接链进内核 ELF：

```bash
cd ../kernel
make build ARCH=x86_64 BOARD=pc
```

产物：

```text
kernel/target/x86_64/release/rcore      # 带用户镜像的内核 ELF
```

> 注意：`BOARD=pc` 与默认的 `BOARD=qemu` 不同。`pc` 开启 `link_user`，`qemu` 默认从 AHCI 磁盘加载用户镜像。无盘场景必须选 `pc`。

---

## 五、改造 rboot：把内核嵌入 EFI

正常 rboot 会从 UEFI 文件系统读取配置文件和 kernel ELF。为了让整个引导器变成一个可从 TFTP 下载运行的文件，需要加一个 `embed_kernel` feature，在编译期把 `rboot.conf` 和 `kernel.elf` 内嵌进去。

### 5.1 在 `rboot/Cargo.toml` 增加 feature

```toml
[features]
embed_kernel = []
```

### 5.2 修改 `rboot/src/main.rs`

在文件顶部（`const CONFIG_PATH` 附近）加入：

```rust
#[cfg(feature = "embed_kernel")]
const EMBED_CONFIG: &str = include_str!(env!("RCORE_CONFIG_PATH"));
#[cfg(feature = "embed_kernel")]
const EMBED_KERNEL: &[u8] = include_bytes!(env!("RCORE_KERNEL_PATH"));
```

然后把原来的配置读取：

```rust
let config = {
    let mut file = open_file(CONFIG_PATH);
    let buf = load_file(&mut file);
    config::Config::parse(buf)
};
```

替换为：

```rust
let config = {
    #[cfg(feature = "embed_kernel")]
    {
        config::Config::parse(EMBED_CONFIG.as_bytes())
    }
    #[cfg(not(feature = "embed_kernel"))]
    {
        let mut file = open_file(CONFIG_PATH);
        let buf = load_file(&mut file);
        config::Config::parse(buf)
    }
};
```

再把原来的 kernel ELF 读取：

```rust
let elf = {
    let mut file = open_file(config.kernel_path);
    let buf = load_file(&mut file);
    ElfFile::new(buf).expect("failed to parse ELF")
};
```

替换为：

```rust
let elf = {
    #[cfg(feature = "embed_kernel")]
    {
        ElfFile::new(EMBED_KERNEL).expect("failed to parse ELF")
    }
    #[cfg(not(feature = "embed_kernel"))]
    {
        let mut file = open_file(config.kernel_path);
        let buf = load_file(&mut file);
        ElfFile::new(buf).expect("failed to parse ELF")
    }
};
```

> 内嵌模式下 `initramfs` 暂不启用，保持 `rboot.conf` 里 `initramfs=` 被注释掉即可。

---

## 六、编译并部署自包含 EFI

### 6.1 编译

```bash
cd rboot
RCORE_KERNEL_PATH=../kernel/target/x86_64/release/rcore \
RCORE_CONFIG_PATH=./rboot.conf \
cargo build --release --target x86_64-unknown-uefi --features embed_kernel
```

产物：

```text
rboot/target/x86_64-unknown-uefi/release/rboot.efi
```

### 6.2 部署到 TFTP 根目录

```bash
sudo cp rboot/target/x86_64-unknown-uefi/release/rboot.efi \
        /var/lib/tftpboot/BootX64.efi
```

此时 TFTP 根目录只需要这一个文件：

```text
/var/lib/tftpboot/
└── BootX64.efi
```

---

## 七、配置 DHCP

示例 `/etc/dhcp/dhcpd.conf`：

```bash
sudo tee /etc/dhcp/dhcpd.conf << 'EOF'
option domain-name "rcore.local";

default-lease-time 600;
max-lease-time 7200;

# TFTP 服务器地址
next-server 192.168.31.57;

subnet 192.168.31.0 netmask 255.255.255.0 {
    range 192.168.31.150 192.168.31.200;
    option routers 192.168.31.1;
    option subnet-mask 255.255.255.0;
    # 下发 UEFI 引导文件
    filename "BootX64.efi";
}
EOF

sudo systemctl restart isc-dhcp-server
```

---

## 八、QEMU 验证

准备一份 `OVMF_CODE.fd`（可通过 `ovmf` 包或单独下载）。启动 QEMU：

```bash
qemu-system-x86_64 \
  -bios OVMF_CODE.fd \
  -m 4G \
  -netdev user,id=net0,tftp=/var/lib/tftpboot,bootfile=BootX64.efi \
  -device e1000,netdev=net0 \
  -nographic
```

进入 OVMF 后选择 **UEFI PXEv4** 启动项，QEMU 内置的 DHCP/TFTP 会把 `BootX64.efi` 发过去。

预期输出：

```text
bootloader is running
...
Hello world! from CPU 0!
...
$
```

> 如果 OVMF 没有显示 PXE 启动项，进入 **Boot Manager** 手动选择 **UEFI IPv4 Network** 或换用 `-device virtio-net-pci`。

---

## 九、真机/真实网络验证

1. 客户端进入 BIOS/UEFI，开启 **UEFI Network Stack** / **PXE Boot**。
2. 启动顺序选择 **UEFI IPv4 Network**。
3. 客户端从 DHCP 拿到 IP 和 `BootX64.efi` 文件名。
4. 通过 TFTP 下载 `BootX64.efi` 并执行。
5. rboot 从 EFI 内部取出 kernel ELF，建立页表后跳入 rCore 内核。
6. 串口输出 `Hello world! from CPU 0!`，随后进入用户 shell。

---

## 十、与 `DISKLESS-GUIDE.md` 的对比

| 对比项 | Ubuntu 无盘 | rCore 无盘 Demo |
|---|---|---|
| 引导器 | PXELINUX / GRUB2 | rboot（UEFI） |
| 内核来源 | ISO `/casper/vmlinuz` | 自己编译的 `rcore` ELF |
| 根文件系统 | 远端 iSCSI LUN | 编译期链入内核（`link_user`） |
| 网络协议 | DHCP + TFTP + NFS + iSCSI | 只需要 DHCP + TFTP |
| 为什么更简单 | Ubuntu 是完整发行版，需要安装盘 + 根盘 | rCore PC 版把用户镜像直接编进内核 |

---

## 十一、常见问题

| 现象 | 原因 | 解决 |
|---|---|---|
| `cargo build` 报 `feature X` 或 `intrinsics` 错误 | Rust 版本太新 | 必须使用 `nightly-2020-06-04` |
| rboot 编译报找不到 `x86_64-unknown-uefi` | target 未安装 | `rustup target add x86_64-unknown-uefi` |
| `include_bytes!` 找不到 kernel | 路径是相对 `rboot/src/main.rs` 或编译时 CWD | 传绝对路径，或确保在 `rboot/` 目录执行 cargo |
| QEMU 里看不到 PXE 启动项 | OVMF 没识别到网卡 PXE | 按 `Esc` 进 Boot Manager 手动选择；或换 `virtio-net-pci` |
| 启动后停在 rboot，报 `failed to open file` | 没启用 `embed_kernel`，UEFI PXE 没有文件系统 | 确认 `--features embed_kernel` 已加 |
| 真机 PXE 提示 `No boot filename received` | DHCP 没下发 `filename` | 检查 `dhcpd.conf` 和 DHCP 服务状态 |
| 内核起来了但没有用户 shell | 用户镜像没链入 | 确认用了 `BOARD=pc`，并且 `user/build/x86_64.img` 存在 |

---

## 十二、参数速查

| 参数/变量 | 含义 |
|---|---|
| `BOARD=pc` | rCore 为真实 PC 编译，启用 `link_user` |
| `PREBUILT=1` | user 目录使用预编译用户程序包 |
| `RCORE_KERNEL_PATH` | 编译 `embed_kernel` 时指定 kernel ELF 路径 |
| `RCORE_CONFIG_PATH` | 编译 `embed_kernel` 时指定 `rboot.conf` 路径 |
| `filename "BootX64.efi"` | DHCP 下发给 UEFI 客户端的引导文件名 |
