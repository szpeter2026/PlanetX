# app.genz.ltd 部署指南

> **产物：** `PlanetX/` 根目录静态 PWA（`index.html` + `sw.js` + `public/`）  
> **API：** 统一走 **Looma API**（`/v1/auth/*` + `/v1/game/*`），海外 `api.genz.ltd` / 大陆 `api.szbolent.cn`，按 hostname 自动切换  
> **Supabase：** 已从主流程移除。Genzer 线 Supabase（demoppi | Genzers）独立演进，不受影响  
> **更新：** 2026-07-17  

---

## 0. 前置检查

本地确认这些文件存在且已提交：

```
PlanetX/
├── index.html
├── sw.js
├── vercel.json
├── .vercelignore
├── icon-192.png / icon-512.png / apple-touch-icon.png  （或 public/ 内同名）
└── public/manifest.json
```

---

## 1. Vercel 新建项目

### 1.1 连接仓库

1. 打开 [vercel.com/new](https://vercel.com/new)
2. **Import Git Repository**
   - 若 PlanetX 仅在 Gitee：先用 GitHub 镜像，或 **Vercel CLI 本地部署**（见 §1.4）
   - 推荐：将 `PlanetX` 同步到 GitHub 后 Import

### 1.2 项目设置（Configure Project）

| 字段 | 值 |
|------|-----|
| **Project Name** | `planetx-app`（或 `app-genz-ltd`） |
| **Framework Preset** | **Other** |
| **Root Directory** | `.`（仓库根即 PlanetX 根；若 monorepo 外层则填 `PlanetX`） |
| **Build Command** | 留空，或 `echo "static"` |
| **Output Directory** | `.` |
| **Install Command** | 留空 |

> ⚠️ 不要选 Vite/Next.js；这是纯静态站点，无 build 步骤。

### 1.3 环境变量

无需配置。`index.html` 纯静态，按 `window.location.hostname` 自动选 API：

| hostname 匹配 | 指向 API |
|---|---|
| `genz.ltd` / `vercel.app` | `https://api.genz.ltd`（海外 Vultr） |
| 其他（szbolent / IP） | `https://api.szbolent.cn`（大陆） |

同一套 JWT + 合约，数据按区域隔离。

### 1.4 无 GitHub 时：CLI 部署

```bash
cd /Users/jason/Projects/PlanetX
npx vercel login
npx vercel --prod
# 按提示 Link to existing project 或 Create new
```

---

## 2. 绑定域名 app.genz.ltd

### 2.1 Vercel 侧

1. Project → **Settings** → **Domains**
2. Add：`app.genz.ltd`
3. 记录 Vercel 显示的 DNS 目标（通常为 `cname.vercel-dns.com`）

### 2.2 Cloudflare 侧（genz.ltd .zone）

| 类型 | 名称 | 内容 | 代理 |
|------|------|------|------|
| **CNAME** | `app` | `cname.vercel-dns.com` | **DNS only（灰云）** 或 Proxied 均可 |

> Vercel 官方建议子域 CNAME 到 `cname.vercel-dns.com`。  
> 若橙云 Proxied，SSL 由 Cloudflare + Vercel 双层处理，一般可用。

### 2.3 等待生效

- DNS：通常 5–30 分钟  
- Vercel 证书：自动签发 Let's Encrypt  

验证：

```bash
curl -I https://app.genz.ltd
curl -s https://app.genz.ltd/manifest.json | head
```

---

## 3. CORS（api.genz.ltd）

已写入 `looma-zervi/.github/workflows/deploy-overseas.yml`：

```
CORS_ORIGINS=https://genz.ltd,https://www.genz.ltd,https://app.genz.ltd
```

### 使生产生效（二选一）

**A. 打 tag 触发海外部署（推荐）**

```bash
cd looma-zervi
git add .github/workflows/deploy-overseas.yml
git commit -m "chore(overseas): CORS allow app.genz.ltd"
git tag overseas-$(date +%Y%m%d)-cors
git push origin main --tags
```

**B. SSH 手动改 Vultr**

```bash
ssh root@139.180.184.25
nano /opt/looma-zervi/backend/.env
# CORS_ORIGINS=...https://app.genz.ltd
cd /opt/looma-zervi/docker && docker compose -p looma-zervi restart
```

验证：

```bash
curl -s https://api.genz.ltd/health
curl -s -X OPTIONS https://api.genz.ltd/v1/auth/register \
  -H "Origin: https://app.genz.ltd" \
  -H "Access-Control-Request-Method: POST" -I | grep -i access-control
```

---

## 4. Looma API（统一后端）

`index.html` 直连 Looma API，使用 JWT token 认证（`localStorage.planetx_token`）。

### 用户生命周期

| 操作 | 端点 | method |
|------|------|--------|
| 注册 | `/v1/auth/register` | POST |
| 登录 | `/v1/auth/login` | POST |
| 人格测试 | `/v1/game/personality` | POST |
| 任务列表 | `/v1/game/tasks` | GET |
| 舰队 | `/v1/game/fleet` | GET/POST |

### 身份一致性边界

| 场景 | 一致性 |
|------|--------|
| 同一 API 域内（如都在 `api.genz.ltd`） | ✅ 同一账号、同一份进度 |
| 海外注册 ↔ 大陆登录 | ⚠️ 暂不同步（各自 SQLite，数据按区域隔离） |

> 若后续要「一个账号全球通用」，加 DB 同步或统一 PostgreSQL 即可，前端不用改。

---

## 5. 联调实测清单

### 5.1 静态 + PWA

- [ ] `https://app.genz.ltd` 打开首页  
- [ ] Chrome DevTools → **Application** → Manifest 无报错  
- [ ] Service Worker 状态 **activated**  
- [ ] 地址栏或菜单出现 **安装应用 / 添加到主屏幕**  
- [ ] 安装后从桌面图标打开，`display: standalone`  

**iOS Safari：** 分享 → **添加到主屏幕**

### 5.2 Looma API 闭环测试

- [ ] 注册新账号（`/v1/auth/register`）  
- [ ] 登录 → 获取 `planetx_token` 存入 localStorage  
- [ ] 完成人格测试并看到结果  
- [ ] 刷新页面 → 仍保持登录状态（Token 持久化）  
- [ ] Console 无 `[Auth] Supabase` 相关日志（已移除）  

### 5.3 API 健康检查

```bash
curl -s https://api.genz.ltd/health
# 期望 JSON 含 ok / healthy
```

---

## 6. 常见问题

| 现象 | 处理 |
|------|------|
| **部署成功但全站 404** | **Root Directory 必须留空**（仓库根已有 `index.html`，不要填 `PlanetX`）；**Output Directory 留空**；Framework = Other；Redeploy |
| Vercel 部署超时 / 体积过大 | 确认 `.vercelignore` 未误排除 `index.html` / `public/`（精简仓用最小 ignore） |
| 404 on `/manifest.json` | 确认 `public/manifest.json` 存在；或根目录有 `manifest.json` |
| SW 不更新 | `sw.js` 已设 `Cache-Control: no-cache`；用户需关闭再开 PWA |
| 注册/登录失败 | 检查 `api.genz.ltd` 健康状态；CORS 是否含 `app.genz.ltd` |
| API CORS 红字 | 确认 Vultr `.env` 的 `CORS_ORIGINS` 已含 `app.genz.ltd` 并重启容器 |
| 大陆访问走错 API | 确认 `index.html` 的 `API_BASE` 自动切换逻辑按 hostname 正确匹配 |

---

## 7. 与 genz.ltd 营销站关系

| 域名 | 用途 | 部署 |
|------|------|------|
| `genz.ltd` | 营销 / Stripe 审查 | `genz-web` → Vercel |
| `app.genz.ltd` | 产品 PWA · 人格测试 | `PlanetX` 根静态 → Vercel |
| `api.genz.ltd` | Looma API | Vultr · `deploy-overseas` |

营销页可加按钮：`https://app.genz.ltd` → 「Start PlanetX Test」
