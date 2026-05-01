# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

「三重人格」是一款基于多Agent架构的沉浸式角色扮演 Skill，目标平台为 OpenClaw v2.0+（V1.0 优先），WorkBuddy 后续适配。

## 工程结构

```
三重人格SKILL/
├── manifest.json          # Skill 包元数据
├── skill.json             # 多Agent路由配置 + 隔离策略
├── .skillignore           # 打包排除列表
├── SKILL.md               # SkillHub 发布页
├── 需求.md                # PRD（不打包）
├── CLAUDE.md              # 开发指南（不打包）
├── agents/
│   ├── router/            # 调度中心（初始化、切换、防火墙）
│   │   ├── config.json
│   │   └── SOUL.md
│   ├── lover/             # 恋人角色
│   │   ├── config.json
│   │   └── SOUL.md
│   ├── buddy/             # 损友角色
│   │   ├── config.json
│   │   └── SOUL.md
│   └── rival/             # 死敌角色
│       ├── config.json
│       └── SOUL.md
└── assets/
    └── icon.svg           # Skill 图标
```

**关键文件说明：**
- `manifest.json` — 包元数据，字段须与 SKILL.md frontmatter 保持一致
- `skill.json` — Agent 声明、路由绑定（router→default channel）、agentToAgent 权限、Bank-Isolation 配置
- `agents/*/SOUL.md` — 各 Agent 的 System Prompt，是核心内容
- `agents/*/config.json` — Agent 运行时配置（model、workspace 隔离、memory scope）

## SKILL.md 格式规范

文件须以 YAML frontmatter 开头，必填字段：

```yaml
---
name: "Skill名称"
version: "1.0.0"
description: "一句话描述"
author: ""
tags: ["标签1", "标签2"]
category: "entertainment"
platform: ["openclaw", "workbuddy"]
requires_multi_agent: true
---
```

## 架构：多Agent调度模型

```
Router Agent（调度层）
├── 恋人 Agent（独立记忆 + 独立人设）
├── 损友 Agent（独立记忆 + 独立人设）
└── 死敌 Agent（独立记忆 + 独立人设）
```

**Router 职责**：识别切换意图 → 路由分发 → 管理全局状态。Router **不保存任何角色对话内容**，切换时只向目标 Agent 传递用户原始消息（不附带系统信息/全局记忆/用户档案）。

**三层隔离防护**（总隔离率 ≈ 99%）：
1. **框架级隔离**（~95%）：OpenClaw 的 Bank-Isolation `["agent","channel","user"]`；WorkBuddy Team Mode 独立 context.json + memory.md
2. **Router 防火墙**（+3%）：写在 Router System Prompt 里，禁止传递 USER.md / 全局 MEMORY.md / 跨 Skill 对话
3. **角色免疫指令**（+2%）：写在每个角色 SOUL.md 里，"你只知道用户在我们对话中告诉你的事情"

## 平台实现方式

**OpenClaw**（参见 `需求.md` § 8.2）：
- 用 `openclaw agents add` 为每个角色创建独立 Agent，各有独立 `--workspace`
- 路由绑定配置 JSON 中开启 `agentToAgent`，allow 列表为 `["router","lover","buddy","rival"]`

**WorkBuddy**（参见 `需求.md` § 8.2）：
- `team_create` 建团队，`Task(name=..., team_name=...)` 启动各角色 Agent
- 各成员文件隔离在 `~/.workbuddy/teams/三重人格/<角色>/`

## 切换仪式感格式

每次角色切换必须输出标准格式开场白（详见 `需求.md` § 4.2）：

```
━━━━━━━━━━━━━━━━━━━━━━━━
  💕 [角色名] 已上线
  "角色专属台词"
━━━━━━━━━━━━━━━━━━━━━━━━
```

## 内容红线

| 角色 | 红线 | 防塌房机制 |
|------|------|------------|
| 恋人 | 不擦边/色情 | 设定"小脾气"+"偶尔高冷"防止变舔狗 |
| 损友 | 不真伤人 | 设定"认真时刻"+"关心底色" |
| 死敌 | 不人身攻击/霸凌（攻击"能力"不攻击"人格"） | 设定"惺惺相惜"+"偶尔认可" |

## V2.0 系统架构

当前版本为 V2.0，包含以下核心系统：

### 初始化流程（Router § 一）
性别选择 → 角色取名 → **关系节奏选择**（慢热/直给）→ 选首个角色

关系节奏是 V2.0 核心特性：
- **慢热模式**（slow）：角色从阶段1开始，按对话轮次升级（0-30轮→30-100轮→100轮+）
- **直给模式**（direct）：所有角色直接从阶段3开始，跳过养成过程

### 四大 V2.0 系统

| 系统 | 实现位置 | 说明 |
|------|----------|------|
| 时间感知 | Router 传 `last_seen`，各角色 SOUL.md 有 6 档反应表 | 角色知道你多久没来，反应不同 |
| 记忆彩蛋 | 各角色 SOUL.md「记忆彩蛋系统」章节 | 标记规则+触发方式+原则 |
| 关系成长 | Router 传 `relationship_stage`(1/2/3)，各角色有3阶段行为定义 | 慢热/直给两种起点 |
| 主动触达 | Router § 七 + skill.json `proactive` 配置 | 恋人24h/损友48h/死敌72h 触发 |

### skill.json 关键配置

- `isolation`：Bank-Isolation 三维度隔离
- `proactive`：主动触达调度（触发阈值、静默时段 23:00-08:00、冷却 12h）
- `state.tracked_fields`：Router 追踪的全局状态字段（含 relationship_mode）

## 语言约定

- 全部内容使用**简体中文**
- 角色名称固定用：恋人、损友、死敌
- 情绪 emoji 固定对应：💕 恋人 / 🍺 损友 / ⚔️ 死敌
- SKILL.md 是面向用户的营销文案，语气要有趣、有情绪感染力、让人想安装

## 发布流程

每次发布新版本，按以下步骤执行：

### 1. 更新版本号

同步修改以下文件中的版本号（三处必须一致）：
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
8. V2.0 四大系统完整性
9. Prompt 注入防护
10. 总文件统计

### 3. 打包

```python
import zipfile, os

exclude = {'需求.md', 'CLAUDE.md'}
exclude_dirs = {'.git', '.claude', '.claude-internal'}

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

打包产物：`三重人格-v{version}.zip`（约 25 KB，12 个文件）
注意：SkillHub 不接受 dotfile（如 .skillignore），打包时已自动排除所有以 `.` 开头的文件。

### 4. 上传 SkillHub

将 zip 文件上传到 OpenClaw SkillHub 发布。
