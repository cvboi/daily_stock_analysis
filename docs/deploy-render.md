# Render 后端部署指南

本文说明如何把 DSA 后端 API 部署到 Render，并与 Netlify 静态 WebUI 打通。

## 部署边界

- Render 运行 FastAPI 后端：`python main.py --serve-only`。
- Netlify 运行 React 静态前端。
- 前端建议通过 Netlify `/api/*` 代理访问 Render 后端，避免跨域 Cookie 问题。

## 1. 准备 Render 服务

推荐使用仓库根目录的 `render.yaml` 创建 Blueprint：

1. 登录 Render。
2. 选择 `New` -> `Blueprint`。
3. 连接本仓库。
4. 确认服务名为 `daily-stock-analysis-api`。
5. 创建服务并等待 Docker 构建完成。

`render.yaml` 会执行以下后端启动命令：

```bash
sh -c "python main.py --serve-only --host 0.0.0.0 --port ${PORT:-8000}"
```

Render 会注入 `PORT`，服务需要监听该端口。

## 2. 配置环境变量

`render.yaml` 已提供基础运行变量：

```env
WEBUI_HOST=0.0.0.0
DATABASE_PATH=/app/data/stock_analysis.db
LOG_DIR=/app/data/logs
ENV_FILE=/app/data/runtime.env
ADMIN_AUTH_ENABLED=true
```

部署后在 Render 控制台补充真实业务配置，例如：

```env
STOCK_LIST=600519,AAPL
REPORT_LANGUAGE=zh
ANSPIRE_API_KEYS=...
AIHUBMIX_KEY=...
OPENAI_API_KEY=...
OPENAI_BASE_URL=...
```

不要把密钥写入仓库文件。没有配置可用 LLM 或新闻检索密钥时，部分分析能力会不可用或降级。

## 3. 验证后端

服务上线后，Render 会给出类似下面的地址：

```text
https://daily-stock-analysis-api.onrender.com
```

先访问健康检查：

```text
https://daily-stock-analysis-api.onrender.com/api/health
```

返回健康状态后，再继续配置 Netlify。

## 4. 打通 Netlify 前端

在仓库根目录 `netlify.toml` 中，把 `/api/*` 代理到 Render 后端，且必须放在 SPA fallback 前面：

```toml
[[redirects]]
  from = "/api/*"
  to = "https://daily-stock-analysis-api.onrender.com/api/:splat"
  status = 200
  force = true

[[redirects]]
  from = "/*"
  to = "/index.html"
  status = 200
```

重新执行 Netlify preview deploy 后，验证：

```text
https://<netlify-preview-url>/api/health
```

如果该地址返回 Render 后端健康状态，前后端已经打通。

## 5. 持久化说明

Render Blueprint 会挂载 1GB 磁盘到 `/app/data`，用于保存：

- SQLite 数据库：`/app/data/stock_analysis.db`
- WebUI 保存的运行时配置：`/app/data/runtime.env`
- 后端日志目录：`/app/data/logs`

如果删除 Render 服务或磁盘，历史数据和 WebUI 保存的配置也会丢失。

## 6. 常见问题

### 健康检查失败

检查 Render 日志中是否出现依赖安装、启动命令或端口监听错误。服务必须监听 Render 注入的 `PORT`。

### Netlify 页面能打开，但 API 报错

确认 `netlify.toml` 中 `/api/*` 代理规则在 `/*` fallback 规则之前，并且 Render 后端 `/api/health` 可公开访问。

### 登录后仍然像没登录

优先使用 Netlify `/api/*` 代理，不建议让前端直接跨域访问 Render 后端。该项目管理员认证使用 Cookie，同源代理更稳定。
