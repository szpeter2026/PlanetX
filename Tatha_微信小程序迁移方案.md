# Tatha 前端预览项目 → 微信小程序 迁移方案

> 版本：v1.1  
> 日期：2026-06-25  
> 目标：将现有的 Tatha 纯前端演示项目（HTML/CSS/JS）完整迁移为微信小程序

---

## 0. 与 looma-zervi 项目的关系（前置背景）

### 0.1 项目血缘

`tatha-frontend-preview` **不是孤立项目**，它是 **looma-zervi 后端服务的官方前端预览**。

```
┌──────────────────────────────────────────────────────────┐
│                    Looma & Zervi 融合项目                  │
│                                                          │
│  ┌─────────────────────┐    ┌──────────────────────────┐ │
│  │  looma-zervi 后端    │◄───│ tatha-frontend-preview   │ │
│  │  (Python FastAPI)   │    │ (静态 HTML 前端预览)      │ │
│  │  localhost:8010     │    │                          │ │
│  │                     │    │  index.html  定价落地页   │ │
│  │  /v1/ask            │    │  auth.html   登录/注册   │ │
│  │  /v1/jobs/match     │    │  demo.html   核心体验    │ │
│  │  /v1/auth/*         │    │                          │ │
│  │  /v1/region         │    │   ── 未来 ──►            │ │
│  │  /v1/documents/*    │    │  微信小程序（本文档目标） │ │
│  └─────────────────────┘    └──────────────────────────┘ │
│                                                          │
│  ┌─────────────────────┐                                │
│  │  Zervi Rust 客户端   │  桌面端 / 树莓派，本地优先     │
│  └─────────────────────┘                                │
└──────────────────────────────────────────────────────────┘
```

### 0.2 关键证据

| 证据来源 | 内容 | 含义 |
|----------|------|------|
| `looma-zervi/docs/api.yaml` 第13-16行 | `移动端（小程序 / 微信生态）：手机为主载体，调 Looma API` | **已明确规划小程序端** |
| `looma-zervi/src/api/routes/` | 包含 `ask.py`, `jobs.py`, `resume.py`, `auth_routes.py`, `region.py` | 完全覆盖前端调用的 6 个接口 |
| API 端口一致 | 后端 `uvicorn --port 8010` ↔ 前端 `fetch('http://localhost:8010/...')` | 直接对应 |
| 配额体系一致 | 后端三档（free/basic/pro）+ 403/429 → 前端三档定价卡片 | **前后端配额联动** |
| 融合决策文档 | `Tatha与DemoPeter_分析融合决策文档.md` 明确 Tatha 部分即此前端 | Tatha = FastAPI 8010 + 静态 HTML |

### 0.3 looma-zervi 后端现状（P1 底座已完成）

| 模块 | 状态 | 技术栈 | 小程序是否调用 |
|------|------|--------|---------------|
| `/v1/ask` 中央大脑 | ✅ done | LiteLLM + LlamaIndex + pgvector | ✅ 核心入口 |
| `/v1/jobs/match` 职位匹配 | ✅ done | LLM 多维打分管道 | ✅ 核心功能 |
| `/v1/auth/*` 认证 | ✅ done | Supabase JWT + Stub fallback | ✅ 必须 |
| `/v1/region` 区域定价 | ✅ done | -- | ✅ 定价页 |
| `/v1/documents/convert` 文件解析 | ✅ done | MarkItDown + PydanticAI | ✅ 简历上传 |
| `/v1/resume/parse` 简历解析 | ✅ done | PydanticAI | ✅ 可选增强 |
| 两层缓存（LLM prompt + 请求结果） | ✅ done | TTL + LRU | -- |
| 多 Provider fallback（ollama→deepseek→openai） | ✅ done | 弹性策略 | -- |
| 熔断器（5次失败/30s冷却） | ✅ done | 自研 | -- |

### 0.4 迁移的架构意义

将 `tatha-frontend-preview` 迁移为微信小程序后，looma-zervi 的终端生态将形成完整闭环：

```
          ┌──────────────────────────────────┐
          │       Looma Python 服务端         │
          │      FastAPI :8010               │
          │      pgvector + LiteLLM           │
          └──────┬───────┬───────┬───────────┘
                 │       │       │
          ┌──────┴─┐ ┌───┴───┐ ┌┴──────────┐
          │ Zervi  │ │ Web   │ │ 微信小程序  │
          │ Rust   │ │ 预览  │ │ (本次目标)  │
          │ 桌面端  │ │ 浏览器 │ │ 移动端     │
          └────────┘ └───────┘ └────────────┘
```

