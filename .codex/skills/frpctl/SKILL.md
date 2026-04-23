---
name: frpctl
description: 通过 frpctl CLI 管理 frp 隧道,用于服务暴露和远程访问场景(本地开发服务共享、远程调试、临时公网访问)。当用户说"开个穿透/隧道"、"暴露端口 X"、"把本地 N 端口放出去"、"关掉那个隧道"、"看下现在有哪些穿透"等,激活此 skill。前提:用户已经配置好 frps 服务器信息,不要让用户手写 frp 配置文件。
---

# frpctl — frp 隧道管理

## 工具定位

`frpctl` 是已装好、配置好 frps 服务器信息的本地 CLI。调用它来开/关/查 frp 隧道，**不要**：

- 手写 frpc.toml / frps.toml
- 让用户提供 token、服务器 IP（已在 `~/.config/frpctl/config.toml`）
- 打印或询问 token
- 直接起 frpc 进程（用 `frpctl up/down`）

## 常用命令速查

### 一键开隧道（最常用）

```bash
frpctl quick -l 4444
# 输出示例：
#   tunnel: tango-742
#     type: tcp
#    local: localhost:4444
#   public: 203.0.113.10:20007
```

带 TTL 自动过期：
```bash
frpctl quick -l 4444 --ttl 4h
```

按类型创建（没有域名只能用 tcp/udp）：
```bash
frpctl quick -l 4444 -t tcp             # TCP，远端端口自动分配
frpctl quick -l 5353 -t udp             # UDP
frpctl quick -l 8080 -t http --custom-domains app.example.com   # 需要域名
frpctl quick -l 8443 -t https --subdomain myapp                 # 需要域名
frpctl quick -l 7001 -t stcp --secret-key 'CHANGE_ME'          # 点对点加密
```

类型参数规则：

- `tcp` / `udp`：可选 `--remote-port`，不支持 `--custom-domains`、`--subdomain`、`--secret-key`
- `http` / `https`：必须设置 `--custom-domains` 或 `--subdomain`，frps 需配置 `vhostHTTPPort` 和域名
- `stcp`：必须设置 `--secret-key`，访问端也需运行 frpc visitor

### 显式定义隧道（长期保留）

```bash
frpctl tunnel add dbshell  -l 5432 -t tcp
frpctl tunnel add dns      -l 5353 -t udp
frpctl tunnel add web      -l 8080 -t http --custom-domains app.example.com
frpctl tunnel add internal -l 7001 -t stcp --secret-key 'CHANGE_ME'
frpctl up                  # 启动守护进程，加载所有隧道
```

### 状态查询

```bash
frpctl tunnel list           # 所有隧道 + 公网地址
frpctl tunnel url <name>     # 只打印 host:port，方便脚本/payload 拼接
frpctl status                # 守护进程状态
frpctl logs -f               # 实时看 frpc 日志
```

### 关闭

```bash
frpctl tunnel rm <name>      # 删一个（运行中会自动 reload）
frpctl down                  # 停掉整个守护进程
```

### 服务器管理（一般不需要动）

```bash
frpctl server list
frpctl server use <name>     # 切换默认服务器
```

只在用户明确说"加一台新 VPS"时才用 `server add`。

## 触发模式 → 命令映射

| 用户说 | 执行 |
|---|---|
| "开个穿透到 4444" / "暴露 4444" / "给我一个公网端口" | `frpctl quick -l 4444` |
| "开个 UDP 穿透到 5353" | `frpctl quick -l 5353 -t udp` |
| "把本地 8080 通过域名放出去" | `frpctl quick -l 8080 -t http --custom-domains <domain>` |
| "给我 https 子域名入口" | `frpctl quick -l 8443 -t https --subdomain <name>` |
| "开 stcp 内网通道" | `frpctl quick -l <port> -t stcp --secret-key <key>` |
| "现在有哪些隧道" / "看下状态" | `frpctl tunnel list` |
| "告诉我 x 的公网地址" | `frpctl tunnel url x` |
| "关掉 x" / "删掉 x" | `frpctl tunnel rm x` |
| "全部关掉" / "收工" | `frpctl down` |
| "重启一下" | `frpctl down && frpctl up` |
| "看 frpc 日志" / "有没有连上" | `frpctl logs` |

## 给用户公网地址时的完整交互模板

当用户说类似"给我一个公网地址访问本地服务"：

1. 先跑 quick（没有域名默认用 tcp）：
   ```bash
   frpctl quick -l <port> --ttl 2h
   ```
2. 从输出里提取 `public:` 字段
3. 直接给用户可访问地址：
   ```
   本地服务：localhost:<local_port>
   公网入口：14.1.99.251:<remote_port>
   ```
4. 提醒 TTL 到期自动回收，或用 `frpctl tunnel rm <name>` 手动关

## 不要做的事

- **不要打印或回显 token**：`auth.token` 是敏感信息
- **不要手写 frpc.toml**：运行时自动生成在 `~/.cache/frpctl/frpc.toml`
- **不要用 sudo 跑 frpctl**：以当前用户身份运行即可
- **不要推荐低位端口（<1024）**：端口池默认 20000-20100
- **没有域名时不要创建 http/https 隧道**：会连接失败，用 tcp 代替

## 故障排查

| 症状 | 处理 |
|---|---|
| `connect to server error: EOF` | frps 关闭了 tcpMux，需在 `~/.config/frpctl/config.toml` 对应服务器下加 `tcp_mux = false` |
| `frpc binary not found` | 让用户装 frpc 或设 `FRPCTL_FRPC=/path/to/frpc` |
| `port pool exhausted` | `frpctl tunnel list` 看僵尸隧道，批量 `rm` |
| `tunnels span multiple servers` | 用 `frpctl up <name>` 指定单个隧道启动 |
| 公网连不上 | `frpctl logs` 看日志；检查 VPS 防火墙是否放行端口池 |
| http 隧道无法访问 | frps 需配置 `vhostHTTPPort` 和 `subdomainHost`，没有域名请改用 tcp |
