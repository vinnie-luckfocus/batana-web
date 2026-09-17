# sync-api — 云端同步接口契约

> 版本：v1-draft（2026-09-17）· 归属仓库：batana-web · 消费方：batana-gui、batana-pi
>
> 契约变更遵循主仓 `docs/versioning.md`：主版本不变则向后兼容（字段只增不改），破坏性变更升主版本并保留旧版本至少一个 combo 周期。

## 1. 概述

sync-api 是 gui / pi 端与 batana-web 之间的 REST 接口，覆盖：**设备注册、会话上传、会话查询、趋势聚合查询**。会话数据体与 batana-core 定义的 **session-schema**（v1-draft）对齐。

- Base URL：`https://<host>/api/v1`
- 数据格式：JSON（`Content-Type: application/json`）
- 时间格式：ISO 8601（UTC），如 `2026-09-17T08:30:00Z`

## 2. 认证

所有接口（除设备注册外）均需 **Bearer token**：

```
Authorization: Bearer <device_token>
```

token 由设备注册接口签发，绑定单个设备与所属用户。token 泄露时可调用设备注销接口吊销。

## 3. 错误格式

统一错误响应：

```json
{
  "error": {
    "code": "SESSION_TOO_LARGE",
    "message": "会话数据超过大小上限（20MB）"
  }
}
```

| HTTP 状态码 | code 示例 | 说明 |
|---|---|---|
| 400 | `INVALID_BODY` | 请求体不符合 schema |
| 401 | `UNAUTHORIZED` | token 缺失或无效 |
| 403 | `FORBIDDEN` | 无权访问目标资源（如他人会话） |
| 404 | `NOT_FOUND` | 资源不存在 |
| 409 | `SESSION_EXISTS` | 会话 id 重复上传（幂等冲突） |
| 413 | `SESSION_TOO_LARGE` | 会话数据超过大小上限 |
| 429 | `RATE_LIMITED` | 触发限流 |

## 4. 接口一览

| 方法 | 路径 | 说明 | 认证 |
|---|---|---|---|
| POST | `/api/v1/devices` | 设备注册，签发 token | 用户凭证（注册流程内） |
| DELETE | `/api/v1/devices/{device_id}` | 设备注销，吊销 token | Bearer |
| GET | `/api/v1/devices` | 当前用户设备列表 | Bearer |
| POST | `/api/v1/sessions` | 上传一次挥棒会话 | Bearer |
| GET | `/api/v1/sessions` | 会话列表（分页/筛选） | Bearer |
| GET | `/api/v1/sessions/{session_id}` | 会话详情 | Bearer |
| GET | `/api/v1/trends` | 趋势聚合查询（按周/月） | Bearer |

---

## 5. 设备注册

### POST /api/v1/devices

注册一台设备（gui 客户端或 pi 边缘设备），返回 Bearer token。设备元数据仅存储，不做硬件管理。

**请求体：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `device_name` | string | 是 | 设备显示名，如「Vinnie 的 iPhone」 |
| `device_type` | string | 是 | `gui` / `pi` |
| `platform` | string | 是 | `ios` / `android` / `macos` / `linux` |
| `app_version` | string | 是 | 客户端版本，如 `0.1.0` |
| `hardware_id` | string | 否 | 硬件序列号（pi 设备建议提供） |

```json
{
  "device_name": "Vinnie 的 iPhone",
  "device_type": "gui",
  "platform": "ios",
  "app_version": "0.1.0"
}
```

**响应 201：**

```json
{
  "device_id": "dev_01J8ZK2Q8M3N4P5R6S7T8V9W0X",
  "token": "btn_xxxxxxxxxxxxxxxxxxxxxxxx",
  "created_at": "2026-09-17T08:00:00Z"
}
```

> `token` 仅在注册响应中返回一次，客户端须本地妥善保存。

### GET /api/v1/devices

**响应 200：**

```json
{
  "devices": [
    {
      "device_id": "dev_01J8ZK2Q8M3N4P5R6S7T8V9W0X",
      "device_name": "Vinnie 的 iPhone",
      "device_type": "gui",
      "platform": "ios",
      "app_version": "0.1.0",
      "last_seen_at": "2026-09-17T08:30:00Z",
      "created_at": "2026-09-17T08:00:00Z"
    }
  ]
}
```

### DELETE /api/v1/devices/{device_id}

注销设备并吊销其 token。**响应 204**（无 body）。

---

## 6. 会话上传

### POST /api/v1/sessions

上传一次挥棒会话。body 结构对齐 **session-schema v1-draft**（姿态序列、IMU 序列、评分、指标）；契约侧字段只增不改。上传为**幂等**操作：`session_id` 重复时返回 409，不覆盖已有数据。

