<picture>
  <source media="(prefers-color-scheme: dark)" srcset="../resources/logos/claude-howto-logo-dark.svg">
  <img alt="Claude How To" src="../resources/logos/claude-howto-logo.svg">
</picture>

# 钩子

钩子是在 Claude Code 会话期间响应特定事件自动执行的脚本。它们实现自动化、验证、权限管理和自定义工作流。

## 概述

钩子是当 Claude Code 中发生特定事件时自动执行的操作（shell 命令、HTTP webhook、LLM 提示或子代理评估）。它们接收 JSON 输入并通过退出码和 JSON 输出传递结果。

**主要功能：**
- 事件驱动的自动化
- 基于 JSON 的输入/输出
- 支持 command、prompt、HTTP 和 agent 钩子类型
- 针对特定工具的模式匹配

## 配置

钩子在设置文件中使用特定结构进行配置：

- `~/.claude/settings.json` - 用户设置（所有项目）
- `.claude/settings.json` - 项目设置（可共享，提交）
- `.claude/settings.local.json` - 本地项目设置（不提交）
- 托管策略 - 组织范围的设置
- 插件 `hooks/hooks.json` - 插件范围的钩子
- 技能/代理 frontmatter - 组件生命周期钩子

### 基本配置结构

```json
{
  "hooks": {
    "EventName": [
      {
        "matcher": "ToolPattern",
        "hooks": [
          {
            "type": "command",
            "command": "your-command-here",
            "timeout": 60
          }
        ]
      }
    ]
  }
}
```

**关键字段：**

| 字段 | 描述 | 示例 |
|-------|-------------|---------|
| `matcher` | 匹配工具名称的模式（区分大小写） | `"Write"`、`"Edit\|Write"`、`"*"` |
| `hooks` | 钩子定义数组 | `[{ "type": "command", ... }]` |
| `type` | 钩子类型：`"command"`（bash）、`"prompt"`（LLM）、`"http"`（webhook）或 `"agent"`（子代理） | `"command"` |
| `command` | 要执行的 shell 命令 | `"$CLAUDE_PROJECT_DIR/.claude/hooks/format.sh"` |
| `timeout` | 可选超时（秒）（默认 60） | `30` |
| `once` | 如果为 `true`，每个会话只运行一次钩子 | `true` |

### 匹配器模式

| 模式 | 描述 | 示例 |
|---------|-------------|---------|
| 精确字符串 | 匹配特定工具 | `"Write"` |
| 正则模式 | 匹配多个工具 | `"Edit\|Write"` |
| 通配符 | 匹配所有工具 | `"*"` 或 `""` |
| MCP 工具 | 服务器和工具模式 | `"mcp__memory__.*"` |

## 钩子类型

Claude Code 支持四种钩子类型：

### 命令钩子

默认钩子类型。执行 shell 命令并通过 JSON stdin/stdout 和退出码传递结果。

```json
{
  "type": "command",
  "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/validate.py\"",
  "timeout": 60
}
```

### HTTP 钩子

> 在 v2.1.63 中添加。

远程 webhook 端点，接收与命令钩子相同的 JSON 输入。HTTP 钩子将 JSON POST 到 URL 并接收 JSON 响应。启用沙箱时，HTTP 钩子通过沙箱路由。URL 中的环境变量插值需要显式的 `allowedEnvVars` 列表以确保安全。

```json
{
  "hooks": {
    "PostToolUse": [{
      "type": "http",
      "url": "https://my-webhook.example.com/hook",
      "matcher": "Write"
    }]
  }
}
```

**关键属性：**
- `"type": "http"` -- 将此标识为 HTTP 钩子
- `"url"` -- webhook 端点 URL
- 启用沙箱时通过沙箱路由
- URL 中的任何环境变量插值需要显式的 `allowedEnvVars` 列表

### 提示钩子

LLM 评估的提示，其中钩子内容是 Claude 评估的提示。主要用于 `Stop` 和 `SubagentStop` 事件进行智能任务完成检查。

```json
{
  "type": "prompt",
  "prompt": "评估 Claude 是否完成了所有请求的任务。",
  "timeout": 30
}
```

