# Python 独立可执行文件打包方案

## 需求

将 nanobot 打包成一个可迁移的独立可执行文件，无需目标机器安装 Python 或任何依赖。

## 主流方案对比

| 方案 | 原理 | 产物体积 | 启动速度 | 跨平台 | 适合 nanobot？ |
|------|------|----------|----------|--------|---------------|
| **PyInstaller** | 冻结 Python + 依赖为自解压包 | 大（200-500MB） | 慢（需解压） | 需分平台构建 | ⚠️ 可行但痛苦 |
| **Nuitka** | Python → C 编译，链接 CPython | 中（150-300MB） | 快 | 需分平台构建 | ⚠️ 可行但编译慢 |
| **cx_Freeze** | 打包 Python 字节码 + 运行时 | 大 | 中 | 需分平台构建 | ⚠️ 类似 PyInstaller |
| **PyApp** | 自解压内嵌 Python + uv | 小（~10MB + 依赖） | 首次慢，后续快 | ✅ 单二进制 | ✅ **最优雅** |
| **Shiv** | 打包为自包含 zipapp | 中 | 中 | 需分平台构建 | ⚠️ 不如 PyApp |
| **pex** | 类似 Shiv，Pinterest 出品 | 中 | 中 | 需分平台构建 | ⚠️ 不如 PyApp |

## 推荐：PyApp（最优雅的方案）

### 什么是 PyApp？

PyApp 的思路完全不同——它不是"冻结"你的应用，而是：

1. 生成一个 **~10MB 的小型 Rust 二进制**
2. 首次运行时，自动下载 Python 运行时 + 用 uv 安装你的包及依赖
3. 后续运行直接使用缓存，秒启动

```
用户执行 nanobot
    ↓
PyApp 二进制启动
    ↓
首次运行？→ 下载 Python + uv install nanobot-ai → 缓存到 ~/.pyapp/
    ↓
执行 nanobot.cli.commands:app
```

### 为什么说它"可迁移"？

- 那个 ~10MB 的二进制是 **真正的独立可执行文件**，不依赖本地 Python
- 首次运行时自动拉取运行时和依赖，**目标机器不需要预装任何东西**
- 同一个二进制在不同机器上都能工作（同架构同 OS）

### 如何为 nanobot 构建

```bash
# 安装 PyApp 构建工具
pip install pyapp

# 构建（指定包名和入口）
pyapp-build nanobot-ai --entry-point nanobot.cli.commands:app

# 产物是一个独立的 nanobot 可执行文件
```

或者用环境变量控制：

```bash
export PYAPP_PROJECT_NAME=nanobot-ai
export PYAPP_PROJECT_VERSION=0.2.2
export PYAPP_EXEC_SPEC=nanobot.cli.commands:app
pyapp-build
```

### 优缺点

**优点：**
- 产物极小（~10MB vs PyInstaller 的 200-500MB）
- 自动处理依赖，无需手动排除/包含
- 利用 uv 的快速安装，首次运行也很快
- 支持版本锁定、pip 配置传递
- Rust 编写，无 Python 构建依赖

**缺点：**
- 首次运行需要网络（下载 Python + 包）
- 目标机器需要能访问 PyPI（或配置私有源）
- 不是真正的"离线单文件"——运行时和依赖是按需下载的

## 备选：PyInstaller（离线单文件方案）

如果需要 **完全离线、不依赖网络** 的可执行文件：

```bash
pip install pyinstaller

# 生成单文件可执行
pyinstaller --onefile --name nanobot \
    --hidden-import=nanobot.cli.commands \
    -c "from nanobot.cli.commands import app; app()" \
    nanobot
```

### nanobot 用 PyInstaller 的挑战

1. **依赖太多**（40 个核心 + 可选），PyInstaller 容易漏隐式导入
2. **动态加载**（`pkgutil` 扫描发现 tools/channels），PyInstaller 静态分析抓不到
3. **C 扩展**（lxml、dulwich 等）需要特殊处理
4. **WebUI 静态文件**（`nanobot/web/dist/`）需要手动 `--add-data`
5. **产物巨大**（200-500MB），因为包含了整个 CPython + 所有依赖

### 如果坚持用 PyInstaller，需要额外配置

```python
# nanobot.spec (PyInstaller spec 文件)
a = Analysis(
    ['nanobot/__main__.py'],
    hiddenimports=[
        # pkgutil 动态发现的模块必须手动列出
        'nanobot.agent.tools.filesystem',
        'nanobot.agent.tools.shell',
        'nanobot.agent.tools.web',
        'nanobot.agent.tools.mcp',
        'nanobot.agent.tools.cron',
        'nanobot.agent.tools.spawn',
        'nanobot.agent.tools.long_task',
        'nanobot.agent.tools.image_generation',
        'nanobot.agent.tools.search',
        'nanobot.agent.tools.self',
        'nanobot.agent.tools.message',
        'nanobot.agent.tools.context',
        'nanobot.channels.telegram',
        'nanobot.channels.discord',
        'nanobot.channels.slack',
        'nanobot.channels.feishu',
        'nanobot.channels.websocket',
        # ... 更多
    ],
    datas=[
        ('nanobot/web/dist', 'nanobot/web/dist'),
        ('nanobot/templates', 'nanobot/templates'),
        ('nanobot/skills', 'nanobot/skills'),
    ],
)
```

## 方案选择建议

```
需要离线运行？
├── 是 → PyInstaller（大但独立）或 Nuitka（编译更快但构建慢）
└── 否 → PyApp（小而优雅，推荐）
    └── 目标机器能联网？→ ✅ 完美方案
    └── 内网环境？→ 配置 PYAPP_PYTHON_INDEX 指向私有 PyPI
```

## 关于"可迁移"的澄清

严格来说，**没有任何 Python 打包方案能生成一个跨 OS/架构 的通用二进制**：

- Windows x64 的可执行文件不能在 Linux 上跑
- macOS ARM 的不能在 macOS x64 上跑
- 需要为每个目标平台分别构建

但 **同 OS 同架构下**，PyApp 和 PyInstaller 的产物都是可以直接拷贝到另一台机器运行的。
