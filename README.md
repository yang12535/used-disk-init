<div align="center">

# Used Disk Init

**二手/拆机硬盘检查、反清零鉴别、坏道验证与初始化 Agent skill**

[![Agent Skill](https://img.shields.io/badge/Agent-Skill-black.svg)](skill/SKILL.md)
[![Platform](https://img.shields.io/badge/Platform-Linux-blue.svg)](#)
[![License](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

</div>

---

## 简介

二手盘市场的常态：SMART 被工具清零冒充低使用时间、盘面敷衍擦除、坏道盘翻新。
本 skill 给出一套实战验证过的完整流程，目标是：**看穿清零、挖出真实履历、
验证盘体健康、安全初始化**。

只要 Agent 能读 Markdown 并能执行本地 shell 命令，就可以使用核心工作流。
仓库额外提供 OpenAI/Codex 元数据，但流程本身不依赖某个特定 Agent 产品。

适用场景：

- 二手/拆机盘入手后的体检与反清零鉴别
- USB 易驱线/硬盘盒接入"没反应"的排查
- 残留数据的只读盘点与前用途画像
- 坏道验证、虚坏道修复、外伤/老化定性
- 格式化初始化、退货取证、HPA 身份档案

> 本项目是社区维护的非官方项目，与任何硬件厂商、NAS 厂商无隶属或背书关系。

## 流程概览

```
识别接入 → 分区表诊断 → SMART 取证（反清零）→ 残留数据盘点 → 坏道验证 → 处置
```

六个阶段均可独立查阅；`skill/SKILL.md` 有意保持简短，只负责任务路由与共享原则，
细节按需加载 `skill/references/` 下的对应文件。

## 仓库布局

```
skill/
├── SKILL.md                        # 路由入口：决策树 + 共享原则 + 报告模板
├── agents/openai.yaml              # OpenAI/Codex 元数据
├── references/
│   ├── intake-partition.md         # 阶段 1-2：识别接入 / 分区表诊断 / 只读探查
│   ├── smart-forensics.md          # 阶段 3：SMART 取证 / 清零鉴别 / 错误日志
│   ├── residual-data.md            # 阶段 4：残留数据判断 / 前用途画像
│   ├── surface-verify.md           # 阶段 5：坏道验证 / 弱区鉴别 / 虚坏道修复
│   ├── initialize.md               # 阶段 6：处置决策 / 格式化 / 数据销毁
│   └── hpa-archive.md              # 附录 A：HPA 身份档案与顺序铁律、桥接与 SATA 的坑
└── examples/
    └── case-wd5000aakx.md          # 一块清零盘的完整鉴定实录（已脱敏）
```

## 原则摘要

- 只读优先：任何写操作前先完成只读检查并报告
- 清零后的 SMART 计数一律不可信，健康结论只信日志与实测扫描
- 他人残留数据只取结构信息，不翻内容、不复述隐私
- 整盘写零 = 数据销毁 + 写入测试，但执行前必须经用户确认
