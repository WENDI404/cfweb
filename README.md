# cf-webproxy

基于 **Cloudflare Workers + Durable Objects** 的 Telegram WEB Proxy 独立实现。在 Worker 内完成 MTProxy 混淆流转换并直连 Telegram 数据中心，**不需要 VPS、后端 MTProxy 或管理面板**。

```text
Telegram Android → HTTPS / WebSocket → Worker + Durable Object → Telegram DC:443
```

支持 WebSocket 多路复用、连接内保序、独立队列、流控与超时清理。仅连接代码内预设的 Telegram DC，不接受任意 TCP 目标。

## 部署到 Cloudflare

> 本项目是 **Worker，不是 Pages**。仓库已包含入口、Durable Object 绑定及迁移配置，不需要填写 `dist` 输出目录。

### 方法一：一键部署（推荐）

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/coldboy404/cf-webproxy)

1. 点击按钮，登录 Cloudflare，按提示连接 GitHub 并创建仓库。
2. 保留默认 Worker 名称，或自行修改。
3. 按提示填写 `PROXY_SECRET`，确认部署。所需 Durable Object 会自动创建。
4. 部署完成后，记录 Worker 的 `*.workers.dev` 地址。

**如果部署页面未要求填写 Secret**，进入：

```text
Workers & Pages → 选择 Worker → Settings → Variables and Secrets → Add
```

添加名称为 `PROXY_SECRET` 的 **Secret（机密），不要使用普通明文变量**。值必须是 **32 位小写十六进制字符串**，也支持 `dd` 加 32 位小写十六进制字符串；不支持 `ee` / FakeTLS。请使用自己的密钥，不要照抄示例。

保存并按控制台提示部署，使配置生效。

### 方法二：导入 GitHub 仓库

1. Fork 本仓库。
2. 在 Cloudflare 的 **Workers & Pages → Create application → Import a repository** 中选择 Fork 后的仓库。
3. 使用以下配置：

| 配置 | 值 |
|---|---|
| Production branch | `main` |
| Build command | 留空或 `npm test` |
| Deploy command | `npx wrangler deploy` |
| Root directory | `/` |

4. 保存并部署，再按上一节添加 `PROXY_SECRET` 并使其生效。

### 方法三：命令行部署

需要 Node.js **22.15+（推荐 24 LTS）**、npm 和 Cloudflare 账户。

```bash
git clone https://github.com/coldboy404/cf-webproxy.git
cd cf-webproxy
npm ci
npx wrangler login
npx wrangler secret put PROXY_SECRET
npm test
npx wrangler deploy
```

运行 `secret put` 时按提示输入自己的密钥。部署成功后，终端会显示 Worker 地址。

> Windows PowerShell 如果提示禁止执行 `npx.ps1`，将命令中的 `npx` 改为 `npx.cmd`；`npm` 也可使用 `npm.cmd`，无需修改系统执行策略。

### 绑定自定义域名（推荐）

如果 `workers.dev` 在你的网络中无法稳定访问，可绑定托管在 Cloudflare 的自定义域名：

1. 打开 **Worker → Settings → Domains & Routes → Add → Custom Domain**。
2. 填写域名，例如 `proxy.example.com`，等待证书签发完成。
3. 在 Telegram 中使用这个域名作为入口。

**无需修改 `wrangler.toml`**：`PUBLIC_HOSTNAME` 默认留空，自动使用请求主机名。仅当需要限制入口域名时才填写它。不要将个人域名写入公共仓库的 `[[routes]]`，以免其他账号 Fork 后部署失败。

## 添加到 Telegram

在支持 WEB Proxy 的 Telegram Android 客户端打开：

```text
https://t.me/webproxy?server=proxy.example.com&secret=你的PROXY_SECRET
```

- 将 `proxy.example.com` 替换为你的 Worker 域名，**不要添加 `https://` 或路径**。
- `secret` 必须与 Cloudflare 中的 `PROXY_SECRET` 完全一致，包括可选的 `dd` 前缀。
- 配置后实际测试消息收发；浏览器能打开域名不代表代理一定可用。

## 配置项