LLM 评估提示并返回结构化决策（详见 [基于提示的钩子](#基于提示的钩子)）。

### 代理钩子

基于子代理的验证钩子，生成专用代理来评估条件或执行复杂检查。与提示钩子（单轮 LLM 评估）不同，代理钩子可以使用工具并执行多步推理。

```json
{
  "type": "agent",
  "prompt": "验证代码更改遵循我们的架构指南。检查相关设计文档并进行比较。",
  "timeout": 120
}
```

**关键属性：**
- `"type": "agent"` -- 将此标识为代理钩子
- `"prompt"` -- 子代理的任务描述
- 代理可以使用工具（Read、Grep、Bash 等）执行评估
- 返回类似提示钩子的结构化决策

## 钩子事件

Claude Code 支持 **25 个钩子事件**：

| 事件 | 触发时机 | 匹配器输入 | 可阻塞 | 常见用途 |
|-------|---------------|---------------|-----------|------------|
| **SessionStart** | 会话开始/恢复/清除/压缩 | startup/resume/clear/compact | 否 | 环境设置 |
| **InstructionsLoaded** | 加载 CLAUDE.md 或规则文件后 | （无） | 否 | 修改/过滤指令 |
| **UserPromptSubmit** | 用户提交提示 | （无） | 是 | 验证提示 |
| **PreToolUse** | 工具执行前 | 工具名称 | 是（允许/拒绝/询问） | 验证、修改输入 |
| **PermissionRequest** | 显示权限对话框 | 工具名称 | 是 | 自动批准/拒绝 |
| **PostToolUse** | 工具成功后 | 工具名称 | 否 | 添加上下文、反馈 |
| **PostToolUseFailure** | 工具执行失败 | 工具名称 | 否 | 错误处理、日志 |
| **Notification** | 发送通知 | 通知类型 | 否 | 自定义通知 |
| **SubagentStart** | 子代理生成 | 代理类型名称 | 否 | 子代理设置 |
| **SubagentStop** | 子代理完成 | 代理类型名称 | 是 | 子代理验证 |
| **Stop** | Claude 完成响应 | （无） | 是 | 任务完成检查 |
| **StopFailure** | API 错误结束轮次 | （无） | 否 | 错误恢复、日志 |
| **TeammateIdle** | 代理团队队友空闲 | （无） | 是 | 队友协调 |
| **TaskCompleted** | 任务标记完成 | （无） | 是 | 任务后操作 |
| **TaskCreated** | 通过 TaskCreate 创建任务 | （无） | 否 | 任务跟踪、日志 |
| **ConfigChange** | 配置文件更改 | （无） | 是（策略除外） | 响应配置更新 |
| **CwdChanged** | 工作目录更改 | （无） | 否 | 目录特定设置 |
| **FileChanged** | 监视文件更改 | （无） | 否 | 文件监视、重建 |
| **PreCompact** | 上下文压缩前 | manual/auto | 否 | 压缩前操作 |
| **PostCompact** | 压缩完成后 | （无） | 否 | 压缩后操作 |
| **WorktreeCreate** | 正在创建 worktree | （无） | 是（返回路径） | Worktree 初始化 |
| **WorktreeRemove** | 正在删除 worktree | （无） | 否 | Worktree 清理 |
| **Elicitation** | MCP 服务器请求用户输入 | （无） | 是 | 输入验证 |
| **ElicitationResult** | 用户响应请求 | （无） | 是 | 响应处理 |
| **SessionEnd** | 会话终止 | （无） | 否 | 清理、最终日志 |

### PreToolUse

在 Claude 创建工具参数之后、处理之前运行。使用此验证或修改工具输入。

**配置：**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/validate-bash.py"
          }
        ]
      }
    ]
  }
}
```

**常见匹配器：** `Task`、`Bash`、`Glob`、`Grep`、`Read`、`Edit`、`Write`、`WebFetch`、`WebSearch`

**输出控制：**
- `permissionDecision`：`"allow"`、`"deny"` 或 `"ask"`
- `permissionDecisionReason`：决策原因说明
- `updatedInput`：修改后的工具输入参数

### PostToolUse

在工具完成后立即运行。用于验证、日志记录或将上下文反馈给 Claude。

**配置：**
```json
{
  "hooks": {
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/security-scan.py"
          }
        ]
      }
    ]
  }
}
```

**输出控制：**
- `"block"` 决策向 Claude 提示反馈
- `additionalContext`：为 Claude 添加的上下文

### UserPromptSubmit

用户提交提示时运行，在 Claude 处理之前。

**配置：**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/validate-prompt.py"
          }
        ]
      }
    ]
  }
}
```

