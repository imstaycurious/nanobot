# 学习笔记索引

## 已完成

- [01-overview.md](01-overview.md) — 项目总体架构概览
- [02-packaging-and-install.md](02-packaging-and-install.md) — 打包与安装机制（uv tool install 原理、可迁移性分析）
- [03-python-executable-packaging.md](03-python-executable-packaging.md) — Python 独立可执行文件方案（PyApp、PyInstaller 等）
- [04-pyproject-hatch-and-publish.md](04-pyproject-hatch-and-publish.md) — pyproject.toml 逐段解析 + Hatch 构建流程 + 发布部署全链路

## 关键知识点总结

### 打包与构建
- pyproject.toml 定义包元数据、依赖、入口点（`nanobot = nanobot.cli.commands:app`）、构建配置
- Hatchling 是构建后端，`python -m build` 是构建前端，通过 `[build-system]` 配对
- hatch_build.py 是构建前钩子，自动构建 WebUI 前端，解决"dist/ git-ignored 但必须进 wheel"的矛盾
- `pip install -e .` editable 模式：.egg-link 指向项目源码，改代码立刻生效
- `uv tool install` 创建隔离 venv + wrapper 脚本，不可迁移；Docker 可迁移
- PyApp 是最优雅的可迁移方案（~10MB Rust 二进制，首次运行自动下载 Python+依赖）

### Python 工具链
- `python -m xxx` 用当前 Python 执行模块，避免 PATH 错配
- `uv tool install`（装 App，隔离环境）vs `uv pip install`（装库，当前环境）
- `uv sync` 是 uv 生态下更自然的方式，默认 editable
- shutil.which() 在 PATH 中查找可执行文件
