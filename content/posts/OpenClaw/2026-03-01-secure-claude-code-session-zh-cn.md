---
title:      "玩转 OpenClaw 系列"
subtitle:   "构建安全的 Claude Code 会话执行环境"
date:       2026-03-01
author:     "p1nant0m"
draft: true
tags:
    - OpenClaw
    - Claude Code
    - Docker
    - Cloud Agent
    - 安全
categories:
    - AI Agent
description: "深入探讨如何利用 OpenClaw 和 clade-supervisor skills 构建安全的 Claude Code 会话执行环境"
---

### 背景与动机

相信大家还记得，在上一篇文章中，我们已经介绍了 OpenClaw 的基础架构和核心概念。那么问题来了：如何在一个隔离、安全的环境中运行 Claude Code？总不能让它直接在我们的生产服务器上"撒野"吧？

这就引出了本文的主题——构建安全的 Claude Code 会话执行环境。接下来，让我们一步步来实现这个方案。

---

## 核心概念：什么是安全会话执行？

在开始动手之前，我们先来聊聊什么是安全会话执行。

安全会话执行是指在一个**隔离的沙箱环境**中运行 AI 代码会话，确保：

- 🛡️ 恶意代码不会影响主机系统
- 🔒 敏感数据得到保护
- ⚙️ 资源使用得到合理控制

{{< admonition type=tip title="为什么要用沙箱？" open=false >}}
简单来说，沙箱就像是一个"隔离舱"——即使里面的代码出了幺蛾子，也不会影响到外部世界。
{{< /admonition >}}

---

## 架构设计：整体方案一览

### 2.1 系统架构

让我们先来看一下整体架构：

```
用户请求 → OpenClaw Gateway → Channel 路由 → Cloud Agent → 沙箱执行环境
```

### 2.2 核心组件

| 组件 | 职责 |
|------|------|
| **OpenClaw Gateway** | 请求入口和路由核心 |
| **Channel** | 消息通道配置（支持飞书、Telegram 等） |
| **Cloud Agent** | 执行代理，负责与 Claude Code 交互 |
| **Clade-supervisor** | 监督技能，管理会话生命周期 |

{{< admonition type=note title="组件解耦" open=false >}}
各个组件之间通过标准接口通信，这意味着我们可以灵活替换或升级某个模块而不影响整体系统。
{{< /admonition >}}

---

## Docker 沙箱配置

接下来是最核心的部分——如何配置 Docker 沙箱环境。

### 3.1 构建基础镜像

```dockerfile
FROM ubuntu:22.04

# 安装基础开发工具
RUN apt-get update && apt-get install -y \
    build-essential \
    git \
    curl \
    tmux \
    docker.io \
    && rm -rf /var/lib/apt/lists/*

# 配置 Node.js 环境
RUN curl -fsSL https://deb.nodesource.com/setup_20.x | bash - \
    && apt-get install -y nodejs

# 创建非 root 用户
RUN useradd -m -s /bin/bash developer
USER developer
```

### 3.2 资源限制配置

```yaml
# docker-compose.yml
services:
  sandbox:
    build: ./sandbox
    mem_limit: 2g
    cpu_shares: 512
    pids_limit: 100
    networks:
      - agent-network
```

{{< admonition type=warning title="资源限制很重要！" open=false >}}
一定要设置合理的资源限制，否则恶意或失控的代码可能会耗尽服务器资源，影响其他服务。
{{< /admonition >}}

---

## Channel 配置详解

OpenClaw 支持多种消息通道，让我们来看看如何配置。

### 4.1 支持的 Channel 类型

| Channel | 用途 | 配置复杂度 |
|---------|------|-----------|
| feishu | 飞书消息 | 🟡 中 |
| telegram | Telegram 机器人 | 🟢 低 |
| discord | Discord 机器人 | 🟢 低 |
| whatsapp | WhatsApp Business | 🔴 高 |

### 4.2 飞书 Channel 配置

```yaml
channels:
  feishu:
    app_id: your_app_id
    app_secret: your_secret
    verification_token: your_token
```

---

## 消息路由机制

在飞书群组中，我们可以将消息路由到不同的下游 Cloud Agent。