**输出控制：**
- `decision`：`"block"` 阻止处理
- `reason`：如果阻塞的原因说明
- `additionalContext`：添加到提示的上下文

### Stop 和 SubagentStop

当 Claude 完成响应（Stop）或子代理完成（SubagentStop）时运行。支持基于提示的评估进行智能任务完成检查。

**额外输入字段：** `Stop` 和 `SubagentStop` 钩子都在其 JSON 输入中接收 `last_assistant_message` 字段，包含停止前 Claude 或子代理的最后一条消息。这对于评估任务完成很有用。

**配置：**
```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "评估 Claude 是否完成了所有请求的任务。",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### SubagentStart

子代理开始执行时运行。匹配器输入是代理类型名称，允许钩子针对特定子代理类型。

**配置：**
```json
{
  "hooks": {
    "SubagentStart": [
      {
        "matcher": "code-review",
        "hooks": [
          {
            "type": "command",
            "command": "$CLAUDE_PROJECT_DIR/.claude/hooks/subagent-init.sh"
          }
        ]
      }
    ]
  }
}
```

### SessionStart

会话开始或恢复时运行。可以持久化环境变量。

**匹配器：** `startup`、`resume`、`clear`、`compact`

**特殊功能：** 使用 `CLAUDE_ENV_FILE` 持久化环境变量（在 `CwdChanged` 和 `FileChanged` 钩子中也可用）：

```bash
#!/bin/bash
if [ -n "$CLAUDE_ENV_FILE" ]; then
  echo 'export NODE_ENV=development' >> "$CLAUDE_ENV_FILE"
fi
exit 0
```

### SessionEnd

会话结束时运行以执行清理或最终日志记录。无法阻塞终止。

**原因字段值：**
- `clear` - 用户清除了会话
- `logout` - 用户退出登录
- `prompt_input_exit` - 用户通过提示输入退出
- `other` - 其他原因

**配置：**
```json
{
  "hooks": {
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/session-cleanup.sh\""
          }
        ]
      }
    ]
  }
}
```

### 通知事件

更新通知事件的匹配器：
- `permission_prompt` - 权限请求通知
- `idle_prompt` - 空闲状态通知
- `auth_success` - 身份验证成功
- `elicitation_dialog` - 向用户显示对话框

## 组件范围的钩子

钩子可以附加到特定组件（技能、代理、命令）在其 frontmatter 中：

**在 SKILL.md、agent.md 或 command.md 中：**

```yaml
---
name: secure-operations
description: 执行带有安全检查的操作
hooks:
  PreToolUse:
    - matcher: "Bash"
      hooks:
        - type: command
          command: "./scripts/check.sh"
          once: true  # 每个会话只运行一次
---
```

**组件钩子支持的事件：** `PreToolUse`、`PostToolUse`、`Stop`

这允许直接在使用它们的组件中定义钩子，保持相关代码在一起。

### 子代理 Frontmatter 中的钩子

当在子代理的 frontmatter 中定义 `Stop` 钩子时，它会自动转换为该子代理范围的 `SubagentStop` 钩子。这确保停止钩子只在该特定子代理完成时触发，而不是主会话停止时。

```yaml
---
name: code-review-agent
description: 自动代码审查子代理
hooks:
  Stop:
    - hooks:
        - type: prompt
          prompt: "验证代码审查全面且完整。"
  # 上面的 Stop 钩子自动转换为该子代理的 SubagentStop
