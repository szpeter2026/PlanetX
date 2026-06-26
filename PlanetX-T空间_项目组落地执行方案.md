# PlanetX-T空间 项目组落地执行方案

> 版本：v1.0  
> 日期：2026-06-25  
> 用途：可直接分发给项目组成员，每个成员拿到自己负责的 Task Pack 即可独立开工

---

## 一、项目组最优配置

### 1.1 人员组成（3-4 人，最优）

| 角色 | 代号 | 人数 | 核心技能 | 负责模块 |
|------|------|------|----------|----------|
| **后端工程师** | BE | 1 人 | Python / FastAPI / PostgreSQL / pgvector / LiteLLM | looma-zervi 后端扩展（新API + 多租户） |
| **前端工程师（小程序）** | FE-MP | 1 人 | 微信小程序 / WXML / WXSS / JavaScript | PlanetX C端小程序开发 |
| **前端工程师（Web）** | FE-Web | 1 人 | React/Vue + 组件库 / TypeScript / Web 管理台 | T空间 B端 Web 管理台 |
| **全栈/DevOps 兼 PM** | FS | 1 人(可兼任) | Docker / Nginx / HTTPS部署 / Supabase / API对接 / 项目协调 | 基础设施 + 双向管道 + 项目推进 |

> **最低配置（3人）：** BE + FE-MP + FS（FS 兼 FE-Web，Web 管理台优先级可适当延后）  
> **推荐配置（4人）：** BE + FE-MP + FE-Web + FS

### 1.2 技术栈选型

| 模块 | 推荐技术栈 | 理由 |
|------|-----------|------|
| **后端** | Python 3.11+ / FastAPI / SQLAlchemy / pgvector / LiteLLM / Supabase Auth | 与 looma-zervi 现有栈完全一致，零学习成本 |
| **PlanetX 小程序** | 原生微信小程序 + 微信云开发（可选） | 无需额外框架，按迁移方案直接开工 |
| **T空间 Web 管理台** | React 18 + TypeScript + Ant Design / shadcn/ui + Vite | 组件库成熟，企业后台首选 |
| **基础设施** | Docker Compose / Nginx / Let's Encrypt / GitHub Actions | 现有 looma-zervi 已可用 |

---

## 二、项目里程碑与时间线

```
Week 1-2:  Sprint 1 — 基础设施 + MVP 闭环
Week 3-4:  Sprint 2 — AI 模拟面试 + 星际人格
Week 5-6:  Sprint 3 — 招聘管道 + 知识库
Week 7-8:  Sprint 4 — 生态飞轮 + 上线
```

| 里程碑 | 截止周 | 交付物 | 验收标准 |
|--------|--------|--------|----------|
| M1 | W2 | HTTPS 部署 + 企业入驻 API | Postman 可调通，返回 200 |
| M2 | W2 | PlanetX 小程序 MVP | 微信开发者工具可完整走通注册→匹配→结果 |
| M3 | W3 | T空间 Web 管理台 MVP | HR 可登录、创建JD、查看匹配候选人 |
| M4 | W4 | AI 模拟面试上线 | 输入 JD → 生成题目 → 回答 → 获得评分 |
| M5 | W6 | 招聘管道 + 企业知识库 | 候选人状态流转 + 文档导入RAG问答 |
| M6 | W8 | 双向飞轮 + 内测上线 | C端用户匹配到B端真实JD |

---

## 三、Task Pack 分配

---

### 📦 Task Pack A：后端工程师（BE）
**负责：looma-zervi 后端扩展**
**预计工时：6 周（含测试）**

---

#### A.1 Sprint 1（W1-W2）：企业入驻 + JD 管理 + 多租户

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| A.1.1 | **多租户数据库设计** | 4h | `src/db/models.py` 新增 `Tenant`, `Enterprise`, `JobPosition` 表 |
| A.1.2 | **企业入驻 API** | 4h | `src/api/routes/enterprise.py` — `POST /v1/enterprise/register`, `GET /v1/enterprise/{id}`, `PUT /v1/enterprise/{id}` |
| A.1.3 | **JD 管理 API** | 4h | `src/api/routes/jobs_manage.py` — `POST /v1/jobs/manage`, `PUT /v1/jobs/manage/{id}`, `DELETE /v1/jobs/manage/{id}`, `GET /v1/jobs/manage` |
| A.1.4 | **多租户中间件** | 3h | `src/api/middleware/tenant.py` — 从 JWT 提取 tenant_id，注入请求上下文 |
| A.1.5 | **内测申请 API** | 2h | `src/api/routes/beta.py` — `POST /v1/beta/apply`, `GET /v1/beta/status` |
| A.1.6 | **企业数据隔离校验** | 3h | 为 `/v1/jobs/match`, `/v1/documents/*` 添加 tenant_id 过滤 |
| A.1.7 | **Sprint 1 集成测试** | 4h | `tests/test_enterprise.py`, `tests/test_jobs_manage.py` |

