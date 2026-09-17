# lib/sync/ — sync-api 服务端实现

实现本仓定义的对外契约 **sync-api**（见 `docs/contracts/sync-api.md`）：

- 设备注册与 Bearer token 签发
- 会话上传（body 对齐 session-schema）
- 会话列表/详情查询
- 趋势聚合查询（按周/月）

被 `app/api/v1/` 路由调用；消费方为 batana-gui 与 batana-pi。