---
```

## PermissionRequest 事件

使用自定义输出格式处理权限请求：

```json
{
  "hookSpecificOutput": {
    "hookEventName": "PermissionRequest",
    "decision": {
      "behavior": "allow|deny",
      "updatedInput": {},
      "message": "自定义消息",
      "interrupt": false
    }
  }
}
```

## 钩子输入和输出

### JSON 输入（通过 stdin）

所有钩子通过 stdin 接收 JSON 输入：

```json
{
  "session_id": "abc123",
  "transcript_path": "/path/to/transcript.jsonl",
  "cwd": "/current/working/directory",
  "permission_mode": "default",
  "hook_event_name": "PreToolUse",
  "tool_name": "Write",
  "tool_input": {
    "file_path": "/path/to/file.js",
    "content": "..."
  },
  "tool_use_id": "toolu_01ABC123...",
  "agent_id": "agent-abc123",
  "agent_type": "main",
  "worktree": "/path/to/worktree"
}
```

**常见字段：**

| 字段 | 描述 |
|-------|-------------|
| `session_id` | 唯一会话标识符 |
| `transcript_path` | 对话转录文件路径 |
| `cwd` | 当前工作目录 |
| `hook_event_name` | 触发钩子的事件名称 |
| `agent_id` | 运行此钩子的代理标识符 |
| `agent_type` | 代理类型（`"main"`、子代理类型名称等） |
| `worktree` | git worktree 的路径（如果代理正在其中运行） |

### 退出码

| 退出码 | 含义 | 行为 |
|-----------|---------|----------|
| **0** | 成功 | 继续，解析 JSON stdout |
| **2** | 阻塞错误 | 阻塞操作，stderr 显示为错误 |
| **其他** | 非阻塞错误 | 继续，stderr 在详细模式下显示 |

### JSON 输出（stdout，退出码 0）

```json
{
  "continue": true,
  "stopReason": "停止时的可选消息",
  "suppressOutput": false,
  "systemMessage": "可选警告消息",
  "hookSpecificOutput": {
    "hookEventName": "PreToolUse",
    "permissionDecision": "allow",
    "permissionDecisionReason": "文件在允许目录中",
    "updatedInput": {
      "file_path": "/modified/path.js"
    }
  }
}
```

## 环境变量

| 变量 | 可用性 | 描述 |
|----------|-------------|-------------|
| `CLAUDE_PROJECT_DIR` | 所有钩子 | 项目根目录的绝对路径 |
| `CLAUDE_ENV_FILE` | SessionStart、CwdChanged、FileChanged | 持久化环境变量的文件路径 |
| `CLAUDE_CODE_REMOTE` | 所有钩子 | 如果在远程环境中运行则为 `"true"` |
| `${CLAUDE_PLUGIN_ROOT}` | 插件钩子 | 插件目录路径 |
| `${CLAUDE_PLUGIN_DATA}` | 插件钩子 | 插件数据目录路径 |
| `CLAUDE_CODE_SESSIONEND_HOOKS_TIMEOUT_MS` | SessionEnd 钩子 | SessionEnd 钩子的可配置超时（毫秒）（覆盖默认值） |

## 基于提示的钩子

对于 `Stop` 和 `SubagentStop` 事件，可以使用基于 LLM 的评估：

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "审查所有任务是否完成。返回您的决策。",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

**LLM 响应 Schema：**
```json
{
  "decision": "approve",
  "reason": "所有任务成功完成",
  "continue": false,
  "stopReason": "任务完成"
}
```

## 示例

### 示例 1：Bash 命令验证器（PreToolUse）

**文件：** `.claude/hooks/validate-bash.py`

```python
#!/usr/bin/env python3
import json
import sys
import re

BLOCKED_PATTERNS = [
    (r"\brm\s+-rf\s+/", "阻塞危险的 rm -rf / 命令"),
    (r"\bsudo\s+rm", "阻塞 sudo rm 命令"),
]

def main():
    input_data = json.load(sys.stdin)

    tool_name = input_data.get("tool_name", "")
    if tool_name != "Bash":
        sys.exit(0)

    command = input_data.get("tool_input", {}).get("command", "")

    for pattern, message in BLOCKED_PATTERNS:
        if re.search(pattern, command):
            print(message, file=sys.stderr)
            sys.exit(2)  # 退出码 2 = 阻塞错误

    sys.exit(0)

if __name__ == "__main__":
    main()
