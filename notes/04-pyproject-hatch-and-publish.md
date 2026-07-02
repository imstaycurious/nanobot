# pyproject.toml 逐段解析 + Hatch 构建流程 + 发布部署全链路

## 一、pyproject.toml 逐段解析

### 1. 项目元数据

```toml
[project]
name = "nanobot-ai"          # PyPI 包名（pip install nanobot-ai）
version = "0.2.2"            # 版本号，手动维护
description = "A lightweight personal AI assistant framework"
readme = { file = "README.md", content-type = "text/markdown" }
requires-python = ">=3.11"   # 最低 Python 版本
license = {text = "MIT"}
authors = [
    {name = "Xubin Ren"},
    {name = "the nanobot contributors"}
]
```

**要点：**
- PyPI 上的包名是 `nanobot-ai`，不是 `nanobot`（因为 PyPI 上 `nanobot` 已被占用）
- 安装命令是 `pip install nanobot-ai`，但安装后 import 用 `import nanobot`
- 版本号在 `nanobot/__init__.py` 中有 fallback 读取逻辑（源码树直接 import 时无 dist-info）

### 2. 核心依赖（40 个）

```toml
dependencies = [
    "typer>=0.20.0,<1.0.0",       # CLI 框架
    "anthropic>=0.45.0,<1.0.0",   # Anthropic SDK
    "pydantic>=2.12.0,<3.0.0",    # 数据验证
    "pydantic-settings>=2.12.0",  # 配置管理
    "websockets>=16.0,<17.0",     # WebSocket 通信
    "httpx>=0.28.0,<1.0.0",       # HTTP 客户端
    "openai>=2.8.0",              # OpenAI SDK
    "mcp>=1.26.0,<2.0.0",        # MCP 协议
    "tiktoken>=0.12.0",           # Token 计数
    "boto3>=1.43.0",              # AWS SDK (Bedrock)
    # ... 还有 30 个（渠道 SDK、文档解析、搜索等）
]
```

**要点：**
- 所有核心依赖都是安装必需的，没有做"核心精简 + 可选插件"的拆分
- 这意味着即使用户只用 CLI 模式，也会安装 telegram-bot、slack-sdk、dingtalk 等
- 产物体积因此较大（pip install 后约 200-300MB）

### 3. 可选依赖

```toml
[project.optional-dependencies]
api     = ["aiohttp>=3.9.0"]           # HTTP API 服务器
azure   = ["azure-identity>=1.19.0"]   # Azure 认证
wecom   = ["wecom-aibot-sdk-python"]   # 企业微信
weixin  = ["qrcode[pil]", "pycryptodome"]  # 微信
msteams = ["PyJWT", "cryptography"]    # MS Teams
matrix  = ["matrix-nio[e2e]", ...]     # Matrix
discord = ["discord.py>=2.5.2"]        # Discord
whatsapp = ["neonize", "segno"]        # WhatsApp
langsmith = ["langsmith"]              # LangSmith 追踪
pdf     = ["pymupdf>=1.25.0"]          # PDF 解析增强
olostep = ["olostep"]                  # Olostep 网页抓取
dev     = ["pytest", "pytest-asyncio", "ruff", ...]  # 开发工具
```

**安装方式：**
```bash
pip install nanobot-ai[discord,matrix]   # 安装核心 + Discord + Matrix 依赖
pip install nanobot-ai[dev]              # 安装核心 + 开发工具
```

### 4. 入口点（最关键）

```toml
[project.scripts]
nanobot = "nanobot.cli.commands:app"
```

这一行做了两件事：
1. 告诉 pip/uv 在安装时生成名为 `nanobot` 的命令行入口
2. 入口指向 `nanobot.cli.commands` 模块的 `app` 对象（Typer 实例）

**安装后效果：**
```bash
$ nanobot          # 直接可用
$ nanobot gateway  # 启动 WebUI 网关
$ nanobot chat     # 交互式聊天
```

### 5. 插件入口点（预留）

```toml
# [project.entry-points."nanobot.tools"]
# my_plugin = "my_package.plugins:MyTool"
```

被注释掉了，但说明项目设计了插件机制。内置 tools/channels 通过 `pkgutil` 自动发现，第三方插件可以通过 entry-points 注册。

### 6. 构建系统

```toml
[build-system]
requires = ["hatchling"]          # 构建依赖
build-backend = "hatchling.build" # 使用 Hatchling 作为构建后端
```

Hatchling 是 Hatch 项目的构建引擎，替代传统的 setuptools。它：
- 读取 `pyproject.toml` 的配置
- 调用自定义 Build Hook（`hatch_build.py`）
- 生成标准 wheel (.whl) 和 sdist (.tar.gz)

### 7. 构建配置