### 5.1 群组消息路由

```python
async def route_message(ctx: MessageContext):
    # 解析消息内容
    content = ctx.message.content
    
    # 根据关键词或用户选择路由
    if content.startswith('/code'):
        await route_to_cloud_agent('code-executor', ctx)
    elif content.startswith('/debug'):
        await route_to_cloud_agent('debugger', ctx)
    else:
        await route_to_default_agent(ctx)
```

### 5.2 路由规则配置

```yaml
routing:
  rules:
    - pattern: "^/code.*"
      agent: cloud-code-agent
      priority: 10
    - pattern: "^/deploy.*"
      agent: cloud-deploy-agent
      priority: 10
    - user: specific_user_id
      agent: admin-agent
      priority: 100
```

---

## Cloud Code Agent 核心逻辑

这是本文最硬核的部分——Cloud Code Agent 是如何工作的。

不过在深入代码之前，我们需要先准备工作环境。让我手把手教你如何搭建这套系统。

### 6.1 第一步：在 Cloud Skills Hub 中找到 Supervisor Skills

OpenClaw 提供了丰富的 Skills 生态，首先我们需要找到 **Cloud Code Supervisor Skills**。

#### 什么是 Cloud Skills Hub？

Cloud Skills Hub 是 OpenClaw 的技能市场，类似于手机应用商店。你可以在这里找到各种预置的 Agent Skills，包括：

| 技能类型 | 功能描述 | 适用场景 |
|---------|---------|---------|
| **Cloud Code Supervisor** | 管理和监督 Cloud Code 会话 | 代码执行、环境管理 |
| **Clade Supervisor** | 容器生命周期管理 | 沙箱创建、清理 |
| **Tmux Controller** | Tmux 会话控制 | 会话创建、命令注入 |
| **File Operator** | 文件系统操作 | 读取、写入、搜索文件 |

#### 如何查找和安装？

你可以使用 OpenClaw CLI 来搜索和安装 Skills：

```bash
# 查看可用的 Skills 列表
openclaw skills list

# 搜索 Cloud Code 相关的 Skills
openclaw skills search cloud-code

# 安装 Cloud Code Supervisor Skills
openclaw skills install cloud-code-supervisor
```

{{< admonition type=tip title="提示" open=false >}}
安装完成后，Skills 会保存在 `~/.openclaw/skills/` 目录下，你可以随时查看和修改它们。
{{< /admonition >}}

### 6.2 第二步：启用 Cloud Code Supervisor Skills

安装完成后，需要在配置文件中启用这个 Skill。以下是典型的配置示例：

```yaml
# openclaw.yaml
skills:
  enabled:
    - cloud-code-supervisor
    - clade-supervisor
    - tmux-controller

  cloud-code-supervisor:
    # 会话超时时间（毫秒）
    sessionTimeout: 3600000
    
    # 最大并发会话数
    maxConcurrentSessions: 5
    
    # 是否自动清理会话
    autoCleanup: true
    
    # 人工干预确认列表
    requireHumanConfirmation:
      - delete_files
      - system_modifications
      - network_changes
```

### 6.3 第三步：理解 Supervisor 的核心工作流程

现在，让我们来看看 Cloud Code Supervisor Skills 是如何工作的。

#### 核心工作流程

```
┌─────────────────────────────────────────────────────────────┐
│                    Cloud Code Supervisor                      │
├─────────────────────────────────────────────────────────────┤
│  📥 接收指令 ──→ 🎯 解析任务 ──→ 🏗️ 创建沙箱                 │
│                                                              │
│  ⌨️ 启动 Tmux ──→ 🤖 启动 Claude Code ──→ 👁️ 监督执行        │
│                                                              │
│  ⚡ 捕获输出 ──→ 🙋 处理人工介入 ──→ 🧹 清理资源             │
└─────────────────────────────────────────────────────────────┘
```

#### Prompt 示例：Supervisor 任务分配

当你向 Cloud Code Agent 发送指令时，Supervisor 会这样处理：

**用户输入**：
```
请帮我创建一个 React 项目，并实现一个简单的计数器组件
```

**Supervisor 内部 Prompt**：