正常使用只需要配置 `PROXY_SECRET`，其他项保持默认即可。

| 名称 | 类型 | 默认值 | 说明 |
|---|---|---|---|
| `PROXY_SECRET` | Secret | 必填 | 32 位小写十六进制密钥，可加 `dd` 前缀。 |
| `PUBLIC_HOSTNAME` | 环境变量 | 空 | 可选，限制入口主机名，不含协议或路径；为空时使用请求主机名。 |
| `MAX_STREAMS` | 环境变量 | `64` | 每个会话允许的最大逻辑连接数。 |
| `SESSION_TTL_SECONDS` | 环境变量 | `300` | Bootstrap 凭证有效期（秒），不是已建立连接的最长时长。 |
| `DIAGNOSTICS` | 环境变量 | 关闭 | 临时设置为 `1` 开启连接诊断，排障后删除或设为 `0`。 |

会话签名密钥由 `PROXY_SECRET` 域隔离派生，**不需要手填 `SESSION_SIGNING_KEY`**，也不会将派生密钥发给客户端。

## 验证与排障

访问 `https://你的域名/healthz`，正常返回：

```json
{
  "ok": true,
  "carrier": "websocket",
  "relay": "direct-telegram-dc"
}
```

这只检查 Worker HTTP 入口，不验证 Durable Object 会话或 Telegram DC 连通性。根路径显示伪装页面是正常现象，只有通过 Bridge 鉴权的请求才会加载代理页面。

**无法连接时，依次检查：**

1. `PROXY_SECRET` 是否已生效，是否与 Telegram 链接一致。
2. 自定义域名是否绑定到正确的 Worker，证书是否就绪。
3. 若设置了 `PUBLIC_HOSTNAME`，是否与实际入口域名一致。
4. 临时开启 `DIAGNOSTICS=1`，查看 Worker 日志；不要公开 Secret、会话 Token 或完整 Bridge URL。

**延迟不是域名 Ping**：Telegram 显示的延迟还包含 Worker / Durable Object 调度及到 Telegram DC 的链路耗时。连接稳定后再比较，并分别测试移动网络和宽带；代码优化无法保证所有网络都降到同一延迟。

## 更新项目

- **GitHub 自动部署**：Fork 用户先同步上游；已连接 Workers Builds 的生产分支更新后会触发部署。
- **命令行部署**：

```bash
git pull
npm ci
npm test
npx wrangler deploy
```

更新成功后，在 Telegram 中关闭再开启代理，重建 Bridge 和会话。

## 性能与稳定性

- AES-CTR 优先使用 WebCrypto 原生处理，减少逐字节 JavaScript 运算。
- 同一连接已排队的小包合并处理，不额外等待凑包，保持顺序和流控。
- Bridge 发送拥堵时短暂等待，保留队列硬上限，连续 30 秒无进展则关闭，避免无限积压。

这些优化不需要新增变量，也不改变 TCP 直连和 Durable Object 架构；实际延迟仍取决于网络链路。

## 本地开发

```bash
npm ci
npm test
npx wrangler dev
```

仅检查构建、不部署：

```bash
npx wrangler deploy --dry-run --outdir .wrangler-dry-run
```

测试覆盖帧校验、连接内顺序、队列限额、流控、AES-CTR 转换及连接生命周期。模拟测试不能替代 Telegram 真机和 Cloudflare 生产网络验证。

## 协议与安全

- 协议参考 Telegram 官方 [tproxy-server](https://github.com/telegramdesktop/tproxy-server) 的 [PROTOCOL.md](https://github.com/telegramdesktop/tproxy-server/blob/master/PROTOCOL.md) 和 [ANDROID.md](https://github.com/telegramdesktop/tproxy-server/blob/master/ANDROID.md)。本项目是独立实现，**不是官方 Cloudflare 部署方案**，也未实现全部载体模式。
- 不要提交 `.dev.vars`、真实 Secret、会话 Token 或带鉴权信息的 URL。
- 可用性受本地网络、Cloudflare TCP 出站限制及 Telegram 链路影响；使用时请遵守相关服务条款及所在地法律法规。

## 许可证

[MIT License](./LICENSE)
