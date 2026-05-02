# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

「三重人格」是一款基于多Agent架构的沉浸式角色扮演 Skill，支持多Agent物理隔离和单Agent模拟降级双模式。目标平台为 OpenClaw v2.0+，兼容任何 AI 平台。

## 工程结构

```
三重人格SKILL/
├── manifest.json          # Skill 包元数据（版本号在此）
├── skill.json             # 多Agent路由配置 + 隔离策略 + 状态追踪字段
├── .skillignore           # 打包排除列表
├── SKILL.md               # SkillHub 发布页（版本号在此）
├── 需求.md                # V2.0 PRD（不打包）
├── 需求-v3.md             # V3.0 PRD（不打包）
├── CLAUDE.md              # 开发指南（不打包）
├── agents/
│   ├── router/            # 调度中心（探测/降级/状态条/情绪管理/结算条/单Agent模拟）
│   │   ├── config.json
│   │   └── SOUL.md
│   ├── lover/             # 恋人角色（4阶段/情绪波动/冷战/结算条）
│   │   ├── config.json
│   │   └── SOUL.md
│   ├── buddy/             # 损友角色（4阶段/情绪波动/互怼计数/结算条）
│   │   ├── config.json
│   │   └── SOUL.md
│   └── rival/             # 死敌角色（4阶段/情绪波动/胜负记录/结算条）
│       ├── config.json
│       └── SOUL.md
└── assets/
    └── icon.svg           # Skill 图标
```

**关键文件说明：**
- `manifest.json` — 包元数据，字段须与 SKILL.md frontmatter 保持一致
- `skill.json` — Agent 声明、路由绑定（router→default channel）、agentToAgent 权限、Bank-Isolation 配置、state.tracked_fields（36个状态字段）
- `agents/*/SOUL.md` — 各 Agent 的 System Prompt，是核心内容
- `agents/*/config.json` — Agent 运行时配置（model、workspace 隔离、memory scope）

## SKILL.md 格式规范

文件须以 YAML frontmatter 开头，必填字段：

```yaml
---
name: "Skill名称"
version: "3.0.0"
description: "一句话描述"
author: ""
tags: ["标签1", "标签2"]
category: "entertainment"
platform: ["openclaw"]
requires_multi_agent: false
---
```

## 架构：V3.0 多Agent调度模型

```
Router Agent（调度层 + 单Agent降级模拟）
├── 恋人 Agent（独立记忆 + 独立人设 + 情绪波动 + 冷战）
├── 损友 Agent（独立记忆 + 独立人设 + 情绪波动 + 互怼计数）
└── 死敌 Agent（独立记忆 + 独立人设 + 情绪波动 + 胜负记录）
```

**双模式：**
- `⚔️ 真·三重人格`（multi）：多Agent物理隔离，完整体验
- `🎮 模拟模式`（single）：Router 自己模拟三角色，自动降级

**Router 职责**：启动探测 → 模式选择 → 初始化引导 → 切换路由 → 状态条展示 → 情绪管理 → 结算条调度 → 主动触达

**三层隔离防护**（总隔离率 ≈ 99%）：
1. **框架级隔离**（~95%）：OpenClaw 的 Bank-Isolation `["agent","channel","user"]`
2. **Router 防火墙**（+3%）：写在 Router System Prompt 里，禁止传递 USER.md / 全局 MEMORY.md / 跨 Skill 对话
3. **角色免疫指令**（+2%）：写在每个角色 SOUL.md 里，"你只知道用户在我们对话中告诉你的事情"

## V3.0 核心系统

### 四大系统

| 系统 | 实现位置 | 说明 |
|------|----------|------|
| 双模式降级 | Router § 〇 启动探测 | 静默探测 lover Agent → 自动选择模式 |
| 4阶段养成 | Router § 三 + 各角色 SOUL.md | 状态词（0-10/10-30/30-60/60+轮），只升不降 |
| 情绪波动 | Router § 六 + 各角色「情绪波动系统」 | 每角色4种情绪+neutral，上次结束→下次承接 |
| 游戏化界面 | Router § 三/§ 七 + 各角色「结算条」 | 状态条2行+结算条5行，对话中无面板 |