```
你是一个 Cloud Code Supervisor，负责管理代码执行环境。

当前任务：创建一个 React 项目并实现计数器组件

执行步骤：
1. 检查沙箱环境是否就绪
2. 创建新的 Tmux 会话
3. 在会话中执行：npx create-react-app counter-app
4. 监听 Claude Code 的输出
5. 当 Claude Code 需要人工确认时，评估风险并决定：
   - 低风险操作（如安装 npm 包）：自动确认
   - 高风险操作（如删除文件）：请求人工确认

请开始执行并报告结果。
```

### 6.4 第四步：深入理解 Tmux 会话管理

Supervisor 通过 Tmux 来管理和控制 Cloud Code 会话，让我们来看看具体是如何实现的。

#### Tmux 会话管理架构

```
┌──────────────────────────────────────────────────────────┐
│                    Tmux Controller                         │
├──────────────────────────────────────────────────────────┤
│                                                           │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐ │
│   │ Session-001 │    │ Session-002 │    │ Session-003 │ │
│   │ (React)     │    │ (Python)    │    │ (Go)        │ │
│   └─────────────┘    └─────────────┘    └─────────────┘ │
│         │                  │                  │          │
│         └──────────────────┼──────────────────┘          │
│                            │                               │
│                    ┌───────┴───────┐                       │
│                    │  Tmux Server  │                       │
│                    │  (主进程)     │                       │
│                    └───────────────┘                       │
│                            │                               │
│                    ┌───────┴───────┐                       │
│                    │  Supervisor   │                       │
│                    │  (控制层)     │                       │
│                    └───────────────┘                       │
└──────────────────────────────────────────────────────────┘
```

#### 常用 Tmux 命令（Supervisor 内部使用）

```bash
# 创建新会话（后台运行）
tmux new-session -d -s {session_id} "bash -l"

# 发送命令到会话
tmux send-keys -t {session_id} "npm install" C-m

# 捕获会话输出
tmux capture-pane -t {session_id} -p

# 监听会话输出（实时）
tmux pipe-pane -t {session_id} "tee /tmp/{session_id}.log"

# 关闭会话
tmux kill-session -t {session_id}
```

#### Supervisor 内部的 Tmux 操作示例

以下是 Supervisor 中实际使用的代码逻辑：

```python
# 伪代码展示，实际逻辑在 Skills 中实现
class TmuxController:
    
    async def create_session(self, session_id: str, config: SandboxConfig):
        """创建新的 Tmux 会话"""
        # 1. 检查会话是否已存在
        if await self.session_exists(session_id):
            return session_id
        
        # 2. 创建新会话
        cmd = f'tmux new-session -d -s {session_id} "bash -l"'
        await self.execute(cmd)
        
        # 3. 配置管道输出（用于实时监控）
        pipe_cmd = f'tmux pipe-pane -t {session_id} "tee /tmp/{session_id}.log"'
        await self.execute(pipe_cmd)
        
        return session_id
    
    async def send_command(self, session_id: str, command: str):
        """发送命令到 Tmux 会话"""
        # 添加回车键执行命令
        cmd = f'tmux send-keys -t {session_id} "{command}" C-m'
        await self.execute(cmd)
    
    async def capture_output(self, session_id: str) -> str:
        """捕获会话当前输出"""
        cmd = f'tmux capture-pane -t {session_id} -p'
        return await self.execute(cmd)
```

### 6.5 第五步：人工操作模拟

Claude Code 有时会弹出需要人工确认的指令，Supervisor 需要处理这些情况。

#### 常见需要确认的场景

| 场景 | 风险等级 | 自动处理 |
|------|---------|---------|
| 是否允许编辑系统文件 | 🔴 高 | ❌ 需人工 |
| 是否执行破坏性操作 | 🔴 高 | ❌ 需人工 |
| 架构决策确认 | 🟡 中 | ✅ 可自动 |
| 安装 npm 包 | 🟢 低 | ✅ 可自动 |
| 创建新文件 | 🟢 低 | ✅ 可自动 |

#### 人工干预处理流程

