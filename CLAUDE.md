# Hermes Agent - Claude Code 项目配置

## 项目概述

**Hermes Agent** 是由 Nous Research 构建的自改进 AI Agent，支持多平台（CLI、Telegram、Discord、Slack 等），具备内置学习循环、技能创建和持久化记忆。

## 基础信息

- **版本**: 0.8.0
- **Python**: 3.11+
- **Node.js**: 18.0.0
- **包管理**: uv (推荐), pip
- **测试**: pytest
- **lint**: ruff check

## 开发环境设置

```bash
# 克隆后初始化
git clone https://github.com/NousResearch/hermes-agent.git
cd hermes-agent
uv venv venv --python 3.11
source venv/bin/activate  # 始终需要激活虚拟环境
uv pip install -e ".[all,dev]"

# RL 训练（可选）
git submodule update --init tinker-atropos
uv pip install -e "./tinker-atropos"
```

## 常用命令

```bash
# 测试
python -m pytest tests/ -q                    # 全套测试 (~3000 tests, ~3 min)
python -m pytest tests/ -q -m integration    # 包含集成测试
python -m pytest tests/test_model_tools.py -q   # 工具集解析
python -m pytest tests/test_cli_init.py -q       # CLI 配置加载
python -m pytest tests/gateway/ -q               # 网关测试
python -m pytest tests/tools/ -q                 # 工具测试

# Lint
ruff check .

# 运行
hermes              # 交互式 CLI
hermes setup        # 设置向导
hermes model        # 选择模型
hermes tools        # 配置工具
hermes gateway      # 消息网关
hermes doctor       # 诊断问题
```

## 项目结构

```
hermes-agent/
├── run_agent.py           # AIAgent 类 — 核心对话循环
├── model_tools.py         # 工具编排, _discover_tools(), handle_function_call()
├── toolsets.py            # 工具集定义, _HERMES_CORE_TOOLS 列表
├── cli.py                 # HermesCLI 类 — 交互式 CLI 编排器
├── hermes_state.py        # SessionDB — SQLite 会话存储 (FTS5 搜索)
├── hermes_constants.py    # 常量, get_hermes_home(), HERMES_HOME
├── agent/                 # Agent 内部模块
│   ├── prompt_builder.py      # 系统提示组装
│   ├── context_compressor.py  # 自动上下文压缩
│   ├── prompt_caching.py      # Anthropic 提示缓存
│   ├── auxiliary_client.py    # 辅助 LLM 客户端 (视觉、摘要)
│   ├── model_metadata.py      # 模型上下文长度、token 估算
│   ├── memory_manager.py      # 记忆上下文管理
│   └── display.py             # KawaiiSpinner, 工具预览格式化
├── hermes_cli/            # CLI 子命令和设置
│   ├── main.py            # 入口点 — 所有 `hermes` 子命令
│   ├── config.py          # DEFAULT_CONFIG, OPTIONAL_ENV_VARS, 迁移
│   ├── commands.py        # 斜杠命令定义 + SlashCommandCompleter
│   ├── setup.py           # 交互式设置向导
│   ├── skin_engine.py     # 皮肤/主题引擎
│   ├── skills_config.py   # `hermes skills` — 按平台启用/禁用技能
│   └── tools_config.py    # `hermes tools` — 按平台启用/禁用工具
├── tools/                 # 工具实现 (每工具一个文件)
│   ├── registry.py        # 中心化工具注册表 (schema, handler, dispatch)
│   ├── approval.py        # 危险命令检测
│   ├── terminal_tool.py   # 终端编排
│   ├── file_tools.py      # 文件读/写/搜索/修补
│   ├── web_tools.py       # Web 搜索/提取
│   ├── browser_tool.py    # Browserbase 浏览器自动化
│   └── environments/      # 终端后端 (local, docker, ssh, modal, daytona, singularity)
├── gateway/               # 消息平台网关
│   ├── run.py             # 主循环, 斜杠命令, 消息分发
│   ├── session.py         # SessionStore — 对话持久化
│   └── platforms/         # 适配器: telegram, discord, slack, whatsapp, homeassistant, signal
├── acp_adapter/           # ACP 服务器 (VS Code / Zed / JetBrains 集成)
├── cron/                  # 调度器 (jobs.py, scheduler.py)
├── environments/          # RL 训练环境 (Atropos)
├── skills/                # 内置技能 (28 类别)
├── optional-skills/       # 可选技能包
├── tests/                 # Pytest 测试套件
└── batch_runner.py        # 并行批处理
```

## 重要约定

### 路径规范 — 必须遵守