```toml
[tool.hatch.build]
include = [
    "nanobot/**/*.py",           # Python 源码
    "nanobot/templates/**/*.md", # 模板文件
    "nanobot/skills/**/*.md",    # 技能定义
    "nanobot/skills/**/*.sh",    # 技能脚本
    "nanobot/web/dist/**/*",     # 前端构建产物
]
artifacts = [
    "nanobot/web/dist/**/*",     # 告诉 Hatch：这个目录虽然 git-ignored，但要打包进 wheel
]
```

**关键理解：**
- `include` 控制哪些文件进入 wheel
- `artifacts` 是 Hatch 特有概念：文件不在 git 跟踪中（被 .gitignore 排除），但构建时如果存在就打包进去
- `nanobot/web/dist/` 是 Vite 构建产物，git-ignored，但必须打包进 wheel（否则安装后没有 WebUI）

### 8. 代码质量工具配置

```toml
[tool.ruff]
line-length = 100
target-version = "py311"

[tool.ruff.lint]
select = ["E", "F", "I", "N", "W"]  # 规则集
ignore = ["E501"]                     # 忽略行长度限制

[tool.pytest.ini_options]
asyncio_mode = "auto"    # 所有 async test 自动标记
testpaths = ["tests"]

[tool.coverage.report]
fail_under = 75          # 覆盖率低于 75% 则失败
```

---

## 二、Hatch 构建流程详解

### 完整构建链路

```
python -m build
│
├── 1. Hatchling 初始化
│   ├── 读取 pyproject.toml
│   ├── 解析 [tool.hatch.build] 配置
│   └── 发现自定义 hook: hatch_build.py → WebUIBuildHook
│
├── 2. WebUIBuildHook.initialize() 执行
│   ├── 检查是否 editable install → 是则跳过
│   ├── 检查 NANOBOT_SKIP_WEBUI_BUILD → 是则跳过
│   ├── 检查 webui/package.json → 不存在则跳过（sdist 场景）
│   ├── 检查 nanobot/web/dist/index.html → 已存在则复用
│   ├── 选择构建工具: bun（优先）或 npm
│   ├── 执行 bun install（安装前端依赖）
│   └── 执行 bun run build（Vite 构建）
│       └── 产物输出到 nanobot/web/dist/
│           ├── index.html
│           ├── assets/*.js
│           └── assets/*.css
│
├── 3. Hatchling 收集文件
│   ├── 匹配 include 规则：
│   │   ├── nanobot/**/*.py        → 所有 Python 源码
│   │   ├── nanobot/templates/**/*.md → 模板
│   │   ├── nanobot/skills/**/*.md    → 技能
│   │   ├── nanobot/skills/**/*.sh    → 技能脚本
│   │   └── nanobot/web/dist/**/*    → 前端产物
│   └── artifacts 规则确保 git-ignored 的 dist/ 也被包含
│
├── 4. 生成 wheel (.whl)
│   ├── 将收集的文件按 Python 包布局写入
│   ├── 生成 METADATA（包名、版本、依赖列表）
│   ├── 生成 RECORD（文件清单 + SHA256）
│   ├── 生成 entry_points.txt：
│   │   [console_scripts]
│   │   nanobot = nanobot.cli.commands:app
│   └── 生成 WHEEL 元数据
│
└── 5. 生成 sdist (.tar.gz)
    └── 包含源码 + hatch_build.py + pyproject.toml 等
```

### wheel 内部结构

```
nanobot_ai-0.2.2-py3-none-any.whl
├── nanobot/
│   ├── __init__.py
│   ├── __main__.py
│   ├── nanobot.py              # SDK 入口
│   ├── agent/
│   │   ├── loop.py
│   │   ├── runner.py
│   │   ├── memory.py
│   │   ├── tools/              # 20+ 工具模块
│   │   └── ...
│   ├── bus/
│   ├── channels/               # 15+ 渠道模块
│   ├── providers/              # LLM Provider
│   ├── config/
│   ├── session/
│   ├── cli/
│   ├── web/                    # Gateway 服务
│   │   └── dist/               # ★ 前端构建产物
│   │       ├── index.html
│   │       └── assets/
│   ├── templates/              # Jinja2 模板
│   │   ├── HEARTBEAT.md
│   │   ├── SOUL.md
│   │   ├── agent/
│   │   └── memory/
│   ├── skills/                 # 技能定义
│   │   ├── cron/SKILL.md
│   │   ├── github/SKILL.md
│   │   └── ...
│   └── ...
├── nanobot_ai-0.2.2.dist-info/
│   ├── METADATA                # 包元数据 + 依赖列表
│   ├── WHEEL
│   ├── RECORD
│   ├── entry_points.txt        # nanobot = nanobot.cli.commands:app
│   └── top_level.txt           # nanobot
```

