---
name: used-disk-init
description: "二手/拆机硬盘的检查、鉴别、修复与初始化（USB 易驱线/硬盘盒或原生 SATA 接入的 HDD/SSD）。当用户提到二手盘检查、拆机盘、USB 硬盘没反应/不识别、清零盘鉴别、检查这块盘、硬盘格式化初始化、SMART 健康检查、磁盘有残留数据时触发。流程：识别接入 → 分区表诊断 → SMART 取证（反清零）→ 残留数据盘点 → 坏道验证 → 处置（格式化/HPA 身份档案/退货取证）。不用于全新未拆封盘的常规分区，也不处理 RAID 阵列重建与数据恢复级故障。"
---

# Used Disk Init：二手盘检查与初始化

二手盘市场的常态：SMART 被工具清零冒充低使用时间、盘面敷衍擦除、坏道盘翻新。本流程的目标：**看穿清零、挖出真实履历、验证盘体健康、安全初始化**。

## 每次任务前

1. **确认盘符**：`lsblk -dno NAME,SIZE,MODEL,TRAN` 找到目标盘——重插、断电、多盘并存都会让 sda/sdb/sdc 变化，绝不凭记忆读写任何盘。
2. **只读优先**：任何写操作（修分区表、格式化、HPA）之前，先把只读检查做完并向用户报告。例外：用户明确下令。
3. **USB 桥接必须加 `-d sat`**：`smartctl /dev/sda` 不加会卡死 60 秒以上；`smartctl -d sat` 秒回。
4. **按任务加载对应参考文件**，不要默认读完全部指南。工具准备：`apt-get install -y smartmontools hdparm sg3-utils gdisk exfatprogs`。

## 快速决策树

- 插上没反应、不知道哪块是目标盘、要看新接入事件 → [references/intake-partition.md](references/intake-partition.md)
- 无分区表、盘首全零、疑似敷衍擦除、要只读探查分区 → [references/intake-partition.md](references/intake-partition.md)
- 怀疑清零盘、要挖真实通电时间/错误史/退货证据 → [references/smart-forensics.md](references/smart-forensics.md)
- 要判断盘里有没有残留数据、画像前用途 → [references/residual-data.md](references/residual-data.md)
- 要验证坏道、判读扫描结果、修复虚坏道、鉴别外伤/老化 → [references/surface-verify.md](references/surface-verify.md)
- 要格式化初始化、评估留用/退货 → [references/initialize.md](references/initialize.md)
- 要把鉴定档案写进盘尾 HPA、或盘已加入 NAS 存储池 → [references/hpa-archive.md](references/hpa-archive.md)

## 共享原则

- 清零后的 SMART 计数一律不可信：重映射/待映射显示 0 不代表没有，只代表被抹过。判健康要靠自检日志、错误日志、实测扫描。
- 挂载一律 `mount -o ro,noload`（只读、不回放日志），防止检查过程改写数据。
- 盘点他人残留数据时只取结构信息（目录名、大小、时间戳、字符串画像），不翻文件内容，不复述账号/姓名等隐私。
- 整盘写零同时是数据销毁和写入测试，但它是破坏性操作，执行前必须经用户确认。
- 弹出用 `udisksctl power-off -b /dev/sdX`，确认 `lsblk` 和 `lsusb` 都消失后再让用户拔线。

## 完成报告

```
型号/序列号/固件/容量：
桥接/接入方式：
分区表状态：（完好 / 主头被抹备份完好 / 全零 / 正常）
SMART 判决：（正常 / 疑似清零 / 实锤清零+证据）
真实履历：（通电小时数+证据来源，错误史，前用途）
残留数据：（无 / 有什么，多少）
坏道验证：（未测 / 结果）
处置建议：（放心用 / 临时盘 / 退货+证据 / 其他）
档案处置：（未立 / HPA 已埋 / 文件版+路径）
```