```

**配置：**
```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/validate-bash.py\""
          }
        ]
      }
    ]
  }
}
```

### 示例 2：安全扫描器（PostToolUse）

**文件：** `.claude/hooks/security-scan.py`

```python
#!/usr/bin/env python3
import json
import sys
import re

SECRET_PATTERNS = [
    (r"password\s*=\s*['\"][^'\"]+['\"]", "潜在的硬编码密码"),
    (r"api[_-]?key\s*=\s*['\"][^'\"]+['\"]", "潜在的硬编码 API 密钥"),
]

def main():
    input_data = json.load(sys.stdin)

    tool_name = input_data.get("tool_name", "")
    if tool_name not in ["Write", "Edit"]:
        sys.exit(0)

    tool_input = input_data.get("tool_input", {})
    content = tool_input.get("content", "") or tool_input.get("new_string", "")
    file_path = tool_input.get("file_path", "")

    warnings = []
    for pattern, message in SECRET_PATTERNS:
        if re.search(pattern, content, re.IGNORECASE):
            warnings.append(message)

    if warnings:
        output = {
            "hookSpecificOutput": {
                "hookEventName": "PostToolUse",
                "additionalContext": f"{file_path} 的安全警告：" + "; ".join(warnings)
            }
        }
        print(json.dumps(output))

    sys.exit(0)

if __name__ == "__main__":
    main()
```

### 示例 3：自动格式化代码（PostToolUse）

**文件：** `.claude/hooks/format-code.sh`

```bash
#!/bin/bash

# 从 stdin 读取 JSON
INPUT=$(cat)
TOOL_NAME=$(echo "$INPUT" | python3 -c "import sys, json; print(json.load(sys.stdin).get('tool_name', ''))")
FILE_PATH=$(echo "$INPUT" | python3 -c "import sys, json; print(json.load(sys.stdin).get('tool_input', {}).get('file_path', ''))")

if [ "$TOOL_NAME" != "Write" ] && [ "$TOOL_NAME" != "Edit" ]; then
    exit 0
fi

# 根据文件扩展名格式化
case "$FILE_PATH" in
    *.js|*.jsx|*.ts|*.tsx|*.json)
        command -v prettier &>/dev/null && prettier --write "$FILE_PATH" 2>/dev/null
        ;;
    *.py)
        command -v black &>/dev/null && black "$FILE_PATH" 2>/dev/null
        ;;
    *.go)
        command -v gofmt &>/dev/null && gofmt -w "$FILE_PATH" 2>/dev/null
        ;;
esac

exit 0
```

### 示例 4：提示验证器（UserPromptSubmit）

**文件：** `.claude/hooks/validate-prompt.py`

```python
#!/usr/bin/env python3
import json
import sys
import re

BLOCKED_PATTERNS = [
    (r"delete\s+(all\s+)?database", "危险：数据库删除"),
    (r"rm\s+-rf\s+/", "危险：根目录删除"),
]

def main():
    input_data = json.load(sys.stdin)
    prompt = input_data.get("user_prompt", "") or input_data.get("prompt", "")

    for pattern, message in BLOCKED_PATTERNS:
        if re.search(pattern, prompt, re.IGNORECASE):
            output = {
                "decision": "block",
                "reason": f"已阻塞：{message}"
            }
            print(json.dumps(output))
            sys.exit(0)

    sys.exit(0)

if __name__ == "__main__":
    main()
