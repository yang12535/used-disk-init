# 阶段 5：坏道验证

```bash
# 单挑错误日志里的坏扇区（能读出=写零时被重映射/修复；I/O错误=还是坏的）
dd if=/dev/sdX bs=512 skip=<坏道LBA> count=1 status=none | hexdump -C

# 扫坏点前后窗口（物理坏道通常成片）
for ((i=L-1000; i<=L+1000; i++)); do
  dd if=/dev/sdX bs=512 skip=$i count=1 status=none 2>/dev/null || echo "LBA $i FAIL"
done

# 全盘只读扫描（约 85 分钟/500G，不破坏数据）——定论级
badblocks -sv -b 4096 /dev/sdX

# 写入式测试（逼出重映射，但摧毁全盘数据，仅空盘用）
badblocks -wsv /dev/sdX

# 或用盘内自检：短 2 分钟 / 扩展 85 分钟，结果查 smartctl -l selftest
smartctl -d sat -t long /dev/sdX
```

扫的同时另开终端 `dmesg -Tw` 盯实时 I/O 错误。

## 写测试结果的判读（实战经验）

- **物理死道：每一轮图案的 readback 都报错**；只报一次的连续区域错误不是死道。
- 一次性连续区域出错要先查盘内错误日志（`smartctl -l xerror` 的错误总数是否增加）：**有新增 UNC = 盘的真实弱区**；无新增 = 传输层瞬时腐坏（桥/线/供电）。
- 弱区（磁性衰减）特征：偶发 UNC，被后续全盘写入"刷新磁化"后恢复读写，固件不重映射。**恢复 ≠ 治愈**，会复发，降级为临时盘。
- 收尾必须定点复核：对出错区域单独写图案回读 3 轮 + 终检 SMART 重映射/待映射计数。

## 弱区形态鉴别（外伤 vs 老化，处置完全不同）

- **连续抱团 + 覆写可复原 + 重映射不增** = 外伤磕碰弱区（运行中受震/被碰，磁头擦碰盘片的弧形区域，LBA 上呈连续一簇）。处置：覆写修复后可继续用，但物理固定必须整改，别再碰它。
- **随机散布 + 覆写后重映射增长** = 介质自然老化，不可逆，降级/退役。
- **边际扇区**：固件自检固定报 read failure（且每次同一 LBA），但主机全重试读取正常、不挂 pending、不重映射——介质信号临界。不是漏修，也修不好，策略是文件系统校验和（btrfs/ZFS）兜底，别反复追。

## 虚坏道（弱磁/断电残留）覆写修复流程

1. 从内核日志收集全部 UNC LBA：`journalctl -k | grep -oE "I/O error, dev sdX, sector [0-9]+" | sort -u`
2. 停掉高 I/O 应用，逐扇区覆零：`dd if=/dev/zero of=/dev/sdX bs=512 seek=<LBA> count=1 conv=notrunc oflag=direct`
3. `sync; echo 3 > /proc/sys/vm/drop_caches`，回读应得全零
4. 复核 SMART：197 应归零；ID 5/196 仍 0 = 虚坏道（物理完好），增长 = 真坏道
5. 盘上是 btrfs 时跑 `btrfs scrub start <挂载点>`：csum 错误会精确列出受损文件与偏移。被覆写扇区上的原数据在读失败时已丢，覆写不产生额外损失，但 scrub 能给出损失清单。
