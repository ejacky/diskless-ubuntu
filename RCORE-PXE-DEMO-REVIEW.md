# RCORE-PXE-DEMO.md 审阅报告

**审阅依据：** `/tmp/rcore-test`（rCore 仓库本地克隆）

**审阅对象：** `RCORE-PXE-DEMO.md`

**审阅日期：** 2026-06-11

---

## 总体评价

文档整体思路正确：rCore `x86_64/pc` 通过 `link_user` 把用户镜像编进内核，再配合一个内嵌了 `kernel.elf` + `rboot.conf` 的 `rboot.efi`，确实可以只依赖 DHCP + TFTP 完成无盘启动。

但文档中存在若干**会直接导致命令失败或看不到输出**的细节错误，需要在正式发布前修正。

---

## 一、会阻断执行的严重问题

### 1. 编译 rboot 的命令缺少 `-Z build-std`

**文档位置：** 第 189–194 行

**原文：**

```bash
cd rboot
RCORE_KERNEL_PATH=../kernel/target/x86_64/release/rcore \
RCORE_CONFIG_PATH=./rboot.conf \
cargo build --release --target x86_64-unknown-uefi --features embed_kernel
```

**问题：** `x86_64-unknown-uefi` target 没有 std，只有 core/alloc。实际 `/tmp/rcore-test/rboot/Makefile` 中明确使用：

```makefile
BUILD_ARGS := -Z build-std=core,alloc --target x86_64-unknown-uefi
```

直接按文档命令编译会失败。

**建议改为：**

```bash
cd rboot
RCORE_KERNEL_PATH=../../kernel/target/x86_64/release/rcore \
RCORE_CONFIG_PATH=../rboot.conf \
cargo build -Z build-std=core,alloc \
            --release --target x86_64-unknown-uefi \
            --features embed_kernel
```

> 注：路径修正见下一条。

---

### 2. `RCORE_KERNEL_PATH` / `RCORE_CONFIG_PATH` 的相对路径错误

**文档位置：** 第 191–192 行

**原文：**

```bash
RCORE_KERNEL_PATH=../kernel/target/x86_64/release/rcore \
RCORE_CONFIG_PATH=./rboot.conf \
```

**问题：** `include_str!` / `include_bytes!` 的相对路径基准是**调用宏的源文件**（即 `rboot/src/main.rs`），而不是 cargo 执行目录（`rboot/`）。

| 文档写法 | 实际解析路径 | 是否存在 |
|---|---|---|
| `../kernel/target/x86_64/release/rcore` | `rboot/kernel/target/x86_64/release/rcore` | ❌ 不存在 |
| `./rboot.conf` | `rboot/src/rboot.conf` | ❌ 不存在 |

**建议改为（相对于 `rboot/src/main.rs`）：**

```bash
RCORE_KERNEL_PATH=../../kernel/target/x86_64/release/rcore \
RCORE_CONFIG_PATH=../rboot.conf \
```

或使用绝对路径：

```bash
RCORE_KERNEL_PATH="$(pwd)/../kernel/target/x86_64/release/rcore" \
RCORE_CONFIG_PATH="$(pwd)/rboot.conf" \
```

---

### 3. `BOARD=pc` 在 QEMU `-nographic` 下看不到串口输出

**文档位置：** 第 90 行与第 248–269 行之间存在冲突

**原文要求：**

- 编译内核使用 `BOARD=pc`：

  ```bash
  make build ARCH=x86_64 BOARD=pc
  ```

- QEMU 验证使用 `-nographic`：

  ```bash
  qemu-system-x86_64 \
    -bios OVMF_CODE.fd \
    -m 4G \
    -netdev user,id=net0,tftp=/var/lib/tftpboot,bootfile=BootX64.efi \
    -device e1000,netdev=net0 \
    -nographic
  ```

**问题：** 实际代码 `/tmp/rcore-test/kernel/src/arch/x86_64/io.rs`：

```rust
pub fn putfmt(fmt: Arguments) {
    // output to serial
    #[cfg(not(feature = "board_pc"))]
    {
        let mut drivers = SERIAL_DRIVERS.write();
        let serial = drivers.first_mut().unwrap();
        serial.write(format!("{}", fmt).as_bytes());
    }
    // ...
}
```

`BOARD=pc` 会启用 `board_pc` feature，导致 `putfmt` **跳过串口输出**。而 `-nographic` 完全依赖串口重定向到终端。结果是：内核可能已正常启动，但终端上**看不到** `Hello world! from CPU 0!`，也看不到用户 shell。

**建议方案（三选一）：**

1. **QEMU 验证时去掉 `-nographic`**，使用图形窗口观察输出；
2. **在文档中明确说明**：QEMU 验证阶段仅验证 PXE 能下载并执行 EFI，串口输出需在真机验证；
3. **补充临时调试说明**：若要在 QEMU `-nographic` 下看到输出，可临时在 `kernel/src/arch/x86_64/io.rs` 中移除 `#[cfg(not(feature = "board_pc"))]` 的限制（仅用于调试，不要提交）。

---

## 二、中等问题

### 4. QEMU 验证与系统 DHCP/TFTP 服务的关系未澄清