**禁止硬编码 `~/.hermes`**：
```python
# 正确
from hermes_constants import get_hermes_home
config_path = get_hermes_home() / "config.yaml"

# 错误 — 破坏 profiles 支持
config_path = Path.home() / ".hermes" / "config.yaml"
```

**用户显示路径**：
```python
# 正确
from hermes_constants import display_hermes_home
print(f"Config saved to {display_hermes_home()}/config.yaml")
```

### 添加新工具（3 步）

1. 创建 `tools/your_tool.py`:
```python
import json, os
from tools.registry import registry

def check_requirements() -> bool:
    return bool(os.getenv("EXAMPLE_API_KEY"))

def your_tool(param: str, task_id: str = None) -> str:
    return json.dumps({"success": True, "data": "..."})

registry.register(
    name="your_tool",
    toolset="example",
    schema={"name": "your_tool", "description": "...", "parameters": {...}},
    handler=lambda args, **kw: your_tool(param=args.get("param", ""), task_id=kw.get("task_id")),
    check_fn=check_requirements,
    requires_env=["EXAMPLE_API_KEY"],
)
```

2. 在 `model_tools.py` 的 `_discover_tools()` 中添加 import
3. 在 `toolsets.py` 中添加到 `_HERMES_CORE_TOOLS` 或新工具集

### 添加斜杠命令

1. 在 `hermes_cli/commands.py` 的 `COMMAND_REGISTRY` 添加 `CommandDef`
2. 在 `cli.py` 的 `HermesCLI.process_command()` 添加处理逻辑
3. 如需在网关可用，在 `gateway/run.py` 添加处理

### 配置文件

- **config.yaml**: 添加到 `hermes_cli/config.py` 的 `DEFAULT_CONFIG`，并更新 `_config_version`
- **.env 变量**: 添加到 `hermes_cli/config.py` 的 `OPTIONAL_ENV_VARS`

## 架构要点

### 工具注册表依赖链
```
tools/registry.py (无依赖 — 所有工具文件都导入它)
       ↑
tools/*.py (每个工具文件在导入时调用 registry.register())
       ↑
model_tools.py (导入 tools/registry + 触发工具发现)
       ↑
run_agent.py, cli.py, batch_runner.py, environments/
```

### 对话循环
```python
while api_call_count < self.max_iterations:
    response = client.chat.completions.create(model=model, messages=messages, tools=tool_schemas)
    if response.tool_calls:
        for tool_call in response.tool_calls:
            result = handle_function_call(tool_call.name, tool_call.args, task_id)
            messages.append(tool_result_message(result))
    else:
        return response.content
```

### 消息格式
OpenAI 格式: `{"role": "system/user/assistant/tool", ...}`，reasoning content 存在 `assistant_msg["reasoning"]`

## Profiles 多实例支持

Hermes 支持 profiles，每个 profile 有独立的 `HERMES_HOME`：
- `_apply_profile_override()` 在 `hermes_cli/main.py` 设置 `HERMES_HOME`
- 所有 `get_hermes_home()` 调用自动作用域到活跃 profile
- profiles 根目录: `~/.hermes/profiles/`

## 已知陷阱

1. **不要硬编码 `~/.hermes` 路径** — 使用 `get_hermes_home()`，这破坏了 profiles
2. **不要在交互菜单使用 `simple_term_menu`** — tmux/iTerm2 渲染 bug，使用 curses
3. **不要在 spinner/display 代码使用 `\033[K`** — 使用空格填充
4. **`_last_resolved_tool_names` 是进程全局的** — `delegate_tool.py` 在子 agent 执行时会保存/恢复
5. **工具 schema 描述不要跨工具引用** — 可能导致模型幻觉调用不存在的工具
6. **测试不能写入 `~/.hermes/`** — 使用 `tests/conftest.py` 的 `_isolate_hermes_home` fixture

## 用户配置位置

- `~/.hermes/config.yaml` — 设置
- `~/.hermes/.env` — API 密钥
- `~/.hermes/state.db` — SQLite 数据库
- `~/.hermes/skills/` — 用户创建的技能
- `~/.hermes/skins/*.yaml` — 自定义皮肤

## 依赖组 (pip install)

| Extra | 用途 |
|-------|------|
| `modal` | Modal 无服务器 |
| `daytona` | Daytona 无服务器 |
| `messaging` | Telegram, Discord, Slack 等 |
| `voice` | 语音识别/合成 |
| `honcho` | Honcho AI 记忆 |
| `mcp` | MCP 服务器 |
| `cron` | Cron 调度 |
| `cli` | CLI 菜单 |
| `rl` | RL 训练 |
| `dev` | 开发工具 |
| `all` | 所有可选依赖 |
