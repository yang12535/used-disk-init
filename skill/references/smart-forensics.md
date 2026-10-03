# 阶段 3：SMART 取证（反清零核心）

```bash
smartctl -d sat -a /dev/sdX           # 总览（USB 桥接才加 -d sat，原生 SATA 不用）
smartctl -d sat -l selftest /dev/sdX  # 自检日志（记录测试时的真实盘龄！）
smartctl -d sat -l xerror /dev/sdX    # 扩展错误日志（含出错时的真实盘龄）
smartctl -d sat -x /dev/sdX           # 寿命温度等生存期统计
```

## 清零盘鉴别特征（中 2 条以上基本实锤）

| 特征 | 说明 |
|---|---|
| `Power_On_Hours` = 0 或个位数 | 配合下面任何一条即可疑 |
| `Power_Cycle/Start_Stop/Load_Cycle` 全为 1 | 一整套计数器整齐归零 |
| `Total_LBAs_Written` 极小（几十/几百） | 有使用痕迹的盘不可能 |
| **自检日志里 LifeTime 远大于 PoH** | 清零工具常忘清自检日志 → 真实年龄 |
| **xerror 日志里 lifetime 远大于 PoH** | 连自检都清了时查这个 → 真实年龄 + 故障史 |
| `Lifetime Min/Max Temperature` 存在 | 生存期统计，辅助佐证 |
| 属性表异常"干净"/缺常见统计项 | 深度清零会连统计属性一并抹除；也可能是固件分支差异，仅作旁证 |

注意手法在进化：早期清零只抹属性、自检日志漏网（真实盘龄直接可查）；较新的处理会把自检日志和扩展错误日志一并清空——此时真实小时数不可考，只能靠盘面残留数据推断使用年代（见 [residual-data.md](residual-data.md)）。

## 错误日志判读

- `Device Error Count` 是**寿命累计错误数**，日志只留最近 24 条。
- 错误类型 `UNC`（不可纠正读错误）= **真实坏道**，盘上退役前出过物理缺陷，风险升一档。
- 错误密集发生在同一 lifetime 且 LBA 相同 = 退役前最后的挣扎（主机反复重试坏扇区）。
- **这些日志是退货刚客服的硬证据**，截图保存：`smartctl -d sat -l xerror /dev/sdX > evidence.txt` 连同 `smartctl -d sat -a` 全量。
