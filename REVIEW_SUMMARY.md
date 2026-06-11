# 文档审阅报告：README.md + ARCHITECTURE.md

> 审阅日期：2026-06-11  
> 审阅对象：项目入口文档与技术架构设计文档  
> 审阅目的：在正式实施前识别文档缺口、风险与改进点

---

## 1. 总体评价

| 维度 | 评分 | 说明 |
|------|------|------|
| 架构合理性 | ★★★★★ | LVM Thin Snapshot + 系统盘/数据盘双盘分离设计成熟、有依据 |
| 文档完整性 | ★★★★☆ | 核心设计覆盖充分，但安全假设、Boot API 接口、运维流程有缺口 |
| 可实施性 | ★★★★☆ | Windows iSCSI Boot 风险（R1）被正确识别，路线图清晰 |
| 文档间一致性 | ★★★★★ | README 与 ARCHITECTURE 无矛盾，术语与技术栈描述统一 |

**结论**：两份文档质量较高，可直接作为实施参考。建议在编码前补齐以下 4 类缺口（安全、API、运维、监控），以避免实施阶段返工。

---

## 2. README.md 审阅意见

### 2.1 优点

- 开篇即点明技术栈（iPXE + iSCSI + LVM Thin Snapshot）和场景边界（1~5 台/千兆），信息密度高。
- 「快速对比」表格让读者一眼看出与旧方案 `DISKLESS-GUIDE.md` 的差异。
- 「两种使用模式」表格把技术方案映射到实际场景（网吧 vs 学校机房），降低理解门槛。

### 2.2 建议改进

| # | 问题 | 建议 |
|---|------|------|
| 1 | 缺少快速开始或部署前提链接 | 架构图下方加一句导航，如："详细部署步骤见 `ARCHITECTURE.md` 第 8 节（MVP 路线图）" |
| 2 | 无硬件/系统要求说明 | 补充：服务端建议 Ubuntu 24.04+/Debian 12+、500 GB+ 存储、支持虚拟化的 CPU（用于 QEMU 准备 Windows 镜像） |
| 3 | 未声明项目阶段 | 在标题或引言加标签，如 `🚧 MVP 阶段` 或 `⚠️ 尚未投入生产环境`，避免误用 |
| 4 | 无 License 信息 | 底部补充开源许可证声明（如 MIT/GPL），符合开源项目惯例 |

---

## 3. ARCHITECTURE.md 审阅意见

### 3.1 优点

1. **分层架构图表达清晰**：控制管理 → 启动协议 → 存储后端 → 客户端四层结构易于理解。
2. **数据流完整**：第 2.2 节的 9 步启动流程覆盖了从开机到重启的全生命周期。
3. **方案对比有理有据**：第 4.2 节 LVM Thin Snapshot 选型表格对比了 fileio、qcow2、ZFS 等方案，说服力强。
4. **风险登记册实用**：第 10 节 R1~R5 覆盖了 Windows 兼容性、网络拥塞等真实风险。

### 3.2 发现的问题与建议

#### 3.2.1 🔴 安全隐患——建议优先补充

文档中 iSCSI Target 配置示例全部关闭认证：

```bash
targetcli ... set attribute authentication=0 generate_node_acls=1 demo_mode_write_protect=0
```

在 1~5 台封闭内网场景下此配置可接受，但文档**完全没有提及安全假设**。建议：

- 新增「安全假设」小节，明确本项目设计前提是**可信局域网环境**（如家庭/小型工作室内网）。
- 补充安全加固方向：如需跨网段或不可信网络部署，可启用 targetcli CHAP 单向/双向认证。

#### 3.2.2 🟡 Boot API 接口定义不足

第 3.3 节仅展示了 iPXE 脚本响应示例，缺少服务端接口的正式定义：

| 缺失项 | 影响 |
|--------|------|
| HTTP 状态码约定 | 客户端无法区分「未知 MAC」与「服务端内部错误」 |
| 错误响应格式 | Boot API 存储操作失败时，iPXE 脚本如何优雅回退？ |
| 持久化存储说明 | 客户端配置（MAC → 系统/数据盘映射）存于 SQLite？文件？内存？ |

建议补充最小 API 接口定义，例如：

```
GET /boot?mac=<mac>&arch=<arch>
  → 200 + iPXE 脚本（Content-Type: text/plain）
  → 404 客户端未注册
  → 503 存储服务暂不可用

POST /api/prepare?mac=<mac>&os=<os>
  → 200 snapshot 与 target 准备完成
  → 500 LVM/targetcli 操作失败（含错误详情）
```

