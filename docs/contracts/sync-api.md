# sync-api — 云端同步接口契约

> 版本：1.0-draft（2026-09-17）· 归属仓库：batana-web · 消费方：batana-gui、batana-pi
>
> 契约变更遵循主仓 `docs/versioning.md`：主版本不变则向后兼容（字段只增不改），破坏性变更升主版本并保留旧版本至少一个 combo 周期。

## 1. 概述

sync-api 是 gui / pi 端与 batana-web 之间的 REST 接口，覆盖：**设备注册与认证、视频带外上传、会话上传、会话查询、趋势聚合查询、模型工件分发**。会话数据体与 batana-core 定义的 **session-schema**（1.0-draft）对齐，请求体直接内嵌 session 结构，不做二次包装。

本契约承担两项带外职责：

- **视频传输**：会话视频不走 JSON body，经预签名 URL 直传对象存储（Cloudflare R2），会话内仅存引用与校验值（见第 6 节）。
- **模型分发**：向 gui / pi 端下发模型工件清单（下载 URL、SHA256、版本与档位约束），本契约是模型工件分发的权威接口（见第 10 节）。

- Base URL：`https://<host>/api/v1`
- 数据格式：JSON（`Content-Type: application/json`）
- 时间格式：ISO 8601 UTC（`Z` 后缀），如 `2026-09-17T08:30:00Z`

## 2. 认证

认证机制与具体实现无关：用户侧采用 **OIDC Authorization Code + PKCE** 完成登录授权，换取服务端签发的 **JWT access token**；设备 token 在用户授权后由设备注册接口签发。

> **客户端约束（Qt 原生）**：gui 为 Qt6 原生客户端，无内嵌浏览器依赖，授权流程须通过系统浏览器跳转 + 自定义 URI scheme（或 loopback 重定向）回调完成 PKCE 流程；不得依赖 cookie 会话。
>
> **实现说明**：Clerk / Auth.js 等仅为服务端可替换的实现细节，消费方不得依赖其专有接口或 SDK，一切交互以本节契约为准。

所有接口（除设备注册、模型清单外）均需 **Bearer token**：

```
Authorization: Bearer <access_token>
```

### token 生命周期

- `access_token`：JWT，有效期由注册响应的 `expires_in`（秒）给出，过期后请求返回 401。
- `refresh_token`：长期凭证，用于换取新 token（见下）；泄露时可调用设备注销接口吊销。
- **刷新流程**：`POST /api/v1/auth/token`，body 为 `{ "grant_type": "refresh_token", "refresh_token": "...", "device_id": "..." }`，响应同设备注册的 token 结构（含新的 `access_token`、`expires_in`，并轮换 `refresh_token`）。
- **重注册流程**：`refresh_token` 失效（被吊销 / 轮换过期）时刷新接口返回 401，客户端须引导用户重新走 OIDC 授权后再次调用设备注册。

### POST /api/v1/auth/token

**请求体：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `grant_type` | string | 是 | 固定 `refresh_token` |
| `refresh_token` | string | 是 | 当前持有的刷新凭证 |
| `device_id` | string | 是 | 设备 id（刷新凭证与设备绑定校验） |

**响应 200：** 同第 5 节设备注册的 token 响应结构（`access_token` / `token_type` / `expires_in` / `refresh_token`）。

## 3. 错误格式

统一错误响应：

```json
{
  "error": {
    "code": "SESSION_JSON_TOO_LARGE",
    "message": "会话 JSON 超过大小上限（20MB）"
  }
}
```

| HTTP 状态码 | code 示例 | 说明 |
|---|---|---|
| 400 | `INVALID_BODY` | 请求体不符合 schema |
| 401 | `UNAUTHORIZED` | token 缺失、过期或无效 |
| 403 | `FORBIDDEN` | 无权访问目标资源（如他人会话） |
| 404 | `NOT_FOUND` | 资源不存在 |
| 409 | `SESSION_EXISTS` | 会话 id 重复上传（幂等冲突，body 携带已存记录 `content_hash`，见第 7 节） |
| 413 | `SESSION_JSON_TOO_LARGE` | 会话 JSON body 超过大小上限（20MB） |
| 413 | `VIDEO_TOO_LARGE` | 视频文件超过大小上限（500MB），仅出现于视频上传接口 |
| 429 | `RATE_LIMITED` | 触发限流 |