```

### 示例 5：智能停止钩子（基于提示）

```json
{
  "hooks": {
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "审查 Claude 是否完成了所有请求的任务。检查：1) 所有文件是否已创建/修改？2) 是否有未解决的错误？如果不完整，解释缺少什么。",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

### 示例 6：上下文使用跟踪器（钩子对）

使用 `UserPromptSubmit`（消息前）和 `Stop`（响应后）钩子跟踪每个请求的令牌消耗。

**文件：** `.claude/hooks/context-tracker.py`

```python
#!/usr/bin/env python3
"""
上下文使用跟踪器 - 跟踪每个请求的令牌消耗。

使用 UserPromptSubmit 作为"消息前"钩子，Stop 作为"响应后"钩子
来计算每个请求的令牌使用增量。

令牌计数方法：
1. 字符估算（默认）：每令牌约 4 个字符，无依赖
2. tiktoken（可选）：更准确（约 90-95%），需要：pip install tiktoken
"""
import json
import os
import sys
import tempfile

# 配置
CONTEXT_LIMIT = 128000  # Claude 的上下文窗口（根据您的模型调整）
USE_TIKTOKEN = False    # 如果安装了 tiktoken 以获得更好的准确性，设置为 True


def get_state_file(session_id: str) -> str:
    """获取存储消息前令牌计数的临时文件路径，按会话隔离。"""
    return os.path.join(tempfile.gettempdir(), f"claude-context-{session_id}.json")


def count_tokens(text: str) -> int:
    """
    计算文本中的令牌数。

    如果可用，使用带 p50k_base 编码的 tiktoken（约 90-95% 准确度），
    否则回退到字符估算（约 80-90% 准确度）。
    """
    if USE_TIKTOKEN:
        try:
            import tiktoken
            enc = tiktoken.get_encoding("p50k_base")
            return len(enc.encode(text))
        except ImportError:
            pass  # 回退到估算

    # 基于字符的估算：英语每令牌约 4 个字符
    return len(text) // 4


def read_transcript(transcript_path: str) -> str:
    """从转录文件读取并连接所有内容。"""
    if not transcript_path or not os.path.exists(transcript_path):
        return ""

    content = []
    with open(transcript_path, "r") as f:
        for line in f:
            try:
                entry = json.loads(line.strip())
                # 从各种消息格式中提取文本内容
                if "message" in entry:
                    msg = entry["message"]
                    if isinstance(msg.get("content"), str):
                        content.append(msg["content"])
                    elif isinstance(msg.get("content"), list):
                        for block in msg["content"]:
                            if isinstance(block, dict) and block.get("type") == "text":
                                content.append(block.get("text", ""))
            except json.JSONDecodeError:
                continue

    return "\n".join(content)


def handle_user_prompt_submit(data: dict) -> None:
    """消息前钩子：在请求前保存当前令牌计数。"""
    session_id = data.get("session_id", "unknown")
    transcript_path = data.get("transcript_path", "")

    transcript_content = read_transcript(transcript_path)
    current_tokens = count_tokens(transcript_content)

    # 保存到临时文件以供稍后比较
    state_file = get_state_file(session_id)
    with open(state_file, "w") as f:
        json.dump({"pre_tokens": current_tokens}, f)


def handle_stop(data: dict) -> None:
    """响应后钩子：计算并报告令牌增量。"""
    session_id = data.get("session_id", "unknown")
    transcript_path = data.get("transcript_path", "")

    transcript_content = read_transcript(transcript_path)
    current_tokens = count_tokens(transcript_content)

    # 加载消息前计数
    state_file = get_state_file(session_id)
    pre_tokens = 0
    if os.path.exists(state_file):
        try:
            with open(state_file, "r") as f:
                state = json.load(f)
                pre_tokens = state.get("pre_tokens", 0)
        except (json.JSONDecodeError, IOError):
            pass

    # 计算增量
    delta_tokens = current_tokens - pre_tokens
    remaining = CONTEXT_LIMIT - current_tokens
    percentage = (current_tokens / CONTEXT_LIMIT) * 100

    # 报告使用情况
    method = "tiktoken" if USE_TIKTOKEN else "估算"
    print(f"上下文（{method}）：约 {current_tokens:,} 个令牌（{percentage:.1f}% 已使用，约 {remaining:,} 剩余）", file=sys.stderr)
    if delta_tokens > 0:
        print(f"本次请求：约 {delta_tokens:,} 个令牌", file=sys.stderr)


def main():
    data = json.load(sys.stdin)
    event = data.get("hook_event_name", "")

    if event == "UserPromptSubmit":
        handle_user_prompt_submit(data)
    elif event == "Stop":
        handle_stop(data)

    sys.exit(0)


if __name__ == "__main__":
    main()
```

**配置：**
```json
{
  "hooks": {
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/context-tracker.py\""
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/context-tracker.py\""
          }
        ]
      }
    ]
  }
}
```

**工作原理：**
1. `UserPromptSubmit` 在处理提示之前触发 - 保存当前令牌计数
2. `Stop` 在 Claude 响应后触发 - 计算增量并报告使用情况
3. 每个会话通过临时文件名中的 `session_id` 隔离

**令牌计数方法：**

| 方法 | 准确度 | 依赖 | 速度 |
|--------|----------|--------------|-------|
| 字符估算 | 约 80-90% | 无 | <1ms |
| tiktoken (p50k_base) | 约 90-95% | `pip install tiktoken` | <10ms |

> **注意：** Anthropic 尚未发布官方离线分词器。两种方法都是近似值。转录包括用户提示、Claude 响应和工具输出，但不包括系统提示或内部上下文。

### 示例 7：种子自动模式权限（一次性设置脚本）

一次性设置脚本，将约 67 个安全权限规则种子化到 `~/.claude/settings.json`，等效于 Claude Code 的自动模式基线 —— 无需任何钩子，无需记住未来选择。运行一次；可以安全重复运行（跳过已存在的规则）。

**文件：** `09-advanced-features/setup-auto-mode-permissions.py`

```bash
# 预览将添加的内容
python3 09-advanced-features/setup-auto-mode-permissions.py --dry-run

# 应用
python3 09-advanced-features/setup-auto-mode-permissions.py
```

**添加的内容：**

| 类别 | 示例 |
|----------|---------|
| 内置工具 | `Read(*)`、`Edit(*)`、`Write(*)`、`Glob(*)`、`Grep(*)`、`Agent(*)`、`WebSearch(*)` |
| Git 读取 | `Bash(git status:*)`、`Bash(git log:*)`、`Bash(git diff:*)` |
| Git 写入（本地） | `Bash(git add:*)`、`Bash(git commit:*)`、`Bash(git checkout:*)` |
| 包管理器 | `Bash(npm install:*)`、`Bash(pip install:*)`、`Bash(cargo build:*)` |
| 构建和测试 | `Bash(make:*)`、`Bash(pytest:*)`、`Bash(go test:*)` |
| 常用 shell | `Bash(ls:*)`、`Bash(cat:*)`、`Bash(find:*)`、`Bash(cp:*)`、`Bash(mv:*)` |
| GitHub CLI | `Bash(gh pr view:*)`、`Bash(gh pr create:*)`、`Bash(gh issue list:*)` |

**有意排除的内容**（此脚本从不添加）：
- `rm -rf`、`sudo`、force push、`git reset --hard`
- `DROP TABLE`、`kubectl delete`、`terraform destroy`
- `npm publish`、`curl | bash`、生产部署

## 插件钩子

插件可以在其 `hooks/hooks.json` 文件中包含钩子：

**文件：** `plugins/hooks/hooks.json`

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "${CLAUDE_PLUGIN_ROOT}/scripts/validate.sh"
          }
        ]
      }
    ]
  }
}
```

**插件钩子中的环境变量：**
- `${CLAUDE_PLUGIN_ROOT}` - 插件目录路径
- `${CLAUDE_PLUGIN_DATA}` - 插件数据目录路径

这允许插件包含自定义验证和自动化钩子。

## MCP 工具钩子

MCP 工具遵循模式 `mcp__<server>__<tool>`：

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "mcp__memory__.*",
        "hooks": [
          {
            "type": "command",
            "command": "echo '{\"systemMessage\": \"记忆操作已记录\"}'"
          }
        ]
      }
    ]
  }
}
```