#### 3.2.3 🟡 Golden Image 更新与运维流程缺失

文档详细描述了首次部署流程，但未涉及**母盘迭代运维**——这是长期运行的关键：

- Ubuntu/Windows 补丁更新后，如何更新 golden image？
- 更新母盘时，现有客户端 snapshot 是否受影响？
- 是否需要维护窗口（停服更新）？

建议新增「母盘更新流程」小节，明确：

1. 创建新 golden LV（如 `ubuntu-golden-v2`）
2. 从旧 snapshot 迁移或通知客户端重启切换
3. 旧 golden 保留回滚，确认稳定后删除

#### 3.2.4 🟡 监控与告警未提及

LVM thin pool 的 snapshot 采用 COW（写时复制）机制。若 thin pool 空间耗尽，snapshot 将损坏且**不可逆**。建议补充：

- thin pool 使用率监控阈值（如 >80% 告警）
- Boot API 健康检查端点（`GET /health`）
- 客户端启动失败日志收集方案

#### 3.2.5 🟢 项目文件规划与实际目录不一致

第 9 节规划的 `scripts/`、`configs/`、`docs/` 目录在当前仓库中尚未创建。建议：

- 加标注说明这是「规划中的目录结构」；或
- 待实际创建后再补充该节，避免文档与仓库状态脱节。

#### 3.2.6 🟢 技术细节勘误与优化建议

| 位置 | 原文 | 建议 |
|------|------|------|
| 4.4 节 | `lvcreate -V 100G -T ... -n ubuntu-golden` 后 5.2.2 节用 `dd` 导入 | 可用 `qemu-img convert` 直接写入 LV 或 QEMU 直接安装到 `/dev/diskless-vg/win10-golden`，减少一次全量拷贝 |
| 5.2.3 节 | PowerShell 中 `$targetIQN = "...client-$env:COMPUTERNAME-data"` | `COMPUTERNAME` 要求母盘内预配置计算机名→MAC 映射。建议改为通过 DHCP/iPXE 参数传递 MAC 或客户端标识，降低母盘制作复杂度 |
| 6.2 节 | iPXE 菜单示例中出现两段 `chain /boot` + `menu` | 建议加注释明确：**客户端嵌入脚本**（发起请求）vs **服务端返回脚本**（显示菜单），避免读者混淆 |
| 8 节 | 工作量估算约 10~15 天 | 建议加粗说明这是纯技术工作量，不含硬件采购/等待时间 |

---

## 4. 优先修复清单

按实施阻塞程度排序：

| 优先级 | 事项 | 归属文档 | 理由 |
|--------|------|----------|------|
| 🔴 P0 | 补充「安全假设」与内网可信声明 | ARCHITECTURE.md | 避免在不可信网络中误用导致数据泄露 |
| 🔴 P0 | 定义 Boot API 最小接口与错误码 | ARCHITECTURE.md | 直接影响编码实现与前后端联调 |
| 🟡 P1 | 补充 Golden Image 更新运维流程 | ARCHITECTURE.md | 系统上线后必须面对补丁与迭代 |
| 🟡 P1 | 补充 thin pool 监控告警方案 | ARCHITECTURE.md | 防止 COW 空间耗尽导致 snapshot 损坏 |
| 🟢 P2 | README 增加 License + 项目阶段标识 | README.md | 开源项目惯例 |
| 🟢 P2 | 统一「规划目录」标注 | ARCHITECTURE.md | 避免文档与仓库状态不一致 |

---

## 5. 附录：术语一致性检查

| 术语 | README.md | ARCHITECTURE.md | 一致性 |
|------|-----------|-----------------|--------|
| 技术栈 | iPXE + iSCSI + LVM Thin Snapshot | iPXE + iSCSI + LVM Thin Snapshot | ✅ 一致 |
| 场景 | 1~5 台，千兆 | 1~5 台，千兆 | ✅ 一致 |
| 启动器 | iPXE | iPXE | ✅ 一致 |
| 存储方案 | LVM Thin Pool + snapshot | LVM Thin Pool + snapshot | ✅ 一致 |
| 用户数据盘 | 独立 thin LV，持久保留 | 独立 thin LV，持久保留 | ✅ 一致 |

---

> 本报告为审阅摘要，具体修改建议已按章节归类。如需针对某一节展开详细修订稿，可继续迭代。