**A.1 验收标准：**
- [ ] `POST /v1/enterprise/register` 返回企业 ID + admin token
- [ ] 企业 A 创建 JD 后，企业 B 无法看到
- [ ] JWT 未携带 tenant 时返回 403

---

#### A.2 Sprint 2（W3-W4）：AI 模拟面试 + 统一用户身份

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| A.2.1 | **用户角色扩展** | 3h | `src/api/auth.py` 支持 `role: personal / enterprise_admin / enterprise_hr` |
| A.2.2 | **面试题生成引擎** | 6h | `src/agents/interview_agent.py` — 基于 JD + 简历，用 LiteLLM 生成 5-8 道面试题 |
| A.2.3 | **面试对话 API** | 4h | `src/api/routes/interview.py` — `POST /v1/interview/generate`, `POST /v1/interview/chat` |
| A.2.4 | **面试评分 API** | 4h | 面试结束后 LLM 多维评分（表达能力、技术深度、逻辑性） |
| A.2.5 | **面试历史记录** | 2h | `GET /v1/interview/history` |
| A.2.6 | **统一认证 API 调整** | 2h | `/v1/auth/*` 返回 `role` 字段 |
| A.2.7 | **Sprint 2 集成测试** | 3h | `tests/test_interview.py` |

**A.2 验收标准：**
- [ ] 输入 JD + 简历 → 返回 5+ 道针对性面试题
- [ ] 连续对话 5 轮 → LLM 评分 ≥ 3 个维度
- [ ] personal / enterprise_hr 登录后权限正确隔离

---

#### A.3 Sprint 3（W5-W6）：招聘管道 + 仪表盘

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| A.3.1 | **招聘管道模型** | 4h | Pipeline 表：`screening → interview → offer → hired/rejected` 状态机 |
| A.3.2 | **管道流转 API** | 3h | `PUT /v1/pipeline/{match_id}/stage` 阶段变更 |
| A.3.3 | **候选人看板 API** | 3h | `GET /v1/pipeline/board?enterprise_id=X` 聚合数据 |
| A.3.4 | **企业仪表盘 API** | 3h | `GET /v1/dashboard/enterprise` — 活跃JD数、待处理候选人、面试通过率 |
| A.3.5 | **用户仪表盘 API** | 2h | `GET /v1/dashboard/user` — 匹配进度、面试邀请数、技能图谱 |
| A.3.6 | **报告生成接口化** | 3h | 将 `report_gen.py` 能力暴露为 `POST /v1/reports/generate` |

**A.3 验收标准：**
- [ ] 候选人从 screening → offer 状态流转完整
- [ ] 企业仪表盘返回 >= 5 个指标
- [ ] 用户仪表盘返回匹配进度 + 面试统计

---

#### A.4 Sprint 4（W7-W8）：企业知识库 + 双向管道 + 安全加固

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| A.4.1 | **企业知识库 RAG API** | 6h | `POST /v1/knowledge/upload`, `POST /v1/knowledge/query`, `DELETE /v1/knowledge/{doc_id}` |
| A.4.2 | **知识库多租户隔离** | 3h | 确保企业 A 查询只返回自己的文档 |
| A.4.3 | **双向匹配管道** | 6h | `POST /v1/pipeline/auto-match` — 求职者简历自动匹配所有企业公开 JD |
| A.4.4 | **求职者主动投递** | 3h | `POST /v1/jobs/{job_id}/apply` — 求职者看到 JD 后投递 |
| A.4.5 | **数据脱敏服务** | 3h | 求职者投递时自动隐藏手机/邮箱，企业看到脱敏版本 |
| A.4.6 | **API 限流加固** | 2h | 为所有企业 API 添加 rate limit |
| A.4.7 | **全线 Smoke 测试** | 5h | 覆盖全部 API 端点的端到端测试 |