**文档位置：** 第七节与第八节之间

**问题：** 第七节配置了 `isc-dhcp-server`，第八节 QEMU 验证却使用 QEMU 内置的 DHCP/TFTP：

```bash
-netdev user,id=net0,tftp=/var/lib/tftpboot,bootfile=BootX64.efi
```

读者可能误以为两节必须同时使用，导致端口冲突或困惑。

**建议：** 在第八节开头加一段说明：

> 本节使用 QEMU 内置的 DHCP/TFTP 服务验证 PXE 流程，**无需启动第七节配置的系统级 `isc-dhcp-server`**。真机/真实网络验证才需要第七节。

---

### 5. `make build ARCH=x86_64 BOARD=pc` 的产物描述不完整

**文档位置：** 第 95–99 行

**原文产物列表：**

```text
kernel/target/x86_64/release/rcore      # 带用户镜像的内核 ELF
```

**问题：** `kernel/Makefile` 中 `build` 目标依赖 `$(kernel_img)`，对 `x86_64` 会：

1. 进入 `rboot` 编译；
2. 创建 ESP 目录；
3. 生成 `kernel/target/x86_64/release/kernel.img` 和 ESP 目录结构。

文档只列出 ELF，会让读者不理解为什么编译过程中还涉及 rboot 和配置文件复制。

**建议补充为：**

```text
kernel/target/x86_64/release/rcore        # 带用户镜像的内核 ELF
kernel/target/x86_64/release/kernel.img   # 完整 ESP 镜像（仅本地验证用）
kernel/target/x86_64/release/esp/         # ESP 目录，含 EFI/Boot/BootX64.efi、rboot.conf、EFI/rCore/kernel.elf
```

---

### 6. `embed_kernel` 是"新增并改造"，不是简单开关

**文档位置：** 第五节

**问题：** 实际 `/tmp/rcore-test/rboot/Cargo.toml` 中：

```toml
[features]
rboot = ["uefi-services"]
default = ["rboot"]
```

根本没有 `embed_kernel` feature，且 `src/main.rs` 也没有 `include_str!` / `include_bytes!` 逻辑。第五节描述的修改是**必须实现的补丁**，而不是启用已有功能。

**建议：** 在 5.1 开头明确说明：

> 当前 rboot 仓库没有内嵌 kernel/config 的能力，需要按以下步骤手动添加 `embed_kernel` feature 并修改源码。

---

## 三、小问题和补充建议

### 7. `make sfsimg` 的 `MODE` 与内核默认不一致

**文档位置：** 第 76 行

**原文：**

```bash
cd user
make sfsimg PREBUILT=1 ARCH=x86_64
```

**问题：** `user/Makefile` 默认 `MODE ?= debug`，而 `kernel/Makefile` 默认 `MODE ?= release`。两者产物路径都是 `user/build/x86_64.img`，内核能链接上，但会出现用 debug 用户镜像链接进 release 内核的情况。

**建议：** 统一 MODE：

```bash
cd user
make sfsimg PREBUILT=1 ARCH=x86_64 MODE=release
```

---

### 8. DHCP 配置示例可更健壮

**文档位置：** 第 222–242 行

**建议：** 示例配置中增加 `authoritative;`，否则在某些网络环境下 `isc-dhcp-server` 可能不响应 PXE 请求。

```bash
sudo tee /etc/dhcp/dhcpd.conf << 'EOF'
option domain-name "rcore.local";

default-lease-time 600;
max-lease-time 7200;

authoritative;

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
```

---

### 9. OVMF 文件名

**文档位置：** 第 248 行

**问题：** 文档 QEMU 命令使用 `OVMF_CODE.fd`，而 `/tmp/rcore-test/rboot/` 下自带的是 `OVMF.fd`。

**建议：** 说明 `OVMF_CODE.fd` 可从 `ovmf` 包获取，或直接使用 rboot 自带的 `OVMF.fd` 并相应替换命令。

---

### 10. 关于 `initramfs` 的说明

**文档位置：** 第 181 行

**评价：** 说明正确。`/tmp/rcore-test/rboot/rboot.conf` 中 `initramfs=` 默认被注释掉，无需额外操作。

---

## 四、审阅结论

| 维度 | 评价 |
|---|---|
| 整体架构 | ✅ 正确 |
| 命令可执行性 | ⚠️ 3 处会失败或看不到输出（build-std、路径、串口输出） |
| 与代码一致性 | ⚠️ 部分描述与实际 rboot/kernel 源码不完全一致 |
| 完整性 | ⚠️ 缺少 QEMU/系统 DHCP 关系说明、产物说明 |

**建议优先修正：**

1. rboot 编译命令加 `-Z build-std=core,alloc`；
2. `RCORE_KERNEL_PATH` / `RCORE_CONFIG_PATH` 改为相对于 `rboot/src/main.rs` 的正确路径；
3. 明确 `BOARD=pc` 在 QEMU `-nographic` 下无串口输出，给出替代验证方案；
4. 补充说明第八节 QEMU 验证不依赖第七节系统 DHCP。

完成以上修正后，文档应可作为可落地的操作指南。
