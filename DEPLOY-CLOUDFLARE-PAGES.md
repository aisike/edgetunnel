# Cloudflare Pages 部署指南（edgetunnel）

本仓库用 **Cloudflare Pages advanced mode** 部署：Pages 直接读取仓库根目录的原始 `_worker.js`（Module Worker 语法 `export default { fetch }`），不做 esbuild 重打包，避免 Workers 方式把大文件压缩损坏导致 522。

> 为什么改用 Pages：edgetunnel 官方推荐 Pages。之前用 Workers + wrangler 打包时，2.8MB 的 `_worker.js` 被压成 382KB，Worker 启动即崩溃、请求 522。Pages advanced mode 用原始文件，正好绕开这个问题。

## 一、部署链路

| 环节 | 说明 |
|---|---|
| 上游同步 | `sync.yml` 每天从 `cmliu/edgetunnel` 同步到本 fork 的 `main` |
| 自动部署 | `deploy.yml` 检测 `main` 更新，执行 `wrangler pages deploy` |
| 部署目标 | Cloudflare Pages 项目，默认名 `edgetunnel-sanbay` |
| 对外域名 | `sanbay.12348123.xyz`（在 Pages 项目里绑定） |

## 二、Cloudflare 端准备（一次性）

### 1. 创建 KV 命名空间
`存储和数据库` → `Workers KV` → 创建命名空间（如 `EDT2`），复制 **Namespace ID**（32 位十六进制，不是名称）。

### 2. 获取 Account ID
`Workers 和 Pages` 页面右侧栏或 URL 中的 **Account ID**。

### 3. 创建 API Token
`我的个人资料` → `API 令牌` → 创建令牌，添加权限：

| 类型 | 资源 | 权限 |
|---|---|---|
| Account | Cloudflare Pages | Edit |
| Account | Workers Scripts | Edit |
| Account | Workers KV Storage | Edit |
| Account | Account Settings | Read |
| Zone | Zone | Read |
| Zone | DNS | Edit |

Zone 资源选 `12348123.xyz`（用于自定义域证书与 DNS）。

## 三、GitHub 端配置

进入仓库 `Settings` → `Secrets and variables` → `Actions`。

### Secrets（必填）
| 名称 | 值 |
|---|---|
| `CLOUDFLARE_API_TOKEN` | 上面创建的 API Token |
| `CLOUDFLARE_ACCOUNT_ID` | Account ID |

### Variables（可选）
| 名称 | 默认值 | 说明 |
|---|---|---|
| `CF_PAGES_PROJECT` | `edgetunnel-sanbay` | Pages 项目名，需全局唯一；填一个全新名字可避免 1101 |

## 四、首次部署步骤（顺序很重要）

1. 配好上面两个 Secrets，并把 `CF_PAGES_PROJECT` Variable 设为一个独特名字（例如 `edt-sanbay-2026`）。
2. `Actions` → `Deploy edgetunnel to Cloudflare Pages` → `Run workflow`。首次运行会用 `wrangler pages deploy` **自动创建**这个 Pages 项目。
3. 部署成功后，进入 Cloudflare 控制台 → `Workers 和 Pages` → 刚建的项目 → `设置`：
   - **环境变量**：添加 `ADMIN` = 后台密码（建议设为「Secret/机密」类型）
   - **绑定 / 函数绑定**：添加 `KV 命名空间`，**变量名称必须精确填 `KV`**，命名空间选第 1 步建的 `EDT2`
   - 保存后，回到 `部署` 页对最新一次部署点 **重试部署**（或重新 Run workflow），让绑定和变量生效
4. `自定义域` → 设置自定义域 → 填 `sanbay.12348123.xyz` → 激活（DNS 在 CF，会自动建记录与证书）。
5. 访问 `https://sanbay.12348123.xyz/admin`，用 `ADMIN` 密码验证。

> 注意：Pages 项目的 `ADMIN` 和 `KV` 必须在 Cloudflare 控制台设置，**不能**只放在 GitHub Secrets——Pages advanced mode 读取的是项目自身的绑定。

## 五、常见问题

| 现象 | 排查方向 |
|---|---|
| 部署日志提示 Missing secret | 检查两个 GitHub Secret 是否拼写正确 |
| `Couldn't find a project` / 权限错 | API Token 补 `Cloudflare Pages: Edit` 权限 |
| 1101 | Pages 项目名太通用，换 `CF_PAGES_PROJECT` 为独特名字重跑 |
| 522 依旧 | 确认用的是 Pages（不是 Worker）、`ADMIN`/`KV` 已在 Pages 项目设置并重试部署 |
| 后台数据不保存 | 检查 Pages 项目的 KV 绑定变量名是否为 `KV` |
| 改 ADMIN 不生效 | 改 Pages 项目环境变量后需重试部署 |