---

### 📦 Task Pack B：前端工程师-小程序（FE-MP）
**负责：PlanetX 微信小程序**
**预计工时：5 周**

---

#### B.1 Sprint 1（W1-W2）：小程序基础框架 + 核心3页面

> **前置依赖：** 阅读 `Tatha_微信小程序迁移方案.md`（已包含完整技术映射表和页面设计）

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| B.1.1 | **项目初始化** | 2h | `app.json`, `app.js`, `app.wxss`, `project.config.json`, `utils/api.js`, `utils/constants.js` |
| B.1.2 | **定价落地页** | 4h | `pages/index/*` — 三档定价卡 + `/v1/region` 调用 |
| B.1.3 | **登录/注册页** | 4h | `pages/auth/*` — Tab切换 + 表单 + token存储 |
| B.1.4 | **核心体验页-简历面板** | 6h | `pages/demo/demo.wxml` 简历Tab — 文件选择 + 上传解析 + 简历列表 |
| B.1.5 | **核心体验页-匹配面板** | 6h | 匹配Tab — 简历输入 + 粘贴 + 匹配按钮 + 结果卡片 |
| B.1.6 | **核心体验页-问答面板** | 4h | 问答Tab — 消息输入 + AI回答展示 |
| B.1.7 | **match-card 组件** | 3h | `components/match-card/*` — 匹配结果卡片 |
| B.1.8 | **upgrade-banner 组件** | 2h | `components/upgrade-banner/*` — 限额升级提示 |
| B.1.9 | **pricing-card 组件** | 2h | `components/pricing-card/*` — 定价卡片 |
| B.1.10 | **基础联调测试** | 5h | 注册→登录→上传简历→匹配→问答 全流程 |

**B.1 验收标准：**
- [ ] 微信开发者工具中完整走通全流程
- [ ] 401 过期自动跳转登录页
- [ ] 403/429 限额正确展示升级横幅
- [ ] 暗色主题展示正常

---

#### B.2 Sprint 2（W3-W4）：星际人格测试 + 组队建舱

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| B.2.1 | **星际人格测试页** | 8h | `pages/personality/*` — 星际主题 UI + 15-20 题 + 结果展示 |
| B.2.2 | **人格测试结果分享卡片** | 4h | Canvas 生成可分享的星球人格卡片图 |
| B.2.3 | **组队建舱页** | 6h | `pages/team/*` — 创建小队 + 邀请码 + 成员列表 + 解锁状态 |
| B.2.4 | **个人主页/仪表盘** | 6h | `pages/dashboard/*` — 匹配进度、技能图谱、求职状态 |
| B.2.5 | **导航整合** | 3h | TabBar 配置 + 页面间跳转逻辑 |

**B.2 验收标准：**
- [ ] 人格测试完成 → 展示星球类型 + 可分享卡片
- [ ] 创建小队 → 生成邀请码 → 3人加入 → 解锁隐藏功能
- [ ] 个人仪表盘展示 >= 3 个数据指标

---

#### B.3 Sprint 3（W5）：求职追踪 + 情绪陪伴 + 真机测试

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| B.3.1 | **求职进度追踪** | 4h | `pages/tracker/*` — 投递记录、面试日程列表 |
| B.3.2 | **情绪陪伴页** | 4h | `pages/mood/*` — 静音模式、诗词推荐、白噪音播放 |
| B.3.3 | **小程序分享功能** | 2h | `onShareAppMessage` — 分享人格测试/组队邀请 |
| B.3.4 | **真机测试 + Bug修复** | 6h | 扫码体验 + 真机兼容性修复 |
| B.3.5 | **性能优化** | 4h | setData 精简、图片懒加载、分包加载 |

---

### 📦 Task Pack C：前端工程师-Web（FE-Web）
**负责：T空间 B端 Web 管理台**
**预计工时：5 周**

---

