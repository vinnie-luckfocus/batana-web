# lib/db/ — 数据库层

Postgres（Neon）的 schema 定义与迁移脚本。

- 存储模型与 batana-core 定义的 **session-schema** 契约对齐（姿态序列、IMU 序列、评分、指标）
- session-schema 升主版本时，在此提供对应迁移脚本
- v2 演进为数仓分层：明细层（原始会话）/ 汇总层（周/月/季聚合）