## 安全考虑

### 免责声明

**使用风险自负**：钩子执行任意 shell 命令。您对以下内容负全责：
- 您配置的命令
- 文件访问/修改权限
- 潜在的数据丢失或系统损坏
- 在生产使用前在安全环境中测试钩子

### 安全说明

- **需要工作区信任：** `statusLine` 和 `fileSuggestion` 钩子输出命令现在需要接受工作区信任才能生效。
- **HTTP 钩子和环境变量：** HTTP 钩子需要显式的 `allowedEnvVars` 列表才能在 URL 中使用环境变量插值。这可以防止敏感环境变量意外泄露到远程端点。
- **托管设置层次结构：** `disableAllHooks` 设置现在遵循托管设置层次结构，意味着组织级设置可以强制禁用个别用户无法覆盖的钩子。

### 最佳实践

| 应该 | 不应该 |
|-----|-------|
| 验证和清理所有输入 | 盲目信任输入数据 |
| 引用 shell 变量：`"$VAR"` | 使用未引用：`$VAR` |
| 阻塞路径遍历（`..`） | 允许任意路径 |
| 使用带 `$CLAUDE_PROJECT_DIR` 的绝对路径 | 硬编码路径 |
| 跳过敏感文件（`.env`、`.git/`、密钥） | 处理所有文件 |
| 首先隔离测试钩子 | 部署未测试的钩子 |
| 对 HTTP 钩子使用显式的 `allowedEnvVars` | 将所有环境变量暴露给 webhook |