### 养成阶段（4阶段，按累计轮次）

| 阶段 | 恋人 | 损友 | 死敌 | 轮次 |
|:----:|------|------|------|------|
| 1 | 🌱 还在试探 | 🤝 新朋友 | 💨 不屑 | 0-10 |
| 2 | 💗 心动了 | 🍻 老铁 | ⚡ 有点意思 | 10-30 |
| 3 | 🔥 离不开你 | 💪 过命交情 | 🗡️ 棋逢对手 | 30-60 |
| 4 | 💎 只有你 | 🛡️ 生死之交 | 👑 惺惺相惜 | 60+ |

### 默认角色名（按性别）

| 用户性别 | 💕 恋人 | 🍺 损友 | ⚔️ 死敌 |
|----------|---------|---------|---------|
| 男生 | 小柔 | 大壮 | 冷锋 |
| 女生 | 小帅 | 糖糖 | 霜降 |
| 不想选 | 小暖 | 老铁 | 冷锋 |

### skill.json 关键配置

- `isolation`：Bank-Isolation 三维度隔离
- `proactive`：主动触达调度（触发阈值、静默时段 23:00-08:00、冷却 12h）
- `state.tracked_fields`：36 个状态字段（含 agent_mode、养成、情绪、计数、角色专属）

## 内容红线

| 角色 | 红线 | 防塌房机制 |
|------|------|------------|
| 恋人 | 不擦边/色情 | 设定"小脾气"+"偶尔高冷"防止变舔狗 |
| 损友 | 不真伤人 | 设定"认真时刻"+"关心底色" |
| 死敌 | 不人身攻击/霸凌（攻击"能力"不攻击"人格"） | 设定"惺惺相惜"+"偶尔认可" |

## 语言约定

- 全部内容使用**简体中文**
- 角色名称固定用：恋人、损友、死敌
- 情绪 emoji 固定对应：💕 恋人 / 🍺 损友 / ⚔️ 死敌
- SKILL.md 是面向用户的营销文案，语气要有趣、有情绪感染力、让人想安装

## 发布流程

每次发布新版本，按以下步骤执行：

### 1. 更新版本号

同步修改以下文件中的版本号（两处必须一致）：
- `manifest.json` → `"version": "x.y.z"`
- `SKILL.md` → frontmatter `version: "x.y.z"`
- 版本号遵循语义化：`主版本.功能版本.修复版本`

### 2. 安全扫描

运行全面检查（10 项）：
1. JSON 语法验证（6 个 JSON 文件）
2. 敏感信息泄露检查（API key / token / 个人信息）
3. 内容安全红线覆盖（三角色各自的红线 + 越线处理）
4. 三层隔离防护完整性（框架级 + Router防火墙 + 角色免疫指令）
5. 文件一致性（版本号一致 + agentId 目录对应）
6. .skillignore 排除有效性
7. 防塌房机制覆盖
8. V3.0 四大系统完整性
9. Prompt 注入防护
10. 总文件统计

### 3. 打包

```python
import zipfile, os

exclude = {'需求.md', 'CLAUDE.md', '需求-v3.md', '分析报告_三重人格Skill完整指南.md'}
exclude_dirs = {'.git', '.claude', '.claude-internal', '_publish_'}

version = "x.y.z"  # 改为当前版本号

with zipfile.ZipFile(f'三重人格-v{version}.zip', 'w', zipfile.ZIP_DEFLATED) as zf:
    for root, dirs, files in os.walk('.'):
        dirs[:] = [d for d in dirs if d not in exclude_dirs]
        for f in files:
            if f in exclude or f.startswith('.') or f.endswith('.log') or f.endswith('.zip'):
                continue
            filepath = os.path.join(root, f)
            zf.write(filepath, filepath[2:])
```

打包产物：`三重人格-v{version}.zip`（约 35 KB，12 个文件）
注意：SkillHub 不接受 dotfile（如 .skillignore），打包时已自动排除所有以 `.` 开头的文件。

### 4. 上传

- GitHub: `git push origin main`
- SkillHub: 将 zip 文件上传到 OpenClaw SkillHub 发布
