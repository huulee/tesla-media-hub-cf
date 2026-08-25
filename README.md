# tesla-media-hub (Cloudflare 版)

把原 `tesla-media-hub`（AppleCMS 影视聚合、适配特斯拉车机的 WebCodecs 播放器）改造为**可直接部署到 Cloudflare** 的版本：

- 前端静态资源（`public/` HTML/CSS/JS/wasm）由 **Cloudflare 静态资产（Assets）** 托管；
- 后端 **AppleCMS 点播 API** 跑在单个 **Cloudflare Worker** 里；
- 源配置 / 管理员账号用 **KV** 持久化（替代原 Docker 的 `data/` 磁盘卷）；
- **已移除 IPTV / ffmpeg**（Cloudflare 无法运行原生二进制、无法 spawn 子进程），车机点播 AppleCMS 直链的能力完整保留。

## 与原版的区别

| 项         | 原版（Docker）                            | 本版（Cloudflare）                        |
| --------- | ------------------------------------- | ------------------------------------- |
| 运行环境      | `node server/index.js` + Express 常驻进程 | 单个 Worker（`export default { fetch }`） |
| 持久化       | `fs` 写 `data/*.json`                  | KV（`TMH_KV`）                          |
| IPTV 直播   | ffmpeg 转码 / 代理                        | ❌ 已移除                                 |
| 登录态       | 内存 Map 存 token                        | 无状态 HMAC 签名 token（内嵌过期+账号版本，改密即失效）    |
| 图片代理 SSRF | `dns`/`net` 解析拦截内网                    | 协议/关键字校验 + 不跟随重定向（内网地址由 CF 网络层直接拦截）   |

> 前端 `app.js` 已去掉 IPTV 入口，点击 IPTV 源会提示「功能已禁用」。

## 部署步骤

### 1. 安装依赖

```bash
npm install
```

### 2. 创建 KV 命名空间

```bash
npx wrangler kv namespace create tesla-media-hub
# 输出类似：
#   { binding = "TMH_KV", id = "a1b2c3d4...", ... }
```

把 `id` 填到 `wrangler.toml` 里的 `[[kv_namespaces]]` 的 `id = "..."` 处。

### 3.（强烈建议）设置登录态签名密钥

不设置也能跑，但未设 `TMH_SECRET` 时每个 Worker 冷启动会随机生成密钥，重启后旧登录态失效。建议设为 Secrets（不进仓库）：

```bash
npx wrangler secret put TMH_SECRET
# 输入一段随机长字符串，例如：openssl rand -hex 32
```

### 4. 部署

```bash
npx wrangler deploy
```

部署完成后访问 `https://<你的子域>.workers.dev`，管理后台在 `/admin`。

### 5. 首次登录

默认管理员账号 `admin` / `admin123`（来自 `wrangler.toml` 的 `ADMIN_USER`/`ADMIN_PASS`，也可在管理后台网页修改，修改后写入 KV 覆盖）。**请务必在管理后台修改默认密码。**

## 本地开发

```bash
npx wrangler dev
```

`wrangler dev` 会用本地预览 KV，无需真实 KV id 即可联调（部分功能会读取/写入本地模拟 KV）。

## 使用说明

- 首页选择数据源 → 浏览分类 / 搜索 → 打开详情 → 点选集即播放（车机本地 WebCodecs 解码到 Canvas，无 `<video>` 标签）。
- 管理后台 `/admin`：添加 AppleCMS 源（采集接口形如 `https://域名/api.php/provide/vod/`）、编辑/删除、修改管理员密码。
- 默认已内置几个 AppleCMS 源；首次运行写入 KV 后可随时增删。

## 已知限制 / 注意事项

- **仅支持 AppleCMS 直链 JSON 接口**（`ac=list`/`ac=detail`/`ac=videolist`），依赖 Spider/XPath 解析的站点无法播放。
- **部分影视源是 `http://`**，Cloudflare Workers 的子请求对纯 HTTP 支持有限，建议优先添加 `https://` 的源。
- 影视聚合类应用涉及版权与地区合规，请确保在合法授权范围内使用；公网部署务必修改默认密码。

## 目录结构

```
tesla-media-hub-cf/
├── wrangler.toml          # Worker + Assets + KV 配置
├── package.json
├── src/
│   ├── index.js           # Worker 入口（fetch handler + 路由）
│   └── lib/
│       ├── store.js       # KV 持久化 + 无状态 token 鉴权
│       ├── sites.js       # AppleCMS 站点适配（首页/分类/搜索/详情）
│       ├── fetcher.js     # fetch 封装（超时/UA）
│       ├── parsePlay.js   # 播放地址解析（$$$/#/$ 拆分）
│       ├── resolvePlay.js # HTML 跳转页真实地址解析
│       └── sourceParser.js# 源解析（仅 applecms）
└── public/                # 前端静态资源（原样托管）
```

