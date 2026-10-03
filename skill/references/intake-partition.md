# 阶段 1-2：识别接入与分区表诊断

## 阶段 1：识别接入（30 秒）

```bash
lsusb                                    # 找桥接芯片（152d:0583=JMicron 等）
dmesg -T | tail -25                      # 看新接入事件：容量、型号、报错
lsblk -dno NAME,SIZE,TYPE,FSTYPE,MODEL,TRAN
```

> ⚠️ `dmesg -T` 的时间戳在挂起过的机器上会整体漂移（printk 用 monotonic 钟，挂起暂停；uptime 含挂起时间）。要精确墙钟时间用 `journalctl -k`，要相对时间用 `dmesg -e`。

判读要点：

- 内核认到盘（`Attached SCSI disk`）≠ 能用。**"插上没反应"十有八九不是驱动问题**，是分区表/文件系统损坏，继续往下查。
- 白牌易驱线常见自定义 USB 字符串（如 `Manufacturer: jack88888`），说明固件被工厂用量产工具刷过，后续 ATA 透传行为可能精分（见 [hpa-archive.md](hpa-archive.md)）。

## 阶段 2：分区表诊断

```bash
fdisk -l /dev/sdX            # 无分区表？
blkid -p /dev/sdX            # 无签名？
wipefs -n /dev/sdX           # 扫所有签名（含盘尾备份 GPT）
hexdump -C -n 512 /dev/sdX   # 盘首全零 = 主 GPT/MBR 被抹
```

**盘首全零但有数据 ≠ 空盘**。GPT 在盘尾有备份，检查：

```bash
gdisk /dev/sdX   # 若提示 "invalid main GPT header, but valid backup"
                 # → 选 1 (Use current GPT) → p 看分区 → v 校验 → q 退出
```

备份 GPT 完好而主头被抹，是典型的**敷衍擦除**（只零了盘首），后面数据区大概率原封不动。修复方法（需用户确认）：gdisk 里按 `w` 把备份表写回主头。

### 只读探查分区内容（内核还认不出分区时）

```bash
# 按 gdisk 给的扇区号建只读 loop（偏移 = 扇区号 × 512）
loop=$(losetup --find --show --read-only -o $((3328000*512)) /dev/sdX)
file -s $loop; blkid -p $loop
# 发现 LVM2 PV → 激活卷组
pvscan; vgchange -ay; lvs
mount -o ro,noload /dev/<vg>/<lv> /mnt/xxx
# 用完清理：umount → vgchange -an <vg> → losetup -d $loop
```