**429 约定**：响应必须携带 `Retry-After` 头（单位：秒），客户端在该时长内不得重试；错误 body 同时给出 `retry_after_seconds` 便于日志记录。

**413 区分**：会话 JSON 超限（`SESSION_JSON_TOO_LARGE`）与视频超限（`VIDEO_TOO_LARGE`）是两个独立上限；视频走带外上传，不计入会话 body 大小。

## 4. 接口一览

| 方法 | 路径 | 说明 | 认证 |
|---|---|---|---|
| POST | `/api/v1/devices` | 设备注册，签发 token | OIDC 用户授权（注册流程内） |
| DELETE | `/api/v1/devices/{device_id}` | 设备注销，吊销 token | Bearer |
| GET | `/api/v1/devices` | 当前用户设备列表 | Bearer |
| POST | `/api/v1/auth/token` | 刷新 token | refresh_token |
| POST | `/api/v1/videos` | 申请视频预签名上传 URL | Bearer |
| POST | `/api/v1/sessions` | 上传一次挥棒会话 | Bearer |
| GET | `/api/v1/sessions` | 会话列表（分页/筛选） | Bearer |
| GET | `/api/v1/sessions/{session_id}` | 会话详情 | Bearer |
| GET | `/api/v1/trends` | 趋势聚合查询（按周/月） | Bearer |
| GET | `/api/v1/models/manifest` | 模型工件清单 | Bearer |

---

## 5. 设备注册

### POST /api/v1/devices