**注意 wheel 名：** `nanobot_ai`（下划线替换连字符），这是 Python 包命名规范。

---

## 三、从源码到可安装包的完整流程

### 场景 A：开发者本地构建

```bash
# 1. 克隆源码
git clone https://github.com/xxx/nanobot.git
cd nanobot

# 2. 构建前端（或让 hatch_build.py 自动构建）
cd webui && bun install && bun run build && cd ..

# 3. 构建 wheel + sdist
pip install build
python -m build
# 产物在 dist/ 目录：
#   nanobot_ai-0.2.2-py3-none-any.whl
#   nanobot_ai-0.2.2.tar.gz

# 4. 本地安装测试
pip install dist/nanobot_ai-0.2.2-py3-none-any.whl
nanobot --help
```

### 场景 B：发布到 PyPI

```bash
# 1. 构建
python -m build

# 2. 上传到 TestPyPI（可选，先测试）
pip install twine
twine upload --repository testpypi dist/*

# 3. 上传到正式 PyPI
twine upload dist/*
```

发布后，用户就可以：
```bash
pip install nanobot-ai
uv tool install nanobot-ai
```

### 场景 C：CI 自动化（当前项目）

项目有 GitHub Actions CI（`.github/workflows/ci.yml`），但 **只做测试，不做发布**：
- 运行 pytest + ruff
- 构建 WebUI
- 没有自动 publish 到 PyPI 的 workflow

发布目前是手动操作。

---

## 四、`pip install nanobot-ai` 需要什么条件？

### 发布者（让包出现在 PyPI 上）需要：

| 条件 | 说明 |
|------|------|
| PyPI 账号 | 注册 https://pypi.org ，建议开启 2FA |
| 包名所有权 | `nanobot-ai` 这个名字必须没被占用 |
| 构建产物 | `python -m build` 生成的 .whl + .tar.gz |
| API Token | PyPI 的 upload token（配置到 twine 或 CI） |
| 前端已构建 | `nanobot/web/dist/` 必须存在（hatch_build.py 自动处理） |

**发布命令：**
```bash
python -m build
twine upload dist/*
```

### 用户（安装 nanobot-ai）需要：

| 条件 | 说明 |
|------|------|
| Python >= 3.11 | `requires-python = ">=3.11"` |
| pip 或 uv | 包管理器 |
| 网络访问 PyPI | 能下载包和依赖 |
| C 编译器（部分场景） | lxml 等有 C 扩展，但 PyPI 提供预编译 wheel |

**安装命令：**
```bash
# 基础安装
pip install nanobot-ai

# 带可选依赖
pip install nanobot-ai[discord,matrix]

# uv tool 安装（隔离环境）
uv tool install nanobot-ai

# 从源码安装（开发模式）
git clone https://github.com/xxx/nanobot.git
cd nanobot
pip install -e .
```

### 从当前源码树直接安装

```bash
# 在项目根目录下：

# 方式 1：开发模式（editable），不构建前端
pip install -e .
# → hatch_build.py 检测到 editable，跳过 WebUI 构建
# → 需要手动 cd webui && bun run dev

# 方式 2：正式安装（先构建前端）
pip install .
# → hatch_build.py 触发，自动 bun install && bun run build
# → 前端产物打包进安装

# 方式 3：uv（更快）
uv pip install .
uv pip install -e .   # 开发模式
```

---

## 五、关键设计决策总结

### 1. 为什么用 Hatch 而不是 setuptools？

- Hatch 的 `artifacts` 机制完美解决了"git-ignored 但需要打包"的问题
- 自定义 Build Hook（`hatch_build.py`）让前端构建自动化
- 比 setuptools 的 `MANIFEST.in` + `setup.py` 更声明式

### 2. 为什么 WebUI 产物是 git-ignored 但打包进 wheel？

- **git-ignored**：避免把编译产物提交到版本控制（体积大、平台相关、每次变）
- **打包进 wheel**：用户安装后需要 WebUI，不能要求用户自己构建前端
- **Hatch artifacts** 机制解决了这个矛盾

### 3. 为什么核心依赖这么多？

- 项目定位是"一体化 AI 助手"，所有渠道 SDK 都是核心依赖
- 没有做"核心 + 插件"拆分，简化了安装体验（一个命令全装）
- 代价是安装体积大（~200-300MB）

### 4. 版本号管理

- 当前是手动维护 `pyproject.toml` 中的 `version = "0.2.2"`
- `nanobot/__init__.py` 有 fallback 逻辑：先从 `importlib.metadata` 读（已安装），读不到就从 `pyproject.toml` 读（源码树）
- 没有使用动态版本（如 hatch-vcs 基于 git tag 自动版本）
