# opencode-free-gate
 
[opencode.ai/zen](https://opencode.ai) 免费模型的**自动代理反代网关**。

从公共代理池自动获取 S 级代理，请求失败自动切换，解除免费模型的额度/频率限制。  
同时兼容 **OpenAI** 和 **Anthropic** 两种 API 格式，任何客户端只需改 `base_url` 即可接入。

---

## 快速开始

```bash
# 安装 Bun（如未安装）
curl -fsSL https://bun.sh/install | bash

# 克隆
git clone https://github.com/Pandas886/opencode-free-gate.git
cd opencode-free-gate

# 启动（Bun 自动安装依赖）
bun run gate.ts

# 指定端口
PORT=8080 bun run gate.ts

# 启用 ZenProxy 备用通道（池子全挂时自动回退）
ZENPROXY_KEY=你的API_Key bun run gate.ts

# 调试：强制所有请求走 ZenProxy relay（跳过代理池）
FORCE_RELAY=1 ZENPROXY_KEY=你的API_Key bun run gate.ts
```

服务默认在 `http://localhost:13339` 启动。

在 [CC-switch](https://github.com/farion1231/cc-switch) 里配置给 Claude code 使用

 <img width="2339" height="1656" alt="shot-2026-06-02_16 20 43" src="https://github.com/user-attachments/assets/ac8464fa-52bf-4ecc-9419-2f8baa762c12" />




---

## 客户端配置

### OpenAI 格式

| 客户端 | 设置 |
|---|---|
| Python OpenAI SDK | `client = OpenAI(base_url="http://localhost:13339/openai/v1", api_key="public")` |
| curl | `curl http://localhost:13339/openai/v1/chat/completions -H 'Authorization: Bearer public' -d '...'` |
| 任何 OpenAI 兼容客户端 | `base_url = http://localhost:13339/openai/v1` |

### Anthropic 格式

| 客户端 | 设置 |
|---|---|
| Python Anthropic SDK | `client = Anthropic(base_url="http://localhost:13339/anthropic", api_key="public")` |
| curl | `curl http://localhost:13339/anthropic/v1/messages -H 'Authorization: Bearer public' -d '...'` |
| 任何 Anthropic 兼容客户端 | `base_url = http://localhost:13339/anthropic` |

### 查看可用模型

```bash
curl http://localhost:13339/openai/v1/models \
  -H 'Authorization: Bearer public'
```

---

## 架构

```
客户端 ──→ gate.ts (:13339) ──→ 代理池 ──→ opencode.ai/zen
                │
                ├── /openai/v1/*     → 转发到 /zen/v1/* (OpenAI 格式)
                ├── /anthropic/v1/*  → 转发到 /zen/v1/* (Anthropic 格式)
                └── 代理自动切换     → 失败自动重试，客户端无感
```

### 核心流程

1. **启动时**从 `proxy.amux.ai/api/proxies` 拉取 S 级免费代理，按延迟排序
2. **粘住一个代理**用，不按请求轮转；出错了才切换
3. **失败处理**：
   - 代理连不上 → 丢弃，换下一个重试
   - 上游 HTTP 5xx → 同样丢弃重试（可能是代理 IP 被限）
   - 最多 3 轮，全挂返回 502
4. **每 5 分钟**自动刷新代理池
5. **流式支持**：自动识别 `Accept: text/event-stream` 或 body 中的 `stream: true`，注入必要的参数，生成原生 SSE

---

## 环境变量

| 变量 | 默认 | 说明 |
|---|---|---|
| `PORT` | `13339` | 监听端口 |
| `ZENPROXY_KEY` | 空 | 启用 ZenProxy 备用通道（[申请 Key](https://zenproxy.top)） |
| `ZENPROXY_RELAY` | `https://zenproxy.top/api/relay` | 自定义 relay 端点 |
| `FORCE_RELAY` | `0` | 设为 `1` 跳过代理池强制走 ZenProxy（调试用） |

### 关于 ZenProxy 备用通道

主路径（免费代理池）失败时，自动回退到 ZenProxy 的 `/api/relay` 转发。回退触发条件：

1. 启动时 `proxy.amux.ai` 拉不到代理
2. 池子里的代理全部进入黑名单
3. 连续 3 轮重试都失败

⚠️ 备用通道会**剥离客户端的 `Authorization` 头**再转发，避免 `Bearer public` 这类占位 token 被 opencode 拒绝。

---

## 依赖

- [hpagent](https://github.com/delvedor/hpagent) — HTTP CONNECT 代理隧道
- [socks-proxy-agent](https://github.com/TooTallNate/proxy-agents) — SOCKS5 代理

Bun 会自动安装，无需 `package.json`。
