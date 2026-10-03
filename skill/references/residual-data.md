# 阶段 4：残留数据判断

```bash
# 全盘抽样：各偏移读 4M 数非零字节
for off in 0 1048576 1073741824 107374182400 268435456000 499000000000; do
  nz=$(dd if=/dev/sdX bs=1M count=4 skip=$((off/1048576)) 2>/dev/null | tr -d '\000' | wc -c)
  echo "偏移 $off: 非零 $nz"
done
```

- 全零 = 整盘写零过（卖家有工具且花了时间，500G 约 85 分钟）。
- 有非零 = 敷衍擦除，用 [intake-partition.md](intake-partition.md) 的 loop 偏移法把分区挂出来盘点，**只列目录名和大小，不翻文件内容**（他人隐私）。告知用户后由用户决定去留。

## 前用途画像（找不到有效文件系统边界时）

分区表被抹、超级块被零化时，仍可直接对大块区域抽样 `strings`：

```bash
dd if=/dev/sdX bs=1M count=8 skip=$((GB*1024)) 2>/dev/null | strings -n 10 | head -12
```

判读线索：

- **glibc 错误信息表**（"permission denied / file exists / no such device"成组出现）= Linux 系统的二进制/库文件区
- **浏览器缓存 / JS SDK / 会话时间戳** = 桌面机使用痕迹与年代（时间戳可直接换算出使用年份）
- **厂商/发行版特定字符串**（如 "Kingsoft Kylin WPS"、openEuler 路径）= 系统发行版与用途线索
- 大块高熵区域 = 压缩或加密的用户数据，无法也无需求解

**只取结构信息用于鉴定，不复述他人账号、姓名、会话令牌等隐私内容**；发现敏感残留时，处置建议里必须包含"整盘覆写销毁"，而不是只删分区表。
