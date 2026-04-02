# CloudflareSub

基于 Cloudflare Workers + Workers KV 的订阅生成器：
- 输入原始节点（`vmess://` / `vless://` / `trojan://`）
- 批量替换为优选 IP / 域名
- 生成可分享的短链订阅（Raw / Clash / Surge）
- 内置 Web 页面，支持一键复制与二维码展示

## 功能特性

- 支持 `vmess`、`vless`、`trojan` 三种节点解析
- 支持输入 Base64 订阅文本并自动展开
- 批量替换节点出口地址（支持 `host:port#备注`）
- 结果落盘到 Workers KV，生成短链（`/sub/:id`）
- 相同输入内容自动去重（7 天 TTL）
- 支持访问令牌保护（`SUB_ACCESS_TOKEN`）
- 支持导出：
  - Raw（Base64）
  - Clash（YAML）
  - Surge（配置文本）

## 项目结构

```text
cloudflaresub/
├─ src/
│  ├─ worker.js      # Worker 入口（API + 订阅输出）
│  └─ core.js        # 解析/渲染核心函数（测试使用）
├─ public/           # 前端页面静态资源
├─ tests/smoke.mjs   # 基础 smoke test
├─ wrangler.toml
└─ package.json
```

## 环境要求

- Node.js 18+
- Wrangler 4+
- 一个 Cloudflare Workers KV Namespace

## 快速开始

### 1) 安装依赖

```bash
npm install
```

### 2) 创建 KV Namespace

```bash
npx wrangler kv namespace create SUB_STORE
```

记下输出中的 `id`，填入 `wrangler.toml`：

```toml
[[kv_namespaces]]
binding = "SUB_STORE"
id = "<your-kv-namespace-id>"
```

### 3) 配置访问令牌（建议）

```bash
npx wrangler secret put SUB_ACCESS_TOKEN
```

说明：
- 已设置时，请求 `/sub/:id` 必须携带 `?token=...`
- 未设置时，订阅链接不做二次鉴权（前端会给 warning）

### 4) 本地开发

```bash
npm run dev
```

### 5) 部署

```bash
npm run deploy
```

## API 说明

### `POST /api/generate`

输入原始节点与优选地址，返回短链订阅。

请求体示例：

```json
{
  "nodeLinks": "vmess://...\nvless://...",
  "preferredIps": "104.16.1.2#HK\n104.17.2.3:2053#US",
  "namePrefix": "CF",
  "keepOriginalHost": true
}
```

字段说明：
- `nodeLinks`: 多行节点链接
- `preferredIps`: 多行优选地址，格式 `host[:port][#remark]`
- `namePrefix`: 节点名附加前缀
- `keepOriginalHost`: 是否保留原始 Host/SNI（默认 `true`）

返回示例（节选）：

```json
{
  "ok": true,
  "shortId": "AbC123xYz9",
  "urls": {
    "auto": "https://<worker>/sub/AbC123xYz9?token=...",
    "raw": "https://<worker>/sub/AbC123xYz9?target=raw&token=...",
    "clash": "https://<worker>/sub/AbC123xYz9?target=clash&token=...",
    "surge": "https://<worker>/sub/AbC123xYz9?target=surge&token=..."
  }
}
```

### `GET /sub/:id`

按 `target` 返回订阅内容：
- `target=raw`（默认）
- `target=clash`
- `target=surge`

示例：

```bash
curl "https://<worker>/sub/<id>?target=clash&token=<SUB_ACCESS_TOKEN>"
```

## 前端页面

默认根路径 `/` 提供网页表单（由 `public/` 静态资源提供）：
- 粘贴节点链接
- 粘贴优选 IP / 域名
- 生成并展示各客户端订阅链接
- 一键复制 / 生成二维码

## 测试

```bash
npm run check
```

当前为 smoke test，覆盖：
- 节点解析
- 节点扩展
- 各格式渲染
- 加解密能力（`core.js`）

## 注意事项

- `src/worker.js` 当前使用 KV 短链方案，不依赖 `SUB_LINK_SECRET`
- 每条订阅记录默认保存 7 天（TTL）
- Surge 导出当前仅包含 `vmess` / `trojan`

## License

MIT