**本次迁移只涉及前端层**，后端 looma-zervi **零改动**，只需对外暴露 HTTPS 域名即可。

---

## 目录

1. [现有项目架构概览](#1-现有项目架构概览)
2. [迁移可行性评估](#2-迁移可行性评估)
3. [技术映射对照表](#3-技术映射对照表)
4. [小程序项目结构设计](#4-小程序项目结构设计)
5. [页面拆分与组件规划](#5-页面拆分与组件规划)
6. [API 封装模块设计](#6-api-封装模块设计)
7. [认证与存储方案](#7-认证与存储方案)
8. [关键技术差异与兼容方案](#8-关键技术差异与兼容方案)
9. [分步迁移执行计划](#9-分步迁移执行计划)
10. [风险与限制](#10-风险与限制)

---

## 1. 现有项目架构概览

### 1.1 文件清单（4个文件，纯前端）

| 文件 | 行数 | 功能定位 |
|------|------|----------|
| `index.html` | 104 | 定价落地页：3档订阅方案展示 + 区域定价动态加载 |
| `auth.html` | 161 | 登录/注册页：演示模式（仅前端校验格式） |
| `demo.html` | 383 | **核心体验页**：简历上传解析 / 职位匹配 / AI问答 / 剪贴板粘贴 |
| `test_match.json` | 1 | 后端 API 测试数据样本（用于调试） |

### 1.2 后端 API 依赖（6个接口）

| 接口 | Method | 说明 | 认证 | 配额控制 |
|------|--------|------|------|----------|
| `/v1/region` | GET | 获取区域定价与货币符号 | ❌ | ❌ |
| `/v1/auth/login` | POST | 邮箱+密码登录 | ❌ | ❌ |
| `/v1/auth/register` | POST | 邮箱+密码注册 | ❌ | ❌ |
| `/v1/jobs/match` | POST | 简历→职位匹配 | ✅ Bearer | ✅ 403/429 |
| `/v1/ask` | POST | AI问答统一入口 | ✅ Bearer | ✅ 403/429 |
| `/v1/documents/convert` | POST (multipart) | 文件上传解析（PDF/Word→文本） | ✅ Bearer | ✅ 403/429 |

### 1.3 数据流

```
用户 → 注册/登录 → 获取 token → [选择简历] → 职位匹配 / AI问答
                                           ↑
                              上传简历文件 → 解析为文本
```

### 1.4 客户端存储

| 存储 | Key | 用途 | 作用域 |
|------|------|------|--------|
| localStorage | `tatha_demo_token` | 认证 Bearer Token | 持久 |
| sessionStorage | `tatha_resume_list` | 已解析简历列表 | 会话 |

### 1.5 视觉设计系统（CSS 变量）

```css
--bg:     #0f0f12   /* 深色背景 */
--card:   #1a1a20   /* 卡片背景 */
--border: #2a2a35   /* 边框 */
--text:   #eaeaf0   /* 主文字 */
--muted:  #8b8b96   /* 次要文字 */
--accent: #6366f1   /* 强调色（indigo-500） */
```

---

## 2. 迁移可行性评估

### ✅ 结论：完全可行，无阻碍因素

| 评估维度 | 结论 | 说明 |
|----------|------|------|
| 技术栈兼容性 | ✅ | HTML→WXML，CSS→WXSS，JS→小程序JS，均有成熟映射路径 |
| 后端API复用 | ✅ | 6个 HTTP JSON API 通过 `wx.request()` 直接调用，无需任何改动 |
| 认证方案 | ✅ | Bearer Token 通过 `wx.getStorageSync()` 存储，完全兼容 |
| 文件上传 | ✅ | `wx.chooseMessageFile()` + `wx.uploadFile()` 替代 FormData fetch |
| UI复杂度 | ✅ | 3个页面均为简单表单+列表，无复杂动画或Canvas |
| 第三方依赖 | ✅ | 零外部依赖（无 npm 包），纯 vanilla JS |
| 域名要求 | ⚠️ | 后端 `http://localhost:8010` 需替换为已备案 HTTPS 域名，且在微信后台配置 |

### 唯一硬性要求
- 后端 API 必须部署到**已备案域名** + **HTTPS**（微信小程序强制要求）
- 域名需在微信公众平台「开发 → 开发管理 → 开发设置 → 服务器域名」中配置 `request合法域名` 和 `uploadFile合法域名`

---

## 3. 技术映射对照表

### 3.1 HTML → WXML 映射

| Web (HTML) | 微信小程序 (WXML) | 备注 |
|------------|-------------------|------|
| `<div>` | `<view>` | 通用容器 |
| `<span>`, `<p>`, `<h1>`-`<h6>` | `<text>` 或 `<view>` | text 组件支持内联选择 |
| `<a href="...">` | `<navigator url="...">` 或 `bindtap`+`wx.navigateTo()` | 页面跳转 |
| `<input type="text">` | `<input type="text">` | 基本相同 |
| `<input type="email">` | `<input type="text">` | 小程序不支持 email 类型，需手动校验 |
| `<input type="password">` | `<input type="password">` | 相同 |
| `<input type="file">` | 无对应组件 | 使用 `wx.chooseMessageFile()` 触发 |
| `<textarea>` | `<textarea>` | 基本相同 |
| `<button>` | `<button>` | 小程序 button 有更多预设样式，需重置 |
| `<form>` | `<form bindsubmit="...">` | submit 事件名不同 |
| `style="display:none"` | `wx:if="{{false}}"` | 条件渲染，性能更好 |
| `style="display:flex"` | 使用 flex 布局，与 Web 相同 | |
| `v-for` / `map()` | `wx:for="{{list}}"` | 列表渲染 |
| `innerHTML` | ❌ 不支持 | 需使用 `rich-text` 组件或 `wx:if`+数据绑定 |
| `<style>` 内嵌 | ❌ 不支持 | 样式单独在 `.wxss` 文件，或内联 `<view style="...">` |

### 3.2 CSS → WXSS 映射

| Web (CSS) | 微信小程序 (WXSS) | 备注 |
|-----------|-------------------|------|
| `background` 渐变 | ❌ 不支持 CSS gradient | 需使用背景图片代替 |
| `filter` | ❌ 不支持 | button hover 的 brightness 效果需移除或改用其他方式 |
| `* { box-sizing: ... }` | ✅ 支持 | |
| CSS 变量 (`:root` / `var()`) | ⚠️ 基础库 2.14.0+ 支持 | 建议使用，兼容性 ok |
| `position: fixed` | ✅ 支持 | |
| `@media` 查询 | ⚠️ 支持 | 但通常用 `wx.getSystemInfoSync()` 做响应式 |
| 字号 `rem` | ⚠️ 支持但推荐 `rpx` | 小程序以 750rpx 为设计稿宽度 |
| `:hover` 伪类 | ❌ 不支持 | 微信无 hover 概念 |
| `::before` / `::after` | ❌ 不支持 | 需用额外 `<view>` 代替或使用图片 |
| `overflow: scroll` | ✅ 使用 `<scroll-view>` 更好 | |
| `word-break: break-all` | ✅ 支持 | |
| `text-overflow: ellipsis` | ✅ 支持 | |
| `gap` (flex gap) | ⚠️ 基础库 2.25.3+ | 建议用 margin 备选方案 |

### 3.3 JavaScript API 映射

| Web API | 微信小程序 API | 备注 |
|---------|---------------|------|
| `fetch(url, options)` | `wx.request({ url, method, data, header })` | Promise 风格，需封装 |
| `r.json()` | 响应直接是 JSON（自动解析） | `res.data` 获取 |
| `FormData` + `fetch` | `wx.uploadFile({ url, filePath, formData })` | 文件上传 |
| `localStorage.setItem(k, v)` | `wx.setStorageSync(k, v)` | 同步版本 |
| `localStorage.getItem(k)` | `wx.getStorageSync(k)` | 同步版本 |
| `localStorage.removeItem(k)` | `wx.removeStorageSync(k)` | |
| `sessionStorage.*` | 用 `wx.setStorageSync()` 替代，无会话概念 | 需在 `onUnload` 中手动清理 |
| `navigator.clipboard.readText()` | `wx.getClipboardData({ success })` | |
| `navigator.clipboard.writeText()` | `wx.setClipboardData({ data })` | |
| `window.location.href = url` | `wx.redirectTo({ url })` | 替换当前页 |
| `window.location.replace(url)` | `wx.redirectTo({ url })` | |
| `window.open(url, '_blank')` | `wx.navigateTo({ url })` | 保留当前页 |
| `history.back()` | `wx.navigateBack()` | |
| `document.querySelector(sel)` | `this.selectComponent()` 或数据驱动 | 不可直接操作 DOM |
| `document.getElementById(id)` | 通过 `data-*` + 事件对象 | 不可直接操作 DOM |
| `el.textContent = ...` | `this.setData({ key: ... })` | 数据驱动渲染 |
| `el.classList.add/remove` | `this.setData()` 绑定 class 变量 | |
| `input.files[0]` | `wx.chooseMessageFile({ success: res => res.tempFiles[0] })` | |
| `new FileReader()` | ❌ 不必要 | `wx.chooseMessageFile` 已返回文件信息 |
| `el.focus()` | ❌ 不支持编程式聚焦 | 可通过 `focus` 属性控制 input 聚焦 |
| `alert()` | `wx.showToast()` 或 `wx.showModal()` | |
| `setTimeout(cb, ms)` | ✅ 支持 | 完全相同 |
| `JSON.parse/stringify` | ✅ 支持 | 完全相同 |
| `Array.map/filter/find` | ✅ 支持 | 完全相同 |
| `Promise` | ✅ 支持 | 完全相同 |
| `try/catch` | ✅ 支持 | 完全相同 |
| ES6 语法 | ✅ 支持（基础库 2.10+） | let/const/箭头函数/模板字符串 |

### 3.4 事件映射

| Web 事件 | 小程序事件 | 备注 |
|----------|-----------|------|
| `onclick` | `bindtap` | 点击事件 |
| `onsubmit` (form) | `bindsubmit` | 表单提交 |
| `onchange` (input) | `bindinput` | 输入变化 |
| `onchange` (file input) | `bindtap` 触发的选择回调 | 非原生事件 |
| `onload` | `onLoad()` 生命周期 | |
| `onunload` | `onUnload()` 生命周期 | |
| `onshow` | `onShow()` 生命周期 | |
| `DOMContentLoaded` | `onReady()` 生命周期 | |

---

## 4. 小程序项目结构设计

```
tatha-miniapp/
│
├── app.json                  # 全局配置（页面注册、窗口样式、tabBar）
├── app.js                    # 全局逻辑（token管理、全局数据）
├── app.wxss                  # 全局样式（暗色主题变量、共用class）
├── project.config.json       # 项目配置（appid、编译设置）
├── sitemap.json              # 搜索索引配置（默认即可）
│
├── utils/
│   ├── api.js                # wx.request 封装（含认证拦截、限流处理）
│   └── constants.js          # 常量（API基址、存储key名、配色等）
│
├── pages/
│   ├── index/                # 定价落地页（原 index.html）
│   │   ├── index.js
│   │   ├── index.json
│   │   ├── index.wxml
│   │   └── index.wxss
│   │
│   ├── auth/                 # 登录/注册页（原 auth.html）
│   │   ├── auth.js
│   │   ├── auth.json
│   │   ├── auth.wxml
│   │   └── auth.wxss
│   │
│   ├── demo/                 # 核心体验页（原 demo.html）
│   │   ├── demo.js
│   │   ├── demo.json
│   │   ├── demo.wxml
│   │   └── demo.wxss
│   │
│   └── match-detail/         # （可选）匹配详情页
│       ├── match-detail.js
│       ├── match-detail.json
│       ├── match-detail.wxml
│       └── match-detail.wxss
│
├── components/               # （可选）可复用组件
│   ├── pricing-card/         # 定价卡片组件
│   │   ├── pricing-card.js
│   │   ├── pricing-card.json
│   │   ├── pricing-card.wxml
│   │   └── pricing-card.wxss
│   │
│   ├── match-card/           # 匹配结果卡片组件
│   │   ├── match-card.js
│   │   ├── match-card.json
│   │   ├── match-card.wxml
│   │   └── match-card.wxss
│   │
│   └── upgrade-banner/       # 升级提示横幅组件
│       ├── upgrade-banner.js
│       ├── upgrade-banner.json
│       ├── upgrade-banner.wxml
│       └── upgrade-banner.wxss
│
└── images/                   # 图标等静态资源（如需要）
```

---

## 5. 页面拆分与组件规划

### 5.1 页面：`pages/index/index`（定价落地页）

**对应原文件：** `index.html`

| WXML 结构 | 说明 |
|-----------|------|
| `<view class="hero">` | 标题 + 描述 |
| `<view class="pricing">` | 3个定价卡片（`wx:for`） |
| → `<pricing-card>` 组件 × 3 | 每个卡片独立组件 |
| `<view class="footer-nav">` | 底部链接（登录 / 演示） |

**数据绑定：**
- `pricingCards[]`：从 `/v1/region` API 动态获取
- `currencySymbol`：货币符号
- `regionPrices`：各地区价格映射

**关键改动：**
- `window.location.href` → `wx.navigateTo({ url: '/pages/auth/auth' })`
- CSS `:hover` → 使用 `hover-class` 属性

---

### 5.2 页面：`pages/auth/auth`（登录/注册）

**对应原文件：** `auth.html`

| WXML 结构 | 说明 |
|-----------|------|
| Tab 切换（登录/注册） | 2个按钮切换面板 |
| 登录表单 | email + password + 提交 |
| 注册表单 | email + password + 确认密码 + 提交 |
| 底部链接 | 返回定价页 / 进入体验 |

**数据绑定：**
- `activeTab: 'login' | 'register'`
- 各表单字段 `email`, `password`, `password2`
- `msg`: 提示信息文本与类型

**关键改动：**
- `onsubmit` → `bindsubmit`
- `fetch()` → 封装的 `api.post()`
- `localStorage.setItem()` → `wx.setStorageSync()`

---

### 5.3 页面：`pages/demo/demo`（核心体验页）

**对应原文件：** `demo.html`（最复杂页面，383行）

| WXML 结构 | 说明 |
|-----------|------|
| 顶部导航 | 标题 + 退出登录链接 |
| 3个Tab | 简历 / 职位匹配 / AI问答 |
| 简历面板 | 文件选择 + 上传解析 + 简历列表 |
| 匹配面板 | 简历输入 + 粘贴按钮 + 匹配按钮 + 结果列表 |
| 问答面板 | 消息输入 + 简历输入 + 发送按钮 + 结果展示 |

**数据绑定（大量）：**
- `activeTab`: 当前面板
- `resumeText`, `resumeAskText`: 简历输入内容
- `message`: AI问答输入
- `resumeList[]`: 已解析简历列表
- `selectedResumeId`: 当前选中简历
- `matchResults[]`: 匹配结果
- `askResult`: 问答结果
- `loading*`: 各操作加载状态
- `msg*`: 各操作提示信息

**关键改动（最多）：**
- `innerHTML` 渲染匹配卡片 → `<match-card>` 组件 + `wx:for`
- 文件上传 `FormData + fetch` → `wx.chooseMessageFile()` + `wx.uploadFile()`
- 剪贴板读取 `navigator.clipboard.readText()` → `wx.getClipboardData()`
- `document.querySelectorAll` + `forEach` 绑定 → 数据驱动 + 事件代理
- `el.style.display` 控制面板切换 → `wx:if`

---

### 5.4 组件拆分建议

| 组件名 | 复用场景 | 属性 (Properties) |
|--------|----------|-------------------|
| `pricing-card` | `index` 页 | `tag`, `price`, `currency`, `desc`, `features[]`, `buttonText`, `isPopular`, `onTap` |
| `match-card` | `demo` 页匹配结果 | `title`, `company`, `location`, `url`, `overall`, `summary` |
| `upgrade-banner` | `demo` 页限额提示 | `message`, `linkText` |

---

## 6. API 封装模块设计

### 6.1 `utils/constants.js`

```javascript
// API 基址（上线前替换为正式域名）
const API_BASE = 'https://your-domain.com';

// 存储 Key
const STORAGE_TOKEN = 'tatha_token';
const STORAGE_RESUME_LIST = 'tatha_resume_list';

module.exports = {
  API_BASE,
  STORAGE_TOKEN,
  STORAGE_RESUME_LIST
};
```

### 6.2 `utils/api.js` 设计

提供统一封装的请求方法，处理以下横切关注点：

```
┌─────────────┐
│   请求层     │
├─────────────┤
│ 1. 自动注入 Authorization: Bearer <token>
│ 2. 401 → 自动清除token，跳转登录页
│ 3. 403/429 → 返回配额错误信息（不跳转）
│ 4. 统一错误处理与 toast 提示
│ 5. Promise 风格返回
└─────────────┘
```

**导出方法：**

| 方法 | 签名 | 对应Web |
|------|------|---------|
| `get(url)` | `api.get('/v1/region') → Promise<data>` | `fetch(url)` |
| `post(url, data)` | `api.post('/v1/auth/login', { email, password }) → Promise<data>` | `fetch(url, { method:'POST', body: JSON.stringify(data) })` |
| `upload(url, filePath, formData)` | `api.upload('/v1/documents/convert', tempFilePath, { document_type: 'resume' }) → Promise<data>` | `fetch(url, { method:'POST', body: formData })` |

**核心实现逻辑：**

```javascript
// utils/api.js 伪代码
const { API_BASE, STORAGE_TOKEN } = require('./constants');

function request(options) {
  return new Promise((resolve, reject) => {
    const token = wx.getStorageSync(STORAGE_TOKEN);
    const header = { 'Content-Type': 'application/json' };
    if (token) header['Authorization'] = 'Bearer ' + token;

    wx.request({
      url: API_BASE + options.url,
      method: options.method || 'GET',
      data: options.data,
      header,
      success(res) {
        if (res.statusCode === 401) {
          // 清除 token，跳转登录
          wx.removeStorageSync(STORAGE_TOKEN);
          wx.redirectTo({ url: '/pages/auth/auth' });
          reject(new Error('unauthorized'));
          return;
        }
        if (res.statusCode === 403 || res.statusCode === 429) {
          // 配额限制，将 detail 一并返回给调用方
          resolve({ _quotaError: true, detail: res.data.detail });
          return;
        }
        resolve(res.data);
      },
      fail(err) {
        reject(err);
      }
    });
  });
}

function upload(url, filePath, formData) {
  // 使用 wx.uploadFile，自动带 token
}

module.exports = { get, post, upload };
```

---

## 7. 认证与存储方案

### 7.1 Token 管理

```
┌─────────────────────────────────────┐
│            app.js（全局）            │
├─────────────────────────────────────┤
│ onLaunch() {                        │
│   // 检查 token 是否存在            │
│   const token = wx.getStorageSync(  │
│     'tatha_token'                   │
│   );                                │
│   if (!token) {                     │
│     // 无 token 跳转定价页          │
│     wx.redirectTo({                 │
│       url: '/pages/index/index'     │
│     });                             │
│   }                                 │
│ }                                   │
└─────────────────────────────────────┘
```

### 7.2 页面鉴权守卫

**当前 Web 端的做法：** `demo.html` 中 IIFE 检查 token，无则跳转

**小程序端做法：**
- 在 `app.js` 全局 `onLaunch` 做首次检查
- 在 `demo.js` 的 `onLoad` 再次检查（防止直接扫码进入）
- 方案一：每个需要登录的页面在 `onLoad` 中检查
- 方案二：使用 `wx.navigateTo` 的 `events` 或全局导航拦截

**建议采用方案一**（简单可靠）：

```javascript
// pages/demo/demo.js
onLoad() {
  const token = wx.getStorageSync('tatha_token');
  if (!token) {
    wx.redirectTo({ url: '/pages/index/index' });
    return;
  }
  this.loadData();
}
```

### 7.3 简历列表存储

| Web 端 | 小程序端 |
|--------|----------|
| `sessionStorage` | 存储到全局变量 `app.globalData.resumeList` |
| `JSON.parse( sessionStorage.getItem(...) )` | `app.globalData.resumeList` 已是对象，无需序列化 |
| Session 隔离 | 小程序无 tab 概念，数据跨页面通过 `app.globalData` 或 `wx.setStorageSync` 传递 |

**建议：** 简历列表改为使用 `wx.setStorageSync('tatha_resume_list', list)` 做持久化存储，体验更好。

---

## 8. 关键技术差异与兼容方案

### 8.1 `innerHTML` 替代方案

**问题：** 原项目广泛使用 `el.innerHTML = htmlString` 动态渲染匹配卡片、升级横幅。微信小程序不支持 `innerHTML`。

**方案：** 使用数据绑定 + `rich-text` 组件 + 组件化

| 场景 | Web 做法 | 小程序做法 |
|------|----------|-----------|
| 渲染匹配卡片列表 | `out('out-match', renderMatches(data))` - 拼接 HTML 字符串 | `this.setData({ matches: data.matches })` + `<match-card wx:for>` |
| 渲染升级提示 | `out('out-match', renderUpgradeBanner(msg))` | `<upgrade-banner wx:if="{{showUpgrade}}" message="{{upgradeMsg}}">` |
| 渲染问答结果 | `out('out-ask', renderAsk(data))` | `this.setData({ askResult: data })` + 条件渲染 |

### 8.2 文件上传替代

**Web 端流程：**
```
<input type="file" onchange> → file.files[0] → new FormData() → fetch()
```

**小程序端流程：**
```javascript
// 1. 选择文件
wx.chooseMessageFile({
  count: 1,
  type: 'file',
  extension: ['pdf', 'doc', 'docx', 'txt', 'md'],
  success(res) {
    const file = res.tempFiles[0];
    // 2. 上传文件
    api.upload('/v1/documents/convert', file.path, {
      document_type: 'resume'
    }).then(data => {
      // 解析成功
    });
  }
});
```

### 8.3 DOM 查询与操作替代

**Web 端：** `document.querySelectorAll('.btn-use-resume').forEach(btn => btn.onclick = ...)`

**小程序端：** 使用数据驱动，通过 `wx:for` 渲染列表，`data-id` 绑定到 `bindtap` 事件中获取

```xml
<!-- WXML -->
<view wx:for="{{resumeList}}" wx:key="id" bindtap="onSelectResume" data-id="{{item.id}}">
  <text>{{item.filename}}</text>
</view>
```

```javascript
// JS
onSelectResume(e) {
  const id = e.currentTarget.dataset.id;
  const item = this.data.resumeList.find(r => r.id === id);
  // ...
}
```

### 8.4 CSS 伪元素 `::before` / `::after` 替代

**Web 端：** 列表项前的圆点 → `li::before { content: "•"; }`

**小程序端：** 使用 `<text>` 代替或内联

```xml
<!-- 方案A：额外 text 组件 -->
<view class="feature-item">
  <text class="bullet">•</text>
  <text>{{feature}}</text>
</view>

<!-- 方案B：用 Unicode 前缀（推荐，更简单） -->
<text wx:for="{{features}}" wx:key="*this">• {{item}}</text>
```

### 8.5 剪贴板操作替代

| Web 端 | 小程序端 |
|--------|----------|
| `navigator.clipboard.readText().then(text => el.value = text)` | `wx.getClipboardData({ success: res => this.setData({ resumeText: res.data }) })` |

**注意：** 微信剪贴板 API 需要用户授权（首次使用时弹出授权框）

### 8.6 人民币符号 ¥ 显示

微信小程序原生支持 UTF-8，`¥` 符号直接使用即可，无需特殊处理。

### 8.7 暗色主题支持

微信小程序支持 CSS 变量（基础库 2.14.0+），可直接将项目的 `:root` 变量体系迁移到 `app.wxss`：

```css
/* app.wxss */
page {
  --bg: #0f0f12;
  --card: #1a1a20;
  --border: #2a2a35;
  --text: #eaeaf0;
  --muted: #8b8b96;
  --accent: #6366f1;
  background: var(--bg);
  color: var(--text);
  font-family: -apple-system, sans-serif;
}
```

**额外福利：** 小程序天然支持 DarkMode（`app.json` 中配置 `"darkmode": true`），可以跟随系统自动切换主题。

---

## 9. 分步迁移执行计划

### 第一阶段：基础框架搭建（预计 1-2 小时）

- [ ] 1.1 创建小程序项目（微信开发者工具新建项目）
- [ ] 1.2 配置 `project.config.json`（appid、编译设置）
- [ ] 1.3 配置 `app.json`（注册页面路径、窗口样式、navigationBar）
- [ ] 1.4 编写 `app.js`（全局 token 检查逻辑）
- [ ] 1.5 编写 `app.wxss`（全局暗色主题变量 + 基础重置样式）
- [ ] 1.6 创建 `utils/constants.js` 和 `utils/api.js`

### 第二阶段：定价页迁移 `pages/index/index`（预计 30 分钟）

- [ ] 2.1 `index.wxml`：Hero + 定价卡片区域 + 底部导航
- [ ] 2.2 `index.wxss`：暗色卡片样式、响应式 grid（用 `flex-wrap` + `rpx`）
- [ ] 2.3 `index.js`：调用 `/v1/region` 获取动态定价数据
- [ ] 2.4 创建 `components/pricing-card` 组件

### 第三阶段：认证页迁移 `pages/auth/auth`（预计 30 分钟）

- [ ] 3.1 `auth.wxml`：Tab 切换 + 登录表单 + 注册表单
- [ ] 3.2 `auth.wxss`：表单样式（与 Web 版一致）
- [ ] 3.3 `auth.js`：登录/注册表单提交逻辑，token 存储，成功后跳转 demo 页

### 第四阶段：核心体验页迁移 `pages/demo/demo`（预计 2-3 小时）

- [ ] 4.1 `demo.wxml`：3个Tab面板 + 表单 + 结果展示区
- [ ] 4.2 `demo.wxss`：面板样式、匹配卡片样式、表单样式
- [ ] 4.3 `demo.js`：
  - Tab 切换逻辑
  - 文件上传（`wx.chooseMessageFile` + `wx.uploadFile`）
  - 简历列表管理（数据驱动渲染）
  - 职位匹配请求
  - AI 问答请求
  - 剪贴板粘贴
  - 限额处理与升级横幅展示
- [ ] 4.4 创建 `components/match-card` 组件
- [ ] 4.5 创建 `components/upgrade-banner` 组件

### 第五阶段：测试与调优（预计 1 小时）

- [ ] 5.1 在微信开发者工具中全流程测试（注册→登录→上传简历→匹配→问答）
- [ ] 5.2 测试 401 过期处理（手动清除 storage 后验证跳转）
- [ ] 5.3 测试 403/429 限额拦截（升级提示是否正确展示）
- [ ] 5.4 真机预览测试（扫码体验）
- [ ] 5.5 修复 WXSS 兼容性问题（按钮样式、间距、rpx 适配）
- [ ] 5.6 性能检查（setData 调用量、包大小）

### 可选增强

- [ ] 添加 `wx.showLoading()` / `wx.hideLoading()` 替代 `loading...` 文字
- [ ] 使用 `wx.showToast()` 替代内联成功/错误消息
- [ ] 匹配结果列表使用分页加载（`onReachBottom`）
- [ ] 添加骨架屏加载动画
- [ ] 配置 `darkmode: true` 实现跟随系统深色模式
- [ ] 添加分享功能（`onShareAppMessage`）

---

## 10. 风险与限制

| 风险/限制 | 影响 | 缓解措施 |
|-----------|------|----------|
| **后端域名** | 小程序必须使用已备案 HTTPS 域名 | 提前准备域名并完成备案，开发阶段可用「不校验合法域名」选项 |
| **文件上传大小** | 小程序单次上传限制 10MB | 对于简历文件完全足够，在 `wx.chooseMessageFile` 中提示用户 |
| **包大小限制** | 小程序主包限制 2MB | 当前项目无图片，纯代码约 50KB，完全安全 |
| **innerHTML 不支持** | 需重构所有 HTML 字符串拼接逻辑 | 改用数据绑定+组件化，工作量可控 |
| **CSS 兼容性** | `:hover`、`::before`、`filter` 等不可用 | 使用 `hover-class`、额外组件、移除不必要效果 |
| **剪贴板授权** | 首次使用需用户授权 | 在「粘贴」按钮处友好提示 |
| **分享到朋友圈** | 小程序默认不支持朋友圈分享 | 如需要可启用（需审核） |
| **WebSocket** | 当前项目无此需求 | 如未来需要，用 `wx.connectSocket()` |

---

## 附录：小程序配置参考

### `app.json` 示例

```json
{
  "pages": [
    "pages/index/index",
    "pages/auth/auth",
    "pages/demo/demo"
  ],
  "window": {
    "navigationBarBackgroundColor": "#0f0f12",
    "navigationBarTitleText": "Tatha",
    "navigationBarTextStyle": "white",
    "backgroundColor": "#0f0f12"
  },
  "darkmode": true,
  "themeLocation": "theme.json",
  "sitemapLocation": "sitemap.json"
}
```

### `project.config.json` 关键配置

```json
{
  "appid": "你的小程序AppID",
  "setting": {
    "urlCheck": false,
    "es6": true,
    "enhance": true,
    "postcss": true,
    "minified": true
  }
}
```

### 微信后台服务器域名配置

```
request合法域名：https://your-api-domain.com
uploadFile合法域名：https://your-api-domain.com
```

---

> **总结：** 本项目从 Web 迁移到微信小程序**零架构障碍**，后端接口完全复用，前端工作量为 3 个页面的 UI 重写 + DOM 操作替换为数据驱动，预计总工时 4-6 小时即可完成完整可用的迁移。