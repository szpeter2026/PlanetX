# PlanetX — 独立源码仓（C 端职业成长伙伴）

> 本仓是主 monorepo 中 `frontend/packages/planetx` + `frontend/packages/shared-core`
> 的**独立镜像仓**，用于独立 CI 测试与 Vercel 部署（app.genz.ltd）。
>
> **同步规则：源码真源在主 monorepo，本仓只读同步，不直接开发。**
> 同步时整体覆盖 `frontend/` 下的包内容并重新生成 lockfile。

## 目录结构

```
frontend/
  pnpm-workspace.yaml      # pnpm 工作区（packages/*）
  tsconfig.base.json       # 共享 TS 配置
  packages/
    planetx/               # PlanetX Web 应用（Vite + React + TS）
    shared-core/           # 共享核心：API client、类型、常量
.github/workflows/ci.yml   # 独立 CI：install → typecheck → build → artifact
```

## 本地开发

```bash
cd frontend
pnpm install
pnpm dev            # PlanetX @ 5173（API 走 /v1 代理）
pnpm typecheck      # 全包类型检查
VITE_API_BASE=https://api.genz.ltd pnpm build   # 生产构建 → packages/planetx/dist
```

## CI

推送到 `main` 或发 PR 自动触发：install → typecheck → build，构建产物以
artifact 形式上传（保留 7 天）。

## Vercel 部署（app.genz.ltd）

`frontend/packages/planetx/vercel.json` 接管安装与构建：

- Vercel 项目 Root Directory 需设为 `frontend/packages/planetx`
- 环境变量 `VITE_API_BASE=https://api.genz.ltd`（必填，勿留空）

详见 `frontend/packages/planetx/VERCEL_APP_GENZ.md`。

## 历史说明

2026-08-31 前，本仓 main 为 app.genz.ltd 的**静态 PWA 切片**（手工同步产物），
现改为完整源码镜像（历史提交仍可回溯切片版本）。
