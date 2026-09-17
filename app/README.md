# app/ — Next.js 路由层

基于 Next.js App Router 的页面与 API 路由。规划路由：

- `dashboard/` — 个人仪表盘（会话统计、核心指标总览）
- `sessions/` — 会话列表与详情
- `trends/` — 训练趋势图（按周/月/季聚合）
- `devices/` — 设备管理（注册设备元数据）
- `api/v1/` — sync-api 服务端入口（实现见 `lib/sync/`）
