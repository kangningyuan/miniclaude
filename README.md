# miniclaude
A simple implement for claude code in python
## 概述

`s13_full_implement.py` 是一个功能完整的 AI 智能体实现，将所有章节 (s01-s11) 的机制整合到一个统一的系统中。
> 这是 s01-s11 所有机制的集大成实现

```
+------------------------------------------------------------------+
|                        FULL AGENT                                 |
|                                                                   |
|  System prompt (s05 skills, task-first + optional todo nag)      |
|                                                                   |
|  Before each LLM call:                                            |
|  +--------------------+  +------------------+  +--------------+  |
|  | Microcompact (s06) |  | Drain bg (s08)   |  | Check inbox  |  |
|  | Auto-compact (s06) |  | notifications    |  | (s09)        |  |
|  +--------------------+  +------------------+  +--------------+  |
|                                                                   |
|  Tool dispatch (s02 pattern):                                     |
|  +--------+----------+----------+---------+-----------+          |
|  | bash   | read     | write    | edit    | TodoWrite |          |
|  | task   | load_sk  | compress | bg_run  | bg_check  |          |
|  | t_crt  | t_get    | t_upd    | t_list  | spawn_tm  |          |
|  | list_tm| send_msg | rd_inbox | bcast   | shutdown  |          |
|  | plan   | idle     | claim    |         |           |          |
|  +--------+----------+----------+---------+-----------+          |
|                                                                   |
|  Subagent (s04):  spawn -> work -> return summary                 |
|  Teammate (s09):  spawn -> work -> idle -> auto-claim (s11)      |
|  Shutdown (s10):  request_id handshake                              |
|  Plan gate (s10): submit -> approve/reject                        |
+------------------------------------------------------------------+
```

## 核心特性

### 1. 文件操作工具 (s02)
- **bash**: 运行 shell 命令（带安全过滤）
- **read_file**: 读取文件内容，支持行数限制
- **write_file**: 写入文件内容
- **edit_file**: 精确文本替换编辑

### 2. 任务管理 (s03/s07)
- **TodoWrite**: 轻量级待办事项跟踪（最多20项，仅1项进行中）
- **任务板系统** (task_create/update/get/list/claim):
  - 持久化 JSON 文件存储在 `.tasks/` 目录
  - 支持任务依赖关系 (blockedBy/blocks)
  - 任务认领机制
  - 状态：pending/in_progress/completed/deleted

### 3. 子智能体 (s04)
- **task**: 生成隔离的子智能体用于探索或工作
- 两种模式：Explore（只读）和 general-purpose（读写）
- 自动汇总返回结果

### 4. 技能系统 (s05)
- **load_skill**: 从 `skills/` 目录动态加载专业技能
- 支持 YAML frontmatter 元数据解析
- 注入到系统提示词中

### 5. 压缩机制 (s06)
- **microcompact**: 自动清理过长的工具结果（保留最近3项）
- **auto_compact**: 基于 token 阈值的对话历史自动压缩
- **compress**: 手动触发压缩
- 转录保存到 `.transcripts/` 目录

### 6. 后台任务 (s08)
- **background_run**: 在后台线程运行命令
- **check_background**: 检查后台任务状态
- 自动通知机制（drain 到对话中）

### 7. 团队与消息 (s09/s10/s11)
- **spawn_teammate**: 生成持久化队友（独立线程运行）
- **list_teammates**: 列出团队成员和状态
- **send_message**: 发送点对点消息
- **read_inbox**: 读取收件箱
- **broadcast**: 广播消息给所有队友
- **shutdown_request**: 优雅的关闭请求（带握手）
- **plan_approval**: 计划审批流程
- **idle**: 进入空闲状态（队友使用）
- **claim_task**: 从任务板认领任务（队友使用）

## 目录结构

```
.
├── s13_full_implement.py    # 主程序
├── skills/                    # 技能目录（SKILL.md 文件）
├── .tasks/                    # 任务板持久化存储
├── .team/                     # 团队配置和成员状态
│   └── inbox/                 # 消息收件箱
└── .transcripts/              # 对话转录存档
```

## 使用方法

### 启动

```bash
python s13_full_implement.py
```

### REPL 命令

```
s13_full >> [你的查询]

可用命令:
  /compact     - 手动触发对话压缩
  /tasks       - 显示任务板状态
  /team        - 显示团队成员状态
  /inbox       - 读取收件箱消息
  q/exit/空行  - 退出
```

### 环境变量

```bash
ANTHROPIC_BASE_URL    # API 基础 URL（可选）
MODEL_ID              # 使用的模型 ID
```

## 架构流程

### 单次 LLM 调用前的处理流程

1. **Microcompact** (s06): 清理过长的工具结果
2. **Auto-compact** (s06): 如果 token 超过阈值，压缩对话历史
3. **Drain background** (s08): 获取后台任务通知并注入对话
4. **Check inbox** (s09): 读取新消息并注入对话

### 工具分派 (s02)

根据 LLM 选择的工具调用对应处理器，执行后将结果返回给 LLM。

### 队友生命周期 (s09/s11)

```
spawn -> working -> idle -> auto-claim tasks -> working -> idle -> ...
         ^                                              |
         |______________________________________________|
```

队友空闲时会：
1. 检查新消息
2. 检查未认领任务
3. 自动认领并执行
4. 完成后回到空闲

## 安全特性

- 路径安全验证（所有文件操作限制在工作目录内）
- bash 命令危险操作过滤（rm -rf /, sudo, shutdown 等）
- 子智能体资源限制（30轮对话上限）
- 超时控制（120秒命令执行超时）

## 许可证

MIT License