```
┌─────────────────────────────────────────────────────────┐
│              人工干预处理流程                              │
├─────────────────────────────────────────────────────────┤
│                                                         │
│   Claude Code 输出确认提示                              │
│           │                                             │
│           ▼                                             │
│   ┌───────────────┐                                     │
│   │ 评估风险等级  │                                     │
│   └───────┬───────┘                                     │
│           │                                             │
│     ┌─────┴─────┐                                       │
│     │ 风险等级? │                                       │
│     └─────┬─────┘                                       │
│      🟢    │    🟡    │    🔴                          │
│      ▼     │     ▼     │    ▼                          │
│   自动确认  │  记录日志  │  请求人工                    │
│      │     │     │     │    │                          │
│      └─────┴─────┴─────┘    │                          │
│           │                 │                            │
│           └────────┬────────┘                           │
│                    ▼                                    │
│            返回执行结果                                  │
└─────────────────────────────────────────────────────────┘
```

#### Prompt 示例：人工干预判断

```
你正在监督 Claude Code 的执行。

检测到以下需要确认的提示：
> "Do you want to allow this tool to edit system files? (y/n)"

根据以下规则判断：
1. 如果涉及系统文件修改 → 标记为高风险，请求人工确认
2. 如果是项目内部文件 → 低风险，可以自动确认
3. 如果是危险操作（删除、格式化）→ 直接拒绝

当前操作：尝试修改 /etc/hosts

请给出你的判断和操作。
```

{{< admonition type=warning title="安全提醒" open=false >}}
对于高风险操作，建议始终交由人工确认，不要完全依赖自动化判断。安全永远是第一位的。
{{< /admonition >}}

---

## 完整实现示例

让我们来看一个完整的实现——OpenClaw Agent 如何编排各个 Supervisor Skills。

```python
class OpenClawAgent:
    """上层编排Agent，负责协调多个Cloud Agent"""
    
    def __init__(self):
        self.code_agent = CloudCodeAgent()
        self.supervisor = CladeSupervisor()
    
    async def execute_task(self, task: Task):
        # 1. 任务解析
        plan = await self.parse_task(task)
        
        # 2. 创建执行环境
        sandbox = await self.supervisor.create_sandbox(plan.requirements)
        
        # 3. 协调执行
        try:
            result = await self.code_agent.run(sandbox, plan.instructions)
        except HumanInterventionRequired as e:
            # 需要人工介入
            result = await self.handle_intervention(e)
        
        # 4. 清理资源
        await self.supervisor.cleanup(sandbox)
        
        return result
```

---

## 安全最佳实践

最后，让我们总结一下安全最佳实践。

### 🛡️ 网络隔离

- 使用网络命名空间限制容器网络访问
- 只允许必要的出站连接
- 禁用危险的网络功能

### ⚙️ 资源限制

- 设置内存上限
- 限制 CPU 使用
- 控制最大进程数

### 📝 操作审计

- 记录所有执行的命令
- 审计文件系统变更
- 监控异常行为

### 🔒 敏感数据保护

- 不在沙箱中持久化敏感信息
- 使用临时文件系统
- 定期清理会话数据

{{< admonition type=quote title="安全无小事" open=false >}}
安全问题永远不要掉以轻心，多一层防护就少一分风险。
{{< /admonition >}}

## 总结

通过本文，我们学习了：

1. 🏗️ 如何设计安全会话执行的架构
2. 🐳 如何配置 Docker 沙箱环境
3. 📡 如何配置 Channel 与消息路由
4. 🤖 Cloud Code Agent 的核心实现
5. 👆 如何模拟人工操作行为
6. 🛡️ 安全最佳实践

下一篇文章我们将介绍如何在生产环境中部署和监控这个系统，敬请期待！

#### 参考资料

- [OpenClaw 官方文档](https://docs.openclaw.ai)
- [Docker 安全最佳实践](https://docs.docker.com/engine/security/)
- [Tmux 使用指南](https://github.com/tmux/tmux/wiki)

{{< admonition type=note title="胡思乱想" open=false >}}
写这篇文章的时候，我一直在思考：AI Agent 的安全性究竟该如何保障？除了技术手段，是否还需要一些制度层面的约束？毕竟，技术的归技术，制度的归制度。或许在后续的文章中，我们可以深入探讨这个话题。
{{< /admonition >}}
