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

## 方式二：通过 GitHub 一键 Git 部署（推荐，持续交付）

把仓库推到 GitHub 后，在 Cloudflare 后台「连接 Git」，之后每次 `git push` 到 `main` 自动部署，PR 自动出预览环境。

> ⚠️ 前置条件：**KV 命名空间必须先存在**，且 `wrangler.toml` 里的 `id` 已填真实值。Cloudflare 的 Git 集成**不会**自动创建 KV，绑定 id 不存在会导致部署失败。

### A. 推送到 GitHub

```bash
cd tesla-media-hub-cf
git remote add origin git@github.com:<你的用户名>/tesla-media-hub-cf.git
git branch -M main
git push -u origin main
```

### B. 在 Cloudflare 后台连接仓库

1. 登录 Cloudflare 控制台 → **Workers & Pages** → **Create** → **Connect to Git**。
2. 授权 GitHub，选择仓库 `tesla-media-hub-cf`、生产分支 `main`。
3. Cloudflare 会读取 `wrangler.toml` 自动识别：
   - `main = src/index.js` → Worker 入口；
   - `[assets]` → 静态资源托管 `public/`；
   - `[[kv_namespaces]] binding = "TMH_KV"` → KV 绑定（id 已在 toml 中）。
4. 点击 **Deploy** 完成首次构建。

### C. 配置变量与 Secret（关键）

Git 集成部署时，以下配置从 `wrangler.toml` 与后台「变量和机密」读取，**无法从 Git 读取 Secret**，需手动设置：

| 类型 | 名称 | 说明 | 设置位置 |
| ---- | ---- | ---- | ---- |
| **KV 绑定** | `TMH_KV` | 源配置/管理员账号存储（id 已在 `wrangler.toml` 填好；若后台未自动识别，需在 Worker → Settings → Variables and Secrets → Add → KV 选命名空间） | 后台 |
| **明文变量** | `ADMIN_USER` / `ADMIN_PASS` | 默认管理员账号（已写在 `wrangler.toml` 的 `[vars]`，会随 Git 部署自动应用） | 无需额外操作（在 toml 内） |
| **Secret（机密）** | `TMH_SECRET` | 登录态 HMAC 签名密钥。**必须手动设**，否则每次冷启动随机密钥、登录态易失效 | Worker → Settings → Variables and Secrets → **Add** → 选 **Secret** |

设置 `TMH_SECRET`（两个环境都要加：Production 和 Preview，否则 PR 预览环境会登录异常）：

1. Worker 详情 → **Settings** → **Variables and Secrets** → **Add**。
2. 类型选 **Secret**，名称 `TMH_SECRET`，值填一段随机串（如本地执行 `openssl rand -hex 32` 的结果）。
3. 对 **Production** 和 **Preview** 环境分别添加一次。

> 账号 ID（`account_id`）：Git 集成部署时由后台所选账号决定，可不填；仅本地 `wrangler dev/deploy` 时需要，到 Cloudflare 后台右上角「账户 ID」获取后填入 `wrangler.toml`。

### D. 之后如何更新

```bash
git commit -am "..." && git push   # 自动触发 Cloudflare 生产部署
# 开 PR → 自动出 Preview 预览环境
```

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

