# 阶段 6：处置决策与格式化初始化

## 健康尚可、用户要自用 → 格式化初始化

```bash
wipefs -a /dev/sdX                          # 清签名（含旧 LVM 等）
sgdisk -o -n1:0:0 -c1:"USB-DATA" /dev/sdX   # 新 GPT + 整盘单分区
mkfs.exfat -n USBDATA /dev/sdX1             # 跨平台选 exfat；纯 Linux 选 ext4
# 挂载写入测试 → umount → sync → udisksctl power-off -b /dev/sdX
```

要进 NAS 存储池的，**用 NAS 系统自己的界面/API 建池**（它会接管分区、阵列和文件系统并登记在册），不要手工 mdadm/LVM/mkfs——系统不认。

## 有 UNC 坏道史 / 真实通电上万小时

明确告知风险：只配当临时盘，不存唯一副本；在退换期内建议退货（[smart-forensics.md](smart-forensics.md) 的证据链）。

## 残留数据含他人隐私

光删分区表或快速格式化抹不掉数据。初始化前先整盘覆写（`dd if=/dev/zero of=/dev/sdX bs=4M conv=fsync`，500G 约 75 分钟）——数据销毁与写入测试一次完成。

整盘 dd 写零收尾报 `No space left on device` 是**正常结束**（bs 不整除盘容），以 records out × bs 或末次字节数核对是否写满，缺尾再补。

写零/扫描结束立刻复核 SMART 三项（ID 5 / 196 / 197）：全零 = 读写双绿；有增量按 [surface-verify.md](surface-verify.md) 判读定性。

## 立身份档案

鉴定结论值得随盘保存（真实履历、健康基线、SMART 快照、校验和），详见 [hpa-archive.md](hpa-archive.md)——注意 HPA 必须先于分区/建池设置，顺序错了就只能存文件版。
