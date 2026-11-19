# chat-demo 部署速记：GitHub Pages 与 Cloudflare Workers

把 DeepSeek 官方开源的 chat-demo 变成自己的网址。纯前端项目，不需要服务器，也不需要在平台上配 API Key——Key 由每个访问者在网页里自己填，只存他自己的浏览器。

## 来源与协议（先说清楚）

| 项目 | 内容 |
|---|---|
| 上游项目 | [deepseek-ai/chat-demo](https://github.com/deepseek-ai/chat-demo)，官方在线版 https://deepseek-ai.github.io/chat-demo/ |
| 上游协议 | **MIT License**，Copyright (c) 2026 DeepSeek。已核对仓库 `LICENSE` 原文与 GitHub API 的 license 字段，一致 |
| 本文来源 | 整理自《chat-demo 完整教程 · 部署与换装》HTML，仅保留两条部署路线；上游仓库的构建配置、工作流、产物结构已按当前 main 复核 |
| 版本差异 | 原 HTML 标注基于 v0.1.1；核对时仓库 main 的 `package.json` 已是 **0.1.2**（tag 有 v0.1.0 / v0.1.1 / v0.1.2），部署相关配置未变 |
| 一键部署模板 | [Git-xian/chat-demo](https://github.com/Git-xian/chat-demo)：从上游 Fork，加了 `wrangler.jsonc` 和可选的云端同步接口，专供下面的 CF 按钮克隆。Fork 保留上游 `LICENSE` 与来源信息，符合 MIT 要求 |

> 本仓库的改动（多出的 `worker.ts`、`src/services/chat/sync.ts`、设置里的同步区块）都是新增的原创代码，用于自托管备份，遵循上游 MIT；不改变上游授权，也不声称官方背书。

MIT 允许修改、复制、分发甚至商用，条件是**保留原版权声明和许可声明**。上游仓库自带 `LICENSE`，且 `rspack.config.ts` 里有把它复制进 `dist` 的配置——正常 Fork 或 Clone 就满足合规，**别删 `LICENSE`**。改标题、换图标、改代码都可以，但不要在改版里声称是 DeepSeek 官方出品或获得背书。

如果只是想用，直接打开官方在线版填自己的 Key 即可，不必部署。自建的意义只有三个：自己的域名、代码在自己手里可改、独立入口。

## 准备

- GitHub 账号：两条路线都要用（CF 路线用它拉代码）
- DeepSeek API Key：`sk-` 开头，在 https://platform.deepseek.com 注册、充值、创建。**部署时不需要它**，是访问者打开网页后自己填的
- CF 路线另外需要：Node.js 20+、pnpm、Cloudflare 免费账号。**静态部署（路线 B 基础版）不用绑卡；但若要用下面的「云端同步」，得先在 CF 后台开通 R2，这一步要绑支付方式**（免费额度内不扣钱，详见同步章节）

## 路线 A · GitHub Pages（浏览器点鼠标，约 5 分钟）

1. 打开 https://github.com/deepseek-ai/chat-demo → 右上角 **Fork** → 默认设置 → Create fork
2. 去 `https://github.com/你的用户名/chat-demo/settings/pages`，把 **Build and deployment → Source** 从 `Deploy from a branch` 改成 **`GitHub Actions`**。这步不能跳：工作流的 build 任务只有 `pages: read` 权限，不先切换来源，deploy 会报 `Get Pages site failed`
3. 去 `https://github.com/你的用户名/chat-demo/actions/workflows/pages.yml`。若顶部提示 *Workflows aren't being run on this forked repository*，先点里面的启用按钮。然后 **Run workflow** → 分支选 `main` → 点绿色 Run workflow
4. 等 `build` 和 `deploy` 两个任务都变绿勾。网址固定为：
   ```
   https://你的用户名.github.io/chat-demo/
   ```
5. 打开网址 → 左下角 Settings → General → 粘入 `sk-` 开头的 Key，即可对话。Key 只写进当前浏览器，别人打开看到的是空白，要各自填各自的

以后改完代码 push 到 `main`，工作流自动重新构建发布，不用再手动点 Run workflow。

## 路线 B · Cloudflare Workers

原理：把构建好的 `dist` 交给 Workers 的静态资源托管，Cloudflare 生成一个只发文件的 Worker，不用写服务端代码，也不用配环境变量。

### 一键部署（推荐）

点下面按钮，Cloudflare 会把[模板仓库](https://github.com/Git-xian/chat-demo)克隆到你自己的 GitHub 账号，自动装依赖、构建、部署。全程不用装 Node，也不用本地敲命令。

[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/Git-xian/chat-demo)

流程里会让你确认仓库名、Worker 名，构建命令填 `pnpm build`，部署命令保持默认 `npx wrangler deploy`。部署完打开给的 `workers.dev` 地址，左下角 Settings → General 填自己的 `sk-` Key 即可。

> 按钮指向的是带 `wrangler.jsonc` 的模板仓库（否则 Cloudflare 找不到静态资源配置）。它比手动路线少一步，但构建配置是固定的；想改 `name`、缓存规则这些，部署完在 Cloudflare 的 Worker → Settings → Build 里改，或克隆后改配置文件。

### 手动部署（约 10 分钟）

```bash
git clone https://github.com/deepseek-ai/chat-demo.git
cd chat-demo
pnpm install
pnpm build
```

Windows 注意：`build` 脚本里含 `rm -rf dist`，在 cmd / PowerShell 下会报错。用 **Git Bash** 或 WSL 执行，或先把脚本改成跨平台的 `node -e "require('fs').rmSync('dist',{recursive:true,force:true})"`。

成功后多出 `dist/`。在项目根目录（与 `package.json` 同级）新建 `wrangler.jsonc`：

```jsonc
{
  "name": "chat-demo",
  "compatibility_date": "2026-11-19",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application"
  }
}
```

这是纯静态版。想要下面的「云端同步」，把模板仓库里的 `worker.ts`、`src/services/chat/sync.ts`、`src/components/settings/SyncSetting.tsx` 也拷进来，并用模板仓库那份带 `main` 和 `r2_buckets` 的 `wrangler.jsonc`。

然后登录并部署：

```bash
npx wrangler@latest login
npx wrangler@latest deploy
```

浏览器没弹出就把终端打印的 `dash.cloudflare.com/oauth2/auth?...` 链接手动打开。首次部署会问是否创建 Worker（输 `y`）、是否发送使用统计（输 `N`）。最后一行就是站点地址：

```
https://chat-demo.你的子域.workers.dev
```

以后更新：

```bash
git pull
pnpm install
pnpm build
npx wrangler@latest deploy
```

不想每次敲命令：把代码推到自己的 GitHub 仓库并保留 `wrangler.jsonc`，然后 Cloudflare Dashboard → Workers & Pages → Create → **Import a repository**，构建命令填 `pnpm build`，部署命令保持默认 `npx wrangler deploy`。之后 push 到 main 自动发布。

**不要给 Worker 配 API Key 环境变量**：应用是纯前端，请求直连 `api.deepseek.com`，Worker 只发文件。把 Key 配进 Worker 既读不到，还多一处泄露面。

## 云端同步（可选，仅 CF 版 / 模板仓库）

默认所有数据只在本机。模板仓库多做了一个开关：**把聊天记录备份到 Cloudflare R2**，解决 Safari 七天清数据、换设备、清缓存丢记录的问题。只搬聊天记录，**API Key 永远不上传**，它仍只存在填 Key 的那台设备浏览器里。

原理很简单：点「上传」就是把现有的整份历史 ZIP 存进你自己的 R2；点「恢复」就是下载下来走一遍原有的导入流程（有覆盖确认、校验、原子替换）。所以云端和本地是**整包覆盖**关系，不是逐条合并——谁后写谁生效，别在两台设备各聊一半再互相覆盖。

### 用一键按钮部署（已带同步）

按钮指向的模板仓库已经包含同步代码和 R2 配置。Cloudflare 部署按钮会**自动创建并绑定 R2 bucket**（官方支持 KV / D1 / R2 等资源的自动配置），部署时按提示确认资源名即可，不用手动建 bucket、也不用配密钥。构建命令 `pnpm build`，部署命令保持 `npx wrangler deploy`。部署完打开站点，会看到多了同步区块。

### 手动部署要改的两处

1. 按上面路线 B 拷入三个新文件（`worker.ts`、`src/services/chat/sync.ts`、`src/components/settings/SyncSetting.tsx`），i18n 两个文件和 `SettingsDialog.tsx`、`Settings.module.css` 用模板仓库的最新版。
2. `wrangler.jsonc` 换成带 Worker 入口和 R2 绑定的版本：

```jsonc
{
  "name": "chat-demo",
  "main": "worker.ts",
  "compatibility_date": "2026-11-19",
  "assets": {
    "directory": "./dist",
    "not_found_handling": "single-page-application",
    "binding": "ASSETS",
    "run_worker_first": ["/api/*"]
  },
  "r2_buckets": [{ "binding": "BACKUP", "bucket_name": "chat-demo-sync" }]
}
```

先建 bucket：`npx wrangler r2 bucket create chat-demo-sync`（名字要和 `bucket_name` 一致；已有 bucket 可改名复用）。然后 `pnpm build && npx wrangler deploy`。`run_worker_first` 让 `/api/*` 走 Worker、其余走静态文件，这是同步接口能被请求到的关键。

### 怎么用

1. 左下角 Settings → Data → 云端同步，点「生成」得到一串 32 位同步码，它自动保存在本机。
2. 点「上传到云端」，整份聊天记录存入 R2。
3. 另一台设备打开同一网址，在同步码框里**填同一串码**，点「从云端恢复」→ 确认覆盖，记录就过去了。以后哪台更新了就重新上传，另一台再恢复。

**同步码就是密码**：服务端不存用户身份，谁能报出这串码，谁就能读写那份备份。用「生成」按钮出随机码，别用生日、手机号之类可猜的；也别把码发到公开渠道。它只存本机 `localStorage`（键 `chat-demo:sync-code`），不进导出归档、不发往 DeepSeek。忘了码等于丢了那份云端备份的入口。

### 边界

- 只同步聊天记录和图片，API Key、主题、语言等设置不跨设备，换设备要各自填 Key。
- 整包覆盖，不合并、不做冲突处理；多设备请固定「一台写、其余只读恢复」。
- **开通门槛**：R2 虽然免费额度很宽（10 GB/月存储、100 万次 A 类操作），但要在 CF Dashboard → Storage & databases → R2 → Overview 走一遍订阅开通，**需绑支付方式**；只是不超额度就不扣费。不想绑卡就用不了同步，静态部署不受影响。
- 备份里没有加密，R2 由你的 CF 账号和同步码两层把守，别在共享电脑上留着同步码。
- GitHub Pages 版本没有后端，不支持同步；设置里会提示需要 CF Worker 版。

## 常见坑

| 现象 | 处理 |
|---|---|
| deploy 报 `Get Pages site failed` | Pages 的 Source 没切成 GitHub Actions，回路线 A 第 2 步 |
| `assets directory does not exist` | 没先跑 `pnpm build`，wrangler 不会替你构建；确认 `dist/index.html` 存在 |
| Pages 打开 404 | URL 结尾带斜杠、仓库名是 `chat-demo` 不是 `chat_demo`；到 Settings → Pages 看是否有 live 提示 |
| 更新后还是旧版本 | Cloudflare 边缘缓存了 `index.html`，无痕窗口验证或等一两分钟 |
| 权限报错 10007 / 10000 / 403 | 重新 `wrangler login`；用 API Token 时确认有 `Workers Scripts: Edit` 与 `Account: Read` |
| Fork 后 Actions 不跑 | 先启用 fork 上的工作流，且提交要落在 `main` |
| 同步区块点上传报「无法连接云端」 | 部署的不是模板仓库（缺 `worker.ts`），或 `wrangler.jsonc` 没配 `main` / `r2_buckets`；Pages 版不支持同步 |
| 同步报「云端还没有备份」 | 这台设备还没上传过，或同步码填错；换设备要用同一串码 |
| 部署报 `bucket not found` | R2 bucket 没建或名字和 `bucket_name` 不一致；`npx wrangler r2 bucket create chat-demo-sync` |

## 部署之后

Key、主题、语言存在访问者浏览器的 `localStorage`（键名 `chat-demo:api-key` 等），聊天记录和附件存在 IndexedDB（库名 `chat-demo`）。默认情况下服务器没有副本，换设备、清缓存、换域名都不会跟过去；可以手动导出 zip，或按上面「云端同步」把记录备份到自己的 R2。**API Key 不随同步上传**，只留在填 Key 的那台设备。Safari 连续 7 天不访问会清掉本地存储，用 Chrome / Edge、保持每周打开一次，或开着云同步。

绑自己的域名两个平台都支持，但不会让国内访问变快——慢在线路，不在应用。