注册一台设备（gui 客户端或 pi 边缘设备），返回 Bearer token。设备元数据仅存储，不做硬件管理。调用前须完成 OIDC Authorization Code + PKCE 用户授权。

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
  "access_token": "eyJhbGciOiJ...",
  "token_type": "Bearer",
  "expires_in": 3600,
  "refresh_token": "btnr_xxxxxxxxxxxxxxxxxxxxxxxx",
  "created_at": "2026-09-17T08:00:00Z"
}
```

> `access_token` 过期后用 `refresh_token` 调 `POST /api/v1/auth/token` 换新（见第 2 节）；`refresh_token` 客户端须本地妥善保存。

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

注销设备并吊销其 token（含 refresh_token）。**响应 204**（无 body）。

---

## 6. 视频上传（带外）

会话视频体积大，不随会话 JSON 传输。流程：**先申请预签名 URL 直传对象存储（Cloudflare R2），再上传会话并在 `video` 字段携带引用与 SHA256 校验值**。

### POST /api/v1/videos

**请求体：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `content_type` | string | 是 | 视频 MIME，如 `video/mp4` |
| `size_bytes` | integer | 是 | 文件大小（字节），超限（500MB）直接返回 413 `VIDEO_TOO_LARGE` |
| `sha256` | string | 是 | 文件内容 SHA256（hex，小写），服务端校验完整性 |

```json
{
  "content_type": "video/mp4",
  "size_bytes": 188743680,
  "sha256": "9f2c4a..."
}
```

**响应 201：**

```json
{
  "video_id": "vid_01J90B2C3D4E5F6G7H8J9K0L",
  "upload_url": "https://<r2-presigned-url>",
  "upload_method": "PUT",
  "upload_headers": { "Content-Type": "video/mp4" },
  "expires_in": 900,
  "uri": "r2://batana-videos/usr_xxx/vid_01J90B2C3D4E5F6G7H8J9K0L.mp4"
}
```

客户端在 `expires_in` 秒内向 `upload_url` 直传文件；会话上传时以返回的 `uri` 填入 `video.uri`，服务端按 `sha256` 校验对象存储中的实际内容。视频未上传完成即提交会话，服务端接受会话但 `video` 状态标记为待补传；会话查询不阻塞。

---

## 7. 会话上传

### POST /api/v1/sessions

上传一次挥棒会话。**body 直接内嵌 session-schema 结构**（1.0-draft），字段与 batana-core 定义逐一对齐；契约侧字段只增不改。上传为**幂等**操作：`meta.session_id` 重复时返回 409，不覆盖已有数据，响应 body 携带已存记录的 `content_hash`（服务端对规范化 session JSON 计算的 SHA256），客户端据此比对本地内容是否一致——一致则视为成功，不一致属冲突须人工处理。

**请求体（顶层）：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `schema_version` | string | 是 | session-schema 语义化版本字符串，当前为 `"1.0"`；主版本号用于兼容性判断（消费方拒绝解析高于自身支持的主版本） |
| `meta` | object | 是 | 会话元数据（见下） |
| `video` | object | 否 | 视频引用（带外上传后回填；standard-imu 档位无此字段） |
| `pose2d` | object | 否 | 2D 姿态序列（standard-vision / pro-fusion / pro-stereo / max 档位出现） |
| `imu` | object | 否 | 棒尾 IMU 序列（standard-imu / pro-fusion / max 档位出现） |
| `phases` | object | 否 | 阶段分割（能力 `phase_split` 产出时出现） |
| `metrics` | object | 是 | 评分与指标 |
| `extensions` | object | 否 | 预留扩展位（3D 姿态等实验字段先进此处，稳定后升版转正） |

**`meta`：**

| 字段 | 类型 | 必填 | 说明 |
|---|---|---|---|
| `session_id` | string | 是 | 客户端生成的全局唯一会话 id（ULID/UUID） |
| `created_at` | string | 是 | 采集时间，ISO 8601 UTC（`Z` 后缀） |
| `device` | object | 是 | 采集设备：`model`、`os` |
| `app` | object | 是 | 产生会话的应用：`name`、`version` |
| `pipeline` | string | 是 | 处理本会话的管线档位：`standard-vision` / `standard-imu` / `pro-fusion` / `pro-stereo` / `max` |
| `handedness` | string | 是 | 打击手惯用手：`right` / `left` |
| `subject` | object | 否 | 受试者（匿名化）：`height_cm`、`age_group`（`adult` / `teen` / `kid`） |

**`video`：** `uri`（第 6 节返回的引用）、`fps`、`resolution`（`[宽, 高]`）、`duration_ms`、`view`（`side` / `front` / `behind` / `other`）、`sha256`。

**`pose2d`：** `model`（姿态模型 id）、`frame_rate`、`frames[]`。每帧：`frame_index`、`timestamp_ms`（相对会话起点毫秒）、`confidence`（整帧置信度）、`keypoints[]`（固定 33 点，MediaPipe BlazePose 拓扑；每点 `name` / `x` / `y` / `visibility`，归一化坐标，缺失点 `visibility=0`）。**2D 序列不含 `z`；3D 姿态经 `extensions` 传输。**

**`imu`：** `cap_id`（batana-cap 设备 id）、`sample_rate_hz`、`samples[]`。每样本：`timestamp_ms`、`accel`（`[x, y, z]`，m/s²）、`gyro`（`[x, y, z]`，rad/s）。

**`phases`：** `granularity`（`fine` / `coarse`）、`segments[]`（`phase` / `start_ms` / `end_ms`，时间轴与 video / imu 同原点）。

**`metrics`：** `capabilities`（本次实际产出的能力 id 数组，必填）、`scores`、各能力指标（`bat_speed_mps`、`swing_count`、`tempo_ratio` 等）、`details`（档位增强指标）。

**评分语义：** `metrics.scores` 固定为 `{ speed, angle, coordination, overall }`，均为 0–100，对齐 core 的能力维度语义；`overall` 为综合评分，聚合与展示一律使用 `overall`。按阶段评分（stance / load / swing / follow_through 维度）属 core 未登记字段，不在 v1 范围内，如需试验放入 `extensions`。

**请求示例：**

```json
{
  "schema_version": "1.0",
  "meta": {
    "session_id": "ses_01J900A1B2C3D4E5F6G7H8J9K0",
    "created_at": "2026-09-17T08:25:00Z",
    "device": { "model": "iPhone 16 Pro", "os": "iOS 19.0" },
    "app": { "name": "batana", "version": "0.1.0" },
    "pipeline": "standard-vision",
    "handedness": "right",
    "subject": { "height_cm": 175, "age_group": "adult" }
  },
  "video": {
    "uri": "r2://batana-videos/usr_xxx/vid_01J90B2C3D4E5F6G7H8J9K0L.mp4",
    "fps": 240,
    "resolution": [1280, 720],
    "duration_ms": 1850,
    "view": "side",
    "sha256": "9f2c4a..."
  },
  "pose2d": {
    "model": "batana-pose-v0.1",
    "frame_rate": 240,
    "frames": [
      {
        "frame_index": 0,
        "timestamp_ms": 0,
        "confidence": 0.92,
        "keypoints": [
          { "name": "nose", "x": 0.512, "y": 0.238, "visibility": 0.98 }
        ]
      }
    ]
  },
  "imu": {
    "cap_id": "cap_04a2",
    "sample_rate_hz": 200,
    "samples": [
      { "timestamp_ms": 0, "accel": [0.12, -9.78, 0.45], "gyro": [0.01, 0.02, -0.15] }
    ]
  },
  "phases": {
    "granularity": "fine",
    "segments": [
      { "phase": "stance", "start_ms": 0, "end_ms": 620 },
      { "phase": "load_stride", "start_ms": 620, "end_ms": 980 },
      { "phase": "swing_contact", "start_ms": 980, "end_ms": 1180 },
      { "phase": "follow_through", "start_ms": 1180, "end_ms": 1850 }
    ]
  },
  "metrics": {
    "capabilities": ["pose2d", "phase_split", "score_basic", "bat_speed"],
    "bat_speed_mps": 28.4,
    "swing_count": 1,
    "tempo_ratio": 2.6,
    "scores": { "speed": 72, "angle": 65, "coordination": 70, "overall": 69 }
  },
  "extensions": {}
}
```

**响应 201：**

```json
{
  "session_id": "ses_01J900A1B2C3D4E5F6G7H8J9K0",
  "content_hash": "sha256:3b8f1d...",
  "received_at": "2026-09-17T08:30:12Z"
}
```

**响应 409（幂等冲突）：**

```json
{
  "error": {
    "code": "SESSION_EXISTS",
    "message": "会话已存在",
    "content_hash": "sha256:3b8f1d..."
  }
}
```

---

## 8. 会话查询

### GET /api/v1/sessions

会话列表，按 `meta.created_at` 倒序分页。

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
      "created_at": "2026-09-17T08:25:00Z",
      "duration_ms": 1850,
      "pipeline": "standard-vision",
      "overall": 69,
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

**响应 200：** 见第 7 节请求示例结构，另含 `"received_at": "2026-09-17T08:30:12Z"`。

---

## 9. 趋势聚合查询

### GET /api/v1/trends

按周/月聚合训练指标，用于趋势图展示。

**查询参数：**

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `bucket` | string | `week` | 聚合粒度：`week` / `month` |
| `from` | string | — | 起始时间（ISO 8601，含） |
| `to` | string | — | 结束时间（ISO 8601，不含） |
| `metrics` | string | 全部 | 逗号分隔的指标名，如 `overall,bat_speed_mps` |

**响应 200：**

```json
{
  "bucket": "week",
  "series": [
    {
      "period": "2026-W38",
      "session_count": 12,
      "aggregates": {
        "overall": { "avg": 68.3, "max": 79, "min": 58 },
        "bat_speed_mps": { "avg": 27.9, "max": 30.1, "min": 25.6 }
      }
    }
  ]
}
```

聚合统计量固定为 `avg` / `max` / `min`；`session_count` 为该周期会话数。聚合指标名与 `metrics` 内字段一致（综合评分为 `overall`，对应 `metrics.scores.overall`）。

---

## 10. 模型分发

本契约承担模型工件分发职责：gui / pi 端启动或按需拉取清单，自行下载、校验并加载模型。

### GET /api/v1/models/manifest

**查询参数：**

| 参数 | 类型 | 默认 | 说明 |
|---|---|---|---|
| `tier` | string | — | 按适用档位筛选：`standard-vision` / `standard-imu` / `pro-fusion` / `pro-stereo` / `max` |
| `runtime_version` | string | — | 客户端 core 运行时版本，服务端据此过滤不满足 `min_runtime_version` 的工件 |

**响应 200：**

```json
{
  "generated_at": "2026-09-17T08:00:00Z",
  "models": [
    {
      "id": "batana-pose",
      "version": "0.1.0",
      "url": "https://<r2-or-cdn>/models/batana-pose/0.1.0/model.bin",
      "sha256": "c41a9e...",
      "size_bytes": 8388608,
      "min_runtime_version": "0.1.0",
      "tiers": ["standard-vision", "pro-fusion", "pro-stereo", "max"]
    }
  ]
}
```

- `url` 为带有效期的下载地址（对象存储 / CDN），客户端下载后必须校验 `sha256`。
- `min_runtime_version` 为该工件要求的 core 运行时最低版本（语义化版本比较）。
- 客户端按本地档位与运行时版本选择工件；清单内同一模型出现多版本时取满足约束的最高版本。

---

## 11. 版本与演进

- v1 仅支持单用户多设备；所有资源按 token 所属用户隔离。
- v2 计划：多身份角色（球员/教练/机构）与跨用户数据权限，查询接口将引入 `subject_id` 等参数（向后兼容方式新增）。
- 会话数据如需云端重分析，由 web 侧异步调用 batana-core，不在本契约范围内。

## 变更记录

### 1.0-draft（2026-09-17）— 评审整改

起草时上游 session-schema 未定稿，导致契约字段系统性不对齐，本次整改：

1. **会话上传 body 重写**为直接内嵌 session 结构：`captured_at` → `meta.created_at`（UTC `Z`）；`pose_sequence` → `pose2d.frames`（`t_ms` → `timestamp_ms`，逐点 `confidence` → `visibility`，删除 `z`，3D 走 `extensions`）；`imu_sequence` → `imu.samples`（`ax/ay/az` → `accel` 数组 m/s²，`gx/gy/gz` → `gyro` 数组 rad/s，补 `sample_rate_hz` / `cap_id`）；`source.capture_mode` → `meta.pipeline`（枚举 `standard-vision` / `standard-imu` / `pro-fusion` / `pro-stereo` / `max`）；补齐 `phases`、`meta.handedness`、`metrics.capabilities`。
2. **评分语义统一**：`scores` 改为 `{ speed, angle, coordination, overall }`；趋势聚合指标 `total_score` → `overall`；按阶段评分属 core 未登记字段，不在 v1。
3. **`schema_version` 统一为语义化版本字符串 `"1.0"`**，主版本号用于兼容性判断。
4. **新增视频带外上传**：`POST /api/v1/videos` 返回预签名 URL 直传对象存储（Cloudflare R2）；会话 body 携带 `video:{uri, fps, resolution, duration_ms, view, sha256}`；413 区分会话 JSON 超限与视频超限。
5. **新增模型分发职责**：`GET /api/v1/models/manifest` 返回模型 id、版本、下载 URL、SHA256、所需 runtime 最低版本、适用档位。
6. **幂等完善**：409 响应 body 携带已存记录 `content_hash` 供客户端比对；429 补 `Retry-After` 约定。
7. **认证完善**：token 响应补 `expires_in`，新增刷新 / 重注册流程；认证机制明确为与实现无关的 OIDC Authorization Code + PKCE 换 JWT（Qt 原生客户端经系统浏览器 + URI scheme 回调；Clerk / Auth.js 仅为服务端可替换实现细节）。
8. 文件头版本统一为 `1.0-draft`；同步司令塔 `repos.yaml` 契约登记（`sync-api: 1.0-draft`）。