**请求体：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `session_id` | string | 是 | 客户端生成的会话唯一 id（ULID/UUID） |
| `schema_version` | string | 是 | session-schema 版本，当前为 `"v1"` |
| `captured_at` | string | 是 | 采集时间（ISO 8601） |
| `duration_ms` | integer | 是 | 会话时长（毫秒） |
| `source` | object | 是 | 采集来源：`device_id`、`capture_mode`（`monocular` / `stereo`） |
| `pose_sequence` | array | 是 | 姿态序列，逐帧 2D/3D 关键点 |
| `imu_sequence` | array | 否 | 棒尾 IMU 序列（cap 接入后提供） |
| `scores` | object | 是 | 评分：总分 + 各维度分 |
| `metrics` | object | 是 | 指标：挥棒速度、髋肩分离角等 |

**请求示例：**

```json
{
  "session_id": "ses_01J900A1B2C3D4E5F6G7H8J9K0",
  "schema_version": "v1",
  "captured_at": "2026-09-17T08:25:00Z",
  "duration_ms": 1450,
  "source": {
    "device_id": "dev_01J8ZK2Q8M3N4P5R6S7T8V9W0X",
    "capture_mode": "monocular"
  },
  "pose_sequence": [
    {
      "t_ms": 0,
      "keypoints": [
        { "name": "left_shoulder", "x": 0.412, "y": 0.305, "z": null, "confidence": 0.97 }
      ]
    }
  ],
  "imu_sequence": [
    { "t_ms": 0, "ax": 0.12, "ay": -9.78, "az": 0.34, "gx": 1.2, "gy": 0.4, "gz": -0.8 }
  ],
  "scores": {
    "total": 82,
    "dimensions": {
      "stance": 85,
      "load": 78,
      "swing": 84,
      "follow_through": 81
    }
  },
  "metrics": {
    "bat_speed_mps": 28.4,
    "hip_shoulder_separation_deg": 42.5,
    "swing_duration_ms": 210
  }
}
```

**响应 201：**

```json
{
  "session_id": "ses_01J900A1B2C3D4E5F6G7H8J9K0",
  "received_at": "2026-09-17T08:30:12Z"
}
```

---

## 7. 会话查询

### GET /api/v1/sessions

会话列表，按 `captured_at` 倒序分页。

**查询参数：**

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `from` | string | — | 起始时间（ISO 8601，含） |
| `to` | string | — | 结束时间（ISO 8601，不含） |
| `device_id` | string | — | 按采集设备筛选 |
| `limit` | integer | 20 | 每页条数（1–100） |
| `cursor` | string | — | 上一页返回的分页游标 |

**响应 200：**

```json
{
  "sessions": [
    {
      "session_id": "ses_01J900A1B2C3D4E5F6G7H8J9K0",
      "captured_at": "2026-09-17T08:25:00Z",
      "duration_ms": 1450,
      "capture_mode": "monocular",
      "total_score": 82,
      "bat_speed_mps": 28.4
    }
  ],
  "next_cursor": "cur_01J900ZZZZZZZZZZZZZZZZZZZZ",
  "has_more": true
}
```

> 列表项为摘要字段；完整姿态/IMU 序列通过详情接口获取。

### GET /api/v1/sessions/{session_id}

会话详情，返回与上传 body 相同的完整结构，附加服务端字段 `received_at`。

**响应 200：** 见第 6 节请求示例结构，另含 `"received_at": "2026-09-17T08:30:12Z"`。

---

## 8. 趋势聚合查询

### GET /api/v1/trends

按周/月聚合训练指标，用于趋势图展示。

**查询参数：**

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `bucket` | string | `week` | 聚合粒度：`week` / `month` |
| `from` | string | — | 起始时间（ISO 8601，含） |
| `to` | string | — | 结束时间（ISO 8601，不含） |
| `metrics` | string | 全部 | 逗号分隔的指标名，如 `total_score,bat_speed_mps` |

**响应 200：**

```json
{
  "bucket": "week",
  "series": [
    {
      "period": "2026-W38",
      "session_count": 12,
      "aggregates": {
        "total_score": { "avg": 79.3, "max": 88, "min": 70 },
        "bat_speed_mps": { "avg": 27.9, "max": 30.1, "min": 25.6 }
      }
    }
  ]
}
```

聚合统计量固定为 `avg` / `max` / `min`；`session_count` 为该周期会话数。

---

## 9. 版本与演进

- v1 仅支持单用户多设备；所有资源按 token 所属用户隔离。
- v2 计划：多身份角色（球员/教练/机构）与跨用户数据权限，查询接口将引入 `subject_id` 等参数（向后兼容方式新增）。
- 会话数据如需云端重分析，由 web 侧异步调用 batana-core，不在本契约范围内。
