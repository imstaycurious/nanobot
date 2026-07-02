# nanobot 打包与安装机制

## 核心问题：`uv tool install` 之后发生了什么？

### 1. 入口点定义

在 `pyproject.toml` 中：

```toml
[project.scripts]
nanobot = "nanobot.cli.commands:app"
```

这是 Python 的 **console_scripts** 机制。格式为：

```
命令名 = 模块路径:对象
```

这里 `app` 是一个 `typer.Typer` 实例（定义在 `nanobot/cli/commands.py:175`），Typer 会自动将其转换为 Click 命令行应用。

### 2. 安装时发生了什么

当你执行 `uv tool install nanobot-ai` 时：

1. **uv 创建一个隔离的虚拟环境**（通常在 `~/.local/share/uv/tools/nanobot-ai/` 下）
2. **将 nanobot 及其所有依赖安装到这个虚拟环境中**
3. **生成一个 wrapper 脚本**（通常在 `~/.local/bin/nanobot`），内容大致是：

```python
#!/usr/bin/python3
# -*- coding: utf-8 -*-
import re
import sys
from nanobot.cli.commands import app
if __name__ == '__main__':
    sys.argv[0] = re.sub(r'(-script\.pyw|\.exe)?$', '', sys.argv[0])
    sys.exit(app())
```

Windows 上则生成 `.exe` wrapper。

所以你执行的 `nanobot` 命令，本质上是 **调用隔离 venv 中的 Python 来运行 `nanobot.cli.commands:app`**。

### 3. 这个"可执行文件"能迁移吗？

**不能直接迁移。** 原因：

| 因素 | 说明 |
|------|------|
| **wrapper 脚本硬编码了 Python 路径** | 指向 `~/.local/share/uv/tools/nanobot-ai/` 下的那个 venv 的 Python |
| **依赖在 venv 中** | 所有 Python 依赖（anthropic、pydantic、httpx 等 60+ 个包）都在那个 venv 的 `site-packages` 里 |
| **平台相关的编译文件** | 部分 C 扩展包（如 lxml、lxml-html-clean）包含 `.pyd`/`.so` 二进制，与平台架构绑定 |
| **Python 版本绑定** | venv 是用特定版本的 Python 创建的 |

简单说：`uv tool install` 生成的不是真正的独立可执行文件，而是一个 **指向隔离 Python 环境的快捷方式**。

### 4. 如果想要可迁移的部署方式

| 方式 | 可迁移性 | 说明 |
|------|----------|------|
| `uv tool install` | ❌ 不可迁移 | 依赖本地 venv |
| `pip install` | ❌ 不可迁移 | 同理，依赖本地 site-packages |
| **Docker** | ✅ 可迁移 | `docker-compose.yml` 已提供，跨平台一致 |
| **PyInstaller / Nuitka** | ✅ 可迁移 | 打包成独立二进制，但项目未配置 |
| **pipx** | ❌ 不可迁移 | 与 uv tool 原理相同，也是隔离 venv |

### 5. 构建流程详解

nanobot 使用 **Hatch** 作为构建后端：

```toml
[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"
```

构建时有一个自定义 hook（`hatch_build.py`）—— **WebUIBuildHook**：

```
python -m build
    ↓
hatchling 触发 WebUIBuildHook.initialize()
    ↓
检测 webui/package.json 是否存在
    ↓
调用 bun install && bun run build
    ↓
前端产物输出到 nanobot/web/dist/
    ↓
hatchling 将 nanobot/web/dist/**/* 打包进 wheel
```

关键细节：
- **editable install 跳过 WebUI 构建**（开发时用 `bun run dev`）
- 可通过 `NANOBOT_SKIP_WEBUI_BUILD=1` 跳过
- 可通过 `NANOBOT_FORCE_WEBUI_BUILD=1` 强制重建
- bun 不可用时回退到 npm

### 6. Wheel 包内容

最终发布的 wheel 包含：

```
nanobot/
├── **/*.py              # Python 源码
├── templates/**/*.md    # 模板文件（HEARTBEAT.md 等）
├── skills/**/*.md       # 技能定义
├── skills/**/*.sh       # 技能脚本
└── web/dist/**/*        # 前端构建产物（git-ignored，但打包进 wheel）
```

其中 `web/dist/` 是 Vite 构建产物，虽然 git-ignored，但通过 Hatch 的 `artifacts` 配置确保打包进 wheel：

```toml
[tool.hatch.build]
artifacts = ["nanobot/web/dist/**/*"]
```

## 总结

- `uv tool install` 本质是 **创建隔离 venv + 生成 wrapper 脚本**，不是真正的独立可执行文件
- 生成的 `nanobot` 命令依赖本地 Python 环境和 venv 中的依赖，**不可直接迁移**
- 如需跨机器部署，推荐使用 Docker（项目已提供 Dockerfile 和 docker-compose.yml）
- 构建流程通过 Hatch + 自定义 hook 自动将前端产物打包进 Python wheel
