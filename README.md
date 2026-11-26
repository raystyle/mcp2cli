# mcp2cli(Windows 支持版)

> 本仓库是 [mcp2cli/source-code](https://github.com/mcp2cli/source-code) 的 Windows 支持 fork(基于上游 `1c514ff`,v0.1.10)。
> 上游明确不支持 Windows(daemon 模式依赖 Unix socket);本 fork 补齐了 Windows 全路径编译与运行,并新增 `server.headers` 静态头注入。改动清单见文末「本 fork 改了什么」。

**把任意 MCP 服务器变成原生命令行工具。**

服务器的工具变成动词,资源变成名词,prompt 变成工作流。在终端里使用不需要任何 MCP 协议知识。

```bash
# 一个二进制。任意服务器。真正的 CLI 命令。
email ls
email send --to user@example.com --subject "Hi" --body "3点开会"
email get mail://inbox
staging deploy --version 2.1.0 --background
prod --json doctor | jq '.data.server'
```

---

## 为什么用 mcp2cli?

MCP(模型上下文协议)定义了服务器暴露工具、资源与 prompt 的强大标准。但与 MCP 服务器交互需要 JSON-RPC 管道、会话协商和协议层知识——这些都不属于命令行。

mcp2cli 弥合了这个鸿沟:

| 问题 | mcp2cli |
|---------|---------|
| MCP 需要 JSON-RPC 管道 | 自动发现能力 → 从 JSON Schema 生成类型化 `--flag` |
| 测试 MCP 服务器要写定制客户端 | 指哪打哪:`mcp2cli --url http://localhost:3001/mcp ls` |
| 每个服务器都要写自己的 CLI 包装 | 一个二进制 + 配置文件 = 无限服务器绑定 |
| AI agent 在 shell 里不好调 MCP 工具 | `--json` 输出 → 结构化信封,程序直接解析 |
| CI/CD 无法编排 MCP 操作 | 退出码、JSON 输出、管道、`--timeout`、`--background` |
| 服务器工具名是晦涩的协议名 | profile 覆盖层可重命名、分组、隐藏命令 |

---

## 快速开始

### 安装

```bash
# Windows(本 fork)
git clone https://github.com/raystyle/mcp2cli.git
cargo install --path mcp2cli

# 上游(Linux/macOS)
cargo install --git https://github.com/mcp2cli/source-code
```

### 方式一:临时连接 — 零配置

```bash
# HTTP 服务器 — 指向即用
mcp2cli --url http://127.0.0.1:3001/mcp ls
mcp2cli --url http://127.0.0.1:3001/mcp echo --message hello

# Stdio 服务器 — 直接运行
mcp2cli --stdio "npx @modelcontextprotocol/server-everything" ls
```

### 方式二:命名配置 + 别名

```bash
# 创建配置(注意:transport 参数是 kebab-case)
mcp2cli config init --name email --transport streamable-http \
  --endpoint https://mcp.example.com/email

# 创建别名(Windows 下自动加 .exe 并复制二进制)
mcp2cli link create --name email

# 认证(服务器需要 bearer token 时)
email auth login

# 像独立应用一样使用
email ls                                      # 发现能力
email send --to user@example.com \
  --subject "Hi" --body "3点开会"             # 调用工具
email search --query "from:boss"              # 搜索邮箱
email get mail://inbox                        # 读资源
email doctor                                  # 健康检查
```

### 方式三:Demo 模式 — 不需要服务器

```bash
mcp2cli config init --name demo --endpoint https://demo.invalid/mcp
mcp2cli use demo
mcp2cli ls
```

> **[完整上手指南 →](docs/getting-started.md)**

---

## 工作原理

```
mcp2cli 发现 → 构建命令清单 → 生成 CLI → 解析输入 → 执行 MCP 调用
```

1. **发现**服务器的工具、资源、资源模板与 prompt
2. **构建命令清单** — 每个能力变成类型化命令,flag 从 JSON Schema 推导
3. **生成 clap CLI 树** — 点号名字变成嵌套子命令,必填字段变成必填 flag
4. **解析输入** — 全类型校验
5. **执行** MCP 操作并渲染结果

### Schema 到 flag 的映射

| JSON Schema | CLI flag | 示例 |
|-------------|----------|------|
| `string` | `--name <TEXT>` | `--message hello` |
| `integer` | `--count <INT>` | `--steps 5` |
| `number` | `--rate <NUM>` | `--temperature 0.7` |
| `boolean` | `--flag` | `--include-image` |
| `enum` | `--kind <A\|B\|C>` | `--level error` |
| `array` | `--tags <VAL,...>` | `--labels bug,urgent` |
| 复杂类型 | `--config <JSON>` | `--config '{"a":1}'` |

### 命名空间分组

点号工具名自动变成嵌套子命令:

```bash
# 服务器工具:send、reply、draft.create
email send --to user@example.com --subject "Hi" --body "Hello"
email reply --thread-id 123 --body "Thanks"
email draft create --subject "新草稿"
```

---

## 功能一览

### 核心

- **[发现驱动的 CLI](docs/features/discovery-driven-cli.md)** — 服务器能力自动生成类型化 CLI 命令,`--flag` 从 JSON Schema 推导
- **[命名配置与别名](docs/features/named-configs-and-aliases.md)** — `mcp2cli use <name>`、symlink 别名、dispatch 路由
- **[Profile 覆盖层](docs/features/profile-overlays.md)** — 重命名、隐藏、分组命令;重命名 flag
- **[临时连接](docs/features/ad-hoc-connections.md)** — `--url` 与 `--stdio`,免配置即用
- **[模糊匹配](docs/features/fuzzy-matching.md)** — 打错命令时"你是不是想用?"
- **[Branded CLI](docs/features/branded-cli.md)** — 以服务器自己的名字发布 CLI(干净的 `--help`、登录,不带 mcp2cli 痕迹)

### 传输

- **[Streamable HTTP](docs/features/transports.md)** — HTTP 上的 JSON-RPC,SSE 流与会话协商
- **[Stdio](docs/features/transports.md)** — 拉起本地子进程服务器,stdin/stdout 通信
- **[Demo 模式](docs/features/transports.md)** — `demo.invalid` 文件后端,离线学习与测试

### 协议覆盖

完整双版本实现 — **[MCP 2026-07-28](https://modelcontextprotocol.io/specification/draft)**(无状态、每请求 `_meta`、`server/discover`、MRTR、tasks 扩展)与 **[MCP 2025-11-25](https://modelcontextprotocol.io/specification/2025-11-25)**(initialize 握手、会话)。默认用 `server/discover` 探测并自动回退 legacy 握手;用 `server.protocol_version` 钉住版本。详见 **[协议覆盖参考](docs/protocol-coverage.md)**。

### 运维

- **[认证](docs/features/authentication.md)** — `auth login/logout/status`,文件持久化 token
- **[事件系统](docs/features/event-system.md)** — 5 种 sink:stderr、HTTP webhook、Unix socket、SSE 服务器、命令执行
- **[输出格式](docs/features/output-formats.md)** — `--json`、`--output json|ndjson|human` — 每个命令都支持结构化输出
- **[请求超时](docs/features/request-timeouts.md)** — 全局默认 + 每命令 `--timeout`
- **[Daemon 模式](docs/features/daemon-mode.md)** — 连接保温(仅 Unix;Windows 暂不支持)
- **Doctor & Inspect** — 运行时健康诊断、全能力导出

---

## 多服务器工作流

每个别名路由到不同的 MCP 服务器,用起来都像独立应用。

```bash
# 配置多个服务器
mcp2cli config init --name dev --transport stdio --stdio-command ./dev-server
mcp2cli config init --name staging --transport streamable-http --endpoint https://staging.api/mcp
mcp2cli config init --name prod --transport streamable-http --endpoint https://prod.api/mcp

# 创建别名
mcp2cli link create --name dev
mcp2cli link create --name staging
mcp2cli link create --name prod

# 每个别名都是独立 CLI
dev ls                          # 本地开发服务器
staging deploy --version 1.2.3  # 预发 HTTP 服务器
prod doctor                     # 生产健康检查
```

**多实例场景**(本 fork 的主要用例):多台主机各挂多个 MCP 服务时,`.mcp.json` 的服务器名是全局单名无法区分实例;CLI 别名把"实例+服务"编码进命令名:

```bash
# 两台实验机,同一类服务,命令名天然区分
repl-lab execute_command --pipeline hostname --shell cmd   # 192.168.100.3 的 REPL
labre execute_command --pipeline hostname --shell cmd      # 192.168.100.20 的 REPL
```

---

## 面向脚本与 Agent 的 JSON 输出

每个命令都支持结构化 JSON 输出:

```bash
# JSON 信封
email --json ls | jq '.data.items[].id'

# 工具结果
email --json send --to user@example.com --subject "Hi" --body "Hello" | jq '.data.content[0].text'

# 健康检查
email --json doctor | jq '.data.server'

# 机器可读发现
email --json ls --tools | jq '[.data.items[] | {id, kind, summary}]'
```

统一的信封格式:

```json
{
  "app_id": "email",
  "command": "invoke",
  "summary": "called send",
  "lines": ["..."],
  "data": { "content": [...] }
}
```

---

## Profile 覆盖层

按服务器定制 CLI 面 — 无需改服务器:

```yaml
profile:
  display_name: "Email CLI"
  aliases:
    long-running-operation: lro     # 重命名命令
    echo: ping
  hide:
    - debug-tool                    # 从 help/ls 隐藏
  groups:
    mail:                           # 自定义分组
      - send
      - reply
      - draft-create
  flags:
    echo:
      message: msg                  # 重命名 flag
  resource_verb: fetch              # 用 "fetch" 替代 "get"
```

---

## 配置模型

每个命名配置是一个 YAML 文件。最小示例:

```yaml
schema_version: 1
server:
  transport: streamable_http
  endpoint: http://localhost:3001/mcp
```

本 fork 新增 `server.headers`(静态头注入):

```yaml
schema_version: 1
server:
  transport: streamable_http
  endpoint: http://192.168.100.3:8768/mcp
  headers:                # 每个请求注入的静态头(最后注入,可覆盖默认头)
    Host: localhost       # 例:钉扎 Host 以通过 DNS-rebinding 防护
    X-Custom-Auth: xxx    # 例:自定义鉴权头
```

> **[完整配置参考 →](docs/reference/config-reference.md)**

---

## 运行时命令(始终可用)

| 命令 | 说明 |
|---------|-------------|
| `ls [--tools\|--resources\|--prompts] [--filter]` | 发现能力 |
| `ping` | 服务器存活与延迟 |
| `doctor` | 运行时健康诊断 |
| `inspect` | 全能力导出 |
| `auth login\|logout\|status` | 认证管理 |
| `jobs list\|show\|wait\|cancel\|watch` | 后台任务管理 |
| `log <LEVEL>` | 设置服务端日志级别 |
| `subscribe <URI>` / `unsubscribe <URI>` | 资源变更通知 |
| `complete <REF> <NAME> <ARG>` | 服务器侧 tab 补全 |

## 宿主命令(管理配置,无需连服务器)

| 命令 | 说明 |
|---------|-------------|
| `mcp2cli config init [options]` | 创建命名配置 |
| `mcp2cli config list` | 列出全部配置 |
| `mcp2cli config show --name <NAME>` | 查看一个配置 |
| `mcp2cli use <NAME>` | 设置活跃配置 |
| `mcp2cli link create --name <NAME>` | 创建别名 |
| `mcp2cli daemon start\|stop\|status` | 管理连接 daemon(仅 Unix) |

> **[完整 CLI 参考 →](docs/reference/cli-reference.md)**

---

## 用例

| 用例 | 做法 |
|----------|-----|
| **测试 MCP 服务器** | `mcp2cli --url http://localhost:3001/mcp doctor` — 秒级健康检查 |
| **E2E 一致性测试** | Bash 测试套件验证 MCP 规范每一节 |
| **本地服务器开发** | `mcp2cli --stdio "./my-server" ls` — 边写边测,零客户端代码 |
| **AI agent 工具调用** | `--json` 输出 → Python/Node 解析 → agent 调任意 MCP 工具 |
| **CI/CD 管线** | JSON 输出、退出码、`--timeout`、`--background` — 管线原生 |
| **基础设施自动化** | 每服务一个别名 → `k8s deploy`、`db backup` |
| **Shell 脚本** | 管道给 jq、循环重试 |
| **多服务器编排** | 命名配置 + 别名 = 跨服务工作流 |
| **生产监控** | 事件 sink → webhook、SSE、Unix socket、自定义命令 |

---

## 测试

```bash
cargo test --lib                # 单元测试
cargo test --test integration   # 集成测试
cargo test                      # 全部测试
```

---

## 本 fork 改了什么(相对上游 1c514ff / v0.1.10)

上游[明示不支持 Windows](https://github.com/mcp2cli/source-code/blob/main/docs/release.md)(daemon 依赖 Unix socket、symlink 安装是 `#[cfg(unix)]`)。本 fork 补齐:

| # | 改动 | 说明 |
| --- | --- | --- |
| 1 | **Windows 编译通过** | `vsock_shim`/`daemon`/`sinks` 的 Unix 路径加 `cfg(unix)`;Windows 提供桩(daemon 调用报 "not supported",事件 sink 走 stderr/http/sse) |
| 2 | **`link create` Windows 化** | 别名自动带 `.exe`(PATHEXT 可执行);hardlink 跨盘报 EXDEV → 改为**复制二进制**(dispatch 按 `argv[0]` file_stem 路由,复制件行为与 symlink 一致;mcp2cli 升级后需重新 link) |
| 3 | **目录发现 Windows 化** | `RuntimeLayout` 无 `HOME` 时回退 `USERPROFILE` |
| 4 | **新增 `server.headers` 静态头注入** | streamable HTTP 每请求最后注入,可覆盖默认头——用于 DNS-rebinding 防护服务器的 `Host` 钉扎与自定义鉴权头(上游只有 OAuth/bearer) |
| 5 | **零 warning** | Unix-only 私有函数补 cfg 门,`cargo build` 干净 |
| 6 | **遥测默认关闭** | 上游 `telemetry.enabled` 默认 true(上报到 telemetry.mcp2cli.dev);本 fork 默认 false,不做任何上报(用户显式开启才报) |
| 7 | **Windows 测试修复** | `package.rs` 的 POSIX 绝对路径在 Windows `is_absolute()=false` 漏报(补 `/`、`\` 前缀判定);config 测试 branded data_dir 断言 `ends_with`→`contains`;`cargo test --lib` 200 全过 |`n| 8 | **新增 `legacy-sse` transport** | MCP 2024-11-05 HTTP+SSE 双通道客户端(官方 TS SDK 保留的 SSEClientTransport 形态):GET 流承载全部服务器→客户端消息,`endpoint` 事件通告 POST 地址,拒绝跨源;`--transport legacy-sse` 启用 |

Windows 实测(2026-11-26):4 个 MCP 服务器端到端(能力发现 + 工具调用 + `--json` 输出),含 Bearer+Host 双注入的 windbg-mcp;`cargo test --lib` 200 用例全过、构建零 warning。

### 已知限制

- **daemon 模式**仅在 Unix 可用(Windows 调用返回 "not supported";每次直连的开销可接受)。
- (已支持) **legacy SSE transport**(MCP 2024-11-05 形态,如 electron-debugger 的 `/sse` 端点):`--transport legacy-sse`,按官方 TypeScript SDK 的 SSEClientTransport 语义实现(GET 流承载全部服务器→客户端消息,endpoint 事件通告 POST 地址,拒绝跨源)。
- `link create` 用复制而非 symlink:升级 mcp2cli 后别名需要重建。

---

## 许可

Apache License 2.0 — 见 [LICENSE](LICENSE)。上游由 [TSOK](https://tsok.org) 运营;Windows 支持改动遵循同许可证。