#### C.1 Sprint 1（W1-W2）：项目初始化 + 登录 + 企业入驻

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| C.1.1 | **项目脚手架** | 3h | Vite + React 18 + TypeScript + Ant Design / shadcn/ui 初始化 |
| C.1.2 | **API 层封装** | 3h | `src/api/client.ts` — axios 封装 + Bearer Token + 401/403 拦截 |
| C.1.3 | **HR 登录页** | 4h | `src/pages/Login.tsx` — 企业账号登录 |
| C.1.4 | **企业入驻页** | 4h | `src/pages/Onboarding.tsx` — 企业信息填写 + 提交审核 |
| C.1.5 | **路由守卫 + 布局框架** | 3h | `src/layouts/DashboardLayout.tsx` — 侧边栏 + 顶栏 + 租户上下文 |

**C.1 验收标准：**
- [ ] HR 登录后看到企业专属仪表盘
- [ ] 入驻表单提交后返回审核状态

---

#### C.2 Sprint 2（W3-W4）：JD 管理 + 候选人看板

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| C.2.1 | **JD 列表页** | 4h | `src/pages/jobs/JobList.tsx` — 表格 + 搜索 + 筛选 |
| C.2.2 | **JD 创建/编辑页** | 6h | `src/pages/jobs/JobEditor.tsx` — 富文本编辑器 + 发布/下架 |
| C.2.3 | **候选人看板** | 8h | `src/pages/pipeline/Kanban.tsx` — 拖拽式看板（screening→interview→offer→hired） |
| C.2.4 | **候选人详情页** | 4h | `src/pages/pipeline/CandidateDetail.tsx` — 简历展示 + 匹配分数 + 阶段变更 |

**C.2 验收标准：**
- [ ] HR 可创建/编辑/发布 JD
- [ ] 看板拖拽变更候选人阶段
- [ ] 候选人详情展示简历 + 匹配分

---

#### C.3 Sprint 3（W5-W6）：AI 面试 + 知识库 + 报告

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| C.3.1 | **AI 模拟面试配置页** | 4h | `src/pages/interview/InterviewSetup.tsx` — 选择JD → 生成面试题 |
| C.3.2 | **面试记录查看页** | 4h | `src/pages/interview/InterviewHistory.tsx` — 历史面试 + 评分详情 |
| C.3.3 | **企业知识库页** | 6h | `src/pages/knowledge/KnowledgeBase.tsx` — 文档上传 + 列表 + RAG 问答 |
| C.3.4 | **企业仪表盘** | 6h | `src/pages/dashboard/Dashboard.tsx` — ECharts/Recharts 图表 + 核心指标卡片 |
| C.3.5 | **分析报告页** | 4h | `src/pages/reports/ReportView.tsx` — 招聘效率 + 渠道分析 |

---

#### C.4 Sprint 4（W7）：内测申请落地页 + 上线

| 编号 | 任务 | 预计工时 | 产出文件 |
|------|------|----------|----------|
| C.4.1 | **内测申请落地页** | 4h | 独立 Landing Page — 对应 T空间海报内容 |
| C.4.2 | **响应式适配** | 3h | 适配 1366px ~ 1920px |
| C.4.3 | **打包上线** | 3h | Vite build + Nginx 部署 + CI/CD 配置 |

---

### 📦 Task Pack D：全栈/DevOps 兼 PM（FS）
**负责：基础设施 + 双向管道 + 项目协调**
**预计工时：6 周**

---

#### D.1 Sprint 1（W1-W2）：基础设施 + HTTPs 部署

| 编号 | 任务 | 预计工时 | 产出 |
|------|------|----------|------|
| D.1.1 | **HTTPS 域名部署 looma-zervi** | 6h | Nginx + Let's Encrypt + `https://api.your-domain.com` 可访问 |
| D.1.2 | **微信后台域名配置** | 2h | request/uploadFile 合法域名配置 |
| D.1.3 | **CI/CD 流水线** | 4h | GitHub Actions — 提交即自动部署 |
| D.1.4 | **Supabase Auth 配置** | 3h | 用户表 + JWT Secret + RLS 策略 |
| D.1.5 | **开发环境 Docker Compose** | 2h | 确保全员 `docker compose up -d` 即可本地开发 |
| D.1.6 | **API 文档更新** | 3h | 将 A.1 新增接口同步到 `docs/api.yaml` |

---

#### D.2 Sprint 2-3（W3-W6）：双向管道 + 数据脱敏 + 项目推进