## 调试

### 启用调试模式

使用调试标志运行 Claude 以获取详细的钩子日志：

```bash
claude --debug
```

### 详细模式

在 Claude Code 中使用 `Ctrl+O` 启用详细模式并查看钩子执行进度。

### 独立测试钩子

```bash
# 使用示例 JSON 输入测试
echo '{"tool_name": "Bash", "tool_input": {"command": "ls -la"}}' | python3 .claude/hooks/validate-bash.py

# 检查退出码
echo $?
```

## 完整配置示例

```json
{
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/validate-bash.py\"",
            "timeout": 10
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Write|Edit",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/format-code.sh\"",
            "timeout": 30
          },
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/security-scan.py\"",
            "timeout": 10
          }
        ]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "command",
            "command": "python3 \"$CLAUDE_PROJECT_DIR/.claude/hooks/validate-prompt.py\""
          }
        ]
      }
    ],
    "SessionStart": [
      {
        "matcher": "startup",
        "hooks": [
          {
            "type": "command",
            "command": "\"$CLAUDE_PROJECT_DIR/.claude/hooks/session-init.sh\""
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "prompt",
            "prompt": "在停止前验证所有任务已完成。",
            "timeout": 30
          }
        ]
      }
    ]
  }
}
```

## 钩子执行详情

| 方面 | 行为 |
|--------|----------|
| **超时** | 默认 60 秒，每个命令可配置 |
| **并行化** | 所有匹配的钩子并行运行 |
| **去重** | 相同的钩子命令去重 |
| **环境** | 在当前目录中使用 Claude Code 的环境运行 |

## 故障排除

### 钩子未执行
- 验证 JSON 配置语法正确
- 检查匹配器模式匹配工具名称
- 确保脚本存在且可执行：`chmod +x script.sh`
- 运行 `claude --debug` 查看钩子执行日志
- 验证钩子从 stdin 读取 JSON（不是命令参数）

### 钩子意外阻塞
- 使用示例 JSON 测试钩子：`echo '{"tool_name": "Write", ...}' | ./hook.py`
- 检查退出码：应该为 0 表示允许，2 表示阻塞
- 检查 stderr 输出（在退出码 2 时显示）

### JSON 解析错误
- 始终从 stdin 读取，而不是命令参数
- 使用正确的 JSON 解析（不是字符串操作）
- 优雅处理缺失字段

## 安装

### 步骤 1：创建钩子目录
```bash
mkdir -p ~/.claude/hooks
```

### 步骤 2：复制示例钩子
```bash
cp 06-hooks/*.sh ~/.claude/hooks/
chmod +x ~/.claude/hooks/*.sh
```

### 步骤 3：在设置中配置
使用上面显示的钩子配置编辑 `~/.claude/settings.json` 或 `.claude/settings.json`。

## 相关概念

- **[检查点和回退](../08-checkpoints/)** - 保存和恢复对话状态
- **[斜杠命令](../01-slash-commands/)** - 创建自定义斜杠命令
- **[技能](../03-skills/)** - 可复用的自主能力
- **[子代理](../04-subagents/)** - 委托的任务执行
- **[插件](../07-plugins/)** - 捆绑的扩展包
- **[高级功能](../09-advanced-features/)** - 探索 Claude Code 高级功能

## 其他资源

- **[官方钩子文档](https://code.claude.com/docs/en/hooks)** - 完整钩子参考
- **[CLI 参考](https://code.claude.com/docs/en/cli-reference)** - 命令行界面文档
- **[记忆指南](../02-memory/)** - 持久上下文配置