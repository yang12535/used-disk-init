# 附录 A：HPA 身份档案

把鉴定结论写进盘尾 HPA（Host Protected Area），格式化/分区都够不到。

> ⚠️ **顺序铁律：HPA 必须先于分区/建池设置**。盘一旦分区或加入 NAS 存储池（md/LVM），盘尾已归阵列数据区，此时再开 HPA 等于抽走活动阵列的扇区，直接故障；NAS 系统通常也拒绝从 linear/jbod 池撤离成员盘（实测 fnOS `stor.removeDisk` 报 errno 196624）。固定顺序：**鉴定 → 写零 → HPA+档案 → 再进池**。若已进池，档案降级为文件形式另存（单盘单文件 + SHA256），别动阵列。

```bash
hdparm -N /dev/sdX                        # 查当前 max sectors / HPA 状态
# 1. 把档案文本写入盘尾 N 个扇区（如 2048 扇区=1MiB），sha256 校验
dd if=info.txt of=/dev/sdX bs=512 seek=$((TOTAL-2048)) conv=sync,fsync
# 2. 永久启用 HPA 隐藏该区域（TOTAL=原生扇区数）
hdparm --yes-i-know-what-i-am-doing -N p$((TOTAL-2048)) /dev/sdX
# 3. 重新插拔后生效验证
sg_readcap /dev/sgX | grep "Last LBA"     # 应显示 TOTAL-2048-1
```

档案内容建议：盘体身份（型号/序列号/固件）、真实履历（真实通电小时+证据来源）、前用途、鉴定日期、鉴定时 SMART 快照、文件自述（位置/读取方法/校验值）。双副本（HPA 首+中部）。容量参考：UTF-8 英文 1B/字符、中文 3B/字符，1MiB 约 35 万中文字，一份 12KB 的档案可放 80 余份。

## JMS583 白牌桥接的精分行为（实测，换正规转接器/原生 SATA 不受限）

- **永久 SET MAX 每次上电只放行一次**，之后一律 I/O 错误；临时（不带 p）的放行逻辑也不稳定，会"假成功"（hdparm 报 OK 但盘不动）。
- **带着 HPA 接入的会话内无法解除 HPA**：桥接固件按接入时容量过滤 LBA，lift 成功也读不到。
- 后果：在 JMS583 盒上写入 HPA 档案后，**该盒上它等于物理封存**；读取需换转接器/原生 SATA：`hdparm -N <TOTAL> /dev/sdX`（临时）→ `echo 1 > /sys/class/block/sdX/device/rescan` → `dd ... | strings`。
- hdparm -N 的显示在该桥上不可信，以 `sg_readcap`（READ CAPACITY 直读盘）为准。
- 解除临时 HPA 后内核容量缓存不自动更新，普通 dd 会读空；native SATA 上 rescan 可解决。

## 原生 SATA 下的坑（实测）

- **SG_IO ATA_16 透传读写，`cdb[1]` 必须带 EXTEND 位（bit0）**：否则 libata 不下发 hob 寄存器，48 位 LBA 高 24 位丢失——命令"成功"但落在了错误位置（用户区！）。写完必须用独立路径（hdparm --read-sector）回读比对验证。
- `hdparm --write-sector` **只写零、忽略 stdin**，别用它写数据。
- 部分主板 POST 会对盘 **SET MAX LOCK**：`hdparm -N` 报 ABRT 时，先经 SG_IO 发 SET MAX UNLOCK（0xF3，non-data 协议）解锁即可，不用断电。
- 临时解除 HPA 后内核容量缓存若 rescan 也不更新，可逐扇区 `hdparm --read-sector` 透传读取，输出 hex 按字节流直接解。