| 编号 | 任务 | 预计工时 | 产出 |
|------|------|----------|------|
| D.2.1 | **C↔B 双向匹配脚本** | 6h | 定时任务：每晚将新求职者匹配到新 JD，生成匹配结果 |
| D.2.2 | **数据脱敏规则** | 4h | 手机号 `138****1234`、邮箱 `u***@email.com` |
| D.2.3 | **mock 职位数据生成** | 3h | 为 V0 预置 30+ 测试 JD，覆盖 5 个行业 |
| D.2.4 | **项目进度跟踪** | 持续 | 每周同步 + 阻塞项协调 + Jira/Notion 看板 |
| D.2.5 | **内测用户邀请** | 4h | 对接海报中的"前30家免费"用户，收集反馈 |
| D.2.6 | **性能压测** | 4h | `k6` 压测 `/v1/ask` + `/v1/jobs/match`，确保 50 并发不崩 |
| D.2.7 | **组队社交 API** | 4h | `POST /v1/team/create`, `POST /v1/team/join`, `GET /v1/team/{id}` |

---

## 四、Sprint 全景甘特图

```
            W1    W2    W3    W4    W5    W6    W7    W8
BE  A.1    ████████████
BE  A.2                ████████████
BE  A.3                              ████████████
BE  A.4                                            ████████████
FE-MP B.1 ████████████
FE-MP B.2              ████████████
FE-MP B.3                            ██████
FE-Web C.1████████████
FE-Web C.2            ████████████
FE-Web C.3                          ████████████
FE-Web C.4                                        ████
FS  D.1   ████████████
FS  D.2     ████████████████████████████████████████████
            ↑              ↑              ↑
           M1             M2-M4          M5-M6
```

---

## 五、验收清单（100% 实现标准）

### 5.1 后端验收（BE）

- [ ] 15+ 新 API 端点全部返回 200/201
- [ ] 多租户数据隔离：企业 A 无法访问企业 B 的数据
- [ ] AI 模拟面试：生成题目 → 对话 → 评分 全链路
- [ ] 面试评分含 ≥3 个维度
- [ ] 企业知识库 RAG 查询延迟 < 3s
- [ ] 全线 Smoke 测试通过（覆盖率 ≥ 80%）

### 5.2 PlanetX 小程序验收（FE-MP）

- [ ] 6个页面可正常访问：定价、登录/注册、核心体验、人格测试、组队、仪表盘
- [ ] 全流程：注册 → 登录 → 上传简历 → 职位匹配 → 查看结果
- [ ] 人格测试完成 → 生成分享卡片
- [ ] 组队创建 → 邀请 → 3人成团解锁
- [ ] 暗色主题 + 真机兼容（iOS/Android）
- [ ] 包大小 < 2MB

### 5.3 T空间 Web 管理台验收（FE-Web）

- [ ] HR 可完成：登录 → 企业入驻 → 创建JD → 查看候选人看板 → 拖拽流转
- [ ] 企业知识库：上传 PDF → RAG 问答
- [ ] 仪表盘展示 ≥ 5 个指标
- [ ] 适配 1366px ~ 1920px
- [ ] AI 模拟面试配置 + 查看历史记录

### 5.4 基础设施验收（FS）

- [ ] `https://api.your-domain.com/v1/health` 返回 200
- [ ] 微信小程序可正常调用 HTTPS API
- [ ] 50 并发压测通过（错误率 < 1%）
- [ ] 数据脱敏规则生效
- [ ] CI/CD 自动部署可用
- [ ] API 文档 v1.2.0 已更新

---

## 六、风险预案

| 风险 | 概率 | 影响 | 预案 |
|------|------|------|------|
| 微信小程序审核不通过 | 中 | 高 | 提前 2 周提交审核，准备申诉材料 |
| 后端 HTTPS 域名备案延迟 | 中 | 高 | 开发阶段用「不校验合法域名」选项 |
| AI 模拟面试质量差 | 中 | 中 | 先用预设题库 + LLM 微调，迭代提升 |
| 人员不足（3人 vs 4人） | 高 | 中 | FE-Web 推迟到 Sprint 2 启动，FS 兼 FE-Web |
| BOSS 直聘等真实 API 不合规 | 高 | 低（V0阶段） | V0 用 mock 数据，V1+ 再评估合规方案 |

---

> **这份文档可直接发给项目组成员。每个 Task Pack 独立可执行，依赖关系已在甘特图中标出。建议每日站会同步进度，每周五做 Sprint Review。**