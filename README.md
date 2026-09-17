<p align="center">
  <img src="https://raw.githubusercontent.com/vinnie-luckfocus/batana/main/assets/logo.png" width="150" alt="batana logo">
</p>

# batana-web — Web 管理平台

batana 生态（棒球打击动作捕捉 / 分析 / 评价）的云端管理平台，提供**数据统计、趋势展示、会话管理**能力；后期演进为支持**多身份、多用户**（球员 / 教练 / 机构）的数据仓库。

本仓同时定义对外契约 **sync-api**（见 `docs/contracts/sync-api.md`），供 batana-gui / batana-pi 消费。

## 技术栈

- **Next.js（App Router）+ TypeScript** — 页面与服务端 API
- **Postgres（Neon）** — 会话/指标云存储，schema 对齐 session-schema
- **Cloudflare R2** — 对象存储：会话视频与模型工件（预签名 URL 直传）
- **Vercel** — 部署
- Recharts / ECharts — 图表；OIDC（Clerk 或 Auth.js，可替换实现）— 认证（选型待定）

## 目录结构

```
batana-web/
├── app/              # Next.js 路由（dashboard / sessions / trends / devices）
├── lib/db/           # schema 与迁移（对齐 session-schema）
├── lib/sync/         # sync-api 服务端实现
├── components/       # 图表与仪表盘组件
└── docs/contracts/   # sync-api.md（本仓定义）
```

## 功能边界

**v1（P4）：** 账号体系（单用户多设备）、会话/指标云存储、个人仪表盘、训练趋势图（按周/月/季）、设备管理

**v2（P5）：** 多身份角色模型（球员/教练/机构）、数据权限隔离、数仓分层（明细层/汇总层）、开放分析接口、周期报告

**不做：** 实时推理（云端重分析调用 batana-core Python 侧，作为异步任务）、客户端 UI、硬件管理细节（仅存设备元数据）

## 相关仓库

| 仓库 | 职责 |
|---|---|
| [batana](https://github.com/vinnie-luckfocus/batana) | 司令塔：产品定义、版本、契约索引 |
| [batana-core](https://github.com/vinnie-luckfocus/batana-core) | 模型系统核心；定义 session-schema |
| [batana-gui](https://github.com/vinnie-luckfocus/batana-gui) | 跨平台 GUI（sync-api 消费方） |
| [batana-pi](https://github.com/vinnie-luckfocus/batana-pi) | 双目边缘计算设备（sync-api 消费方） |
| [batana-cap](https://github.com/vinnie-luckfocus/batana-cap) | 棒尾 IMU 传感器 |

## License

见 [LICENSE](LICENSE)。
