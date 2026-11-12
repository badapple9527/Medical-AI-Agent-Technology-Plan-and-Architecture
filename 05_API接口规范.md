# API 接口规范

> 文档版本：V0.2  
> Base URL：`/api/v1`  
> 协议：HTTPS + JSON；流式接口使用 SSE。

---

## 1. 通用约定

### 1.1 认证

```http
Authorization: Bearer <access_token>
X-Tenant-ID: <tenant_id>
X-Request-ID: <uuid>
```

### 1.2 响应格式

成功：

```json
{
  "data": {},
  "request_id": "req_001",
  "trace_id": "trace_001"
}
```

错误：

```json
{
  "error": {
    "code": "CASE_NOT_FOUND",
    "message": "病例不存在",
    "details": {}
  },
  "request_id": "req_001",
  "trace_id": "trace_001"
}
```

### 1.3 错误码

| HTTP | code | 含义 |
|---|---|---|
| 400 | VALIDATION_ERROR | 参数校验失败 |
| 401 | UNAUTHORIZED | 未认证 |
| 403 | FORBIDDEN | 无权限 |
| 404 | NOT_FOUND | 资源不存在 |
| 409 | CONFLICT | 状态冲突 |
| 422 | UNPROCESSABLE_ENTITY | 业务校验失败 |
| 429 | RATE_LIMITED | 限流 |
| 500 | INTERNAL_ERROR | 内部错误 |
| 503 | DEPENDENCY_UNAVAILABLE | 依赖服务不可用 |

### 1.4 幂等

- 创建类接口支持 `Idempotency-Key`；
- 任务重试必须使用同一幂等键；
- 索引同步事件使用 `event_id` 幂等。

### 1.5 分页

```http
GET /api/v1/evidence?page=1&page_size=20
```

```json
{
  "data": {
    "items": [],
    "page": 1,
    "page_size": 20,
    "total": 100
  }
}
```

---

## 2. 认证与用户

### POST `/auth/login`

请求：

```json
{
  "username": "doctor001",
  "password": "..."
}
```

响应：

```json
{
  "data": {
    "access_token": "...",
    "refresh_token": "...",
    "expires_in": 3600,
    "user": {
      "user_id": "U001",
      "name": "张医生",
      "roles": ["doctor"],
      "tenant_id": "H001"
    }
  }
}
```

### GET `/auth/me`

返回当前用户、角色、数据范围。

---

## 3. 病例与患者状态

### POST `/cases`

创建病例。

```json
{
  "patient_ref": "P001",
  "case_name": "肺癌复杂病例",
  "disease": "NSCLC",
  "created_by": "U001"
}
```

响应：

```json
{
  "data": {
    "case_id": "CASE001",
    "status": "draft"
  }
}
```

### GET `/cases/{case_id}`

返回病例摘要、当前快照、状态。

### POST `/cases/{case_id}/snapshots`

创建患者状态快照。

```json
{
  "as_of": "2026-01-01T12:00:00Z",
  "diagnosis": ["非小细胞肺癌"],
  "stage": "IV",
  "pathology": "腺癌",
  "biomarkers": {
    "EGFR": "pending",
    "PD-L1": "unknown"
  },
  "comorbidities": ["间质性肺病"],
  "organ_function": {
    "creatinine": 1.8,
    "egfr": 45
  },
  "current_medications": [],
  "allergies": [],
  "pending_items": ["基因检测"],
  "signed_by": "U001"
}
```

响应：

```json
{
  "data": {
    "snapshot_id": "PS001",
    "version": 1,
    "status": "signed"
  }
}
```

### GET `/cases/{case_id}/snapshots/{snapshot_id}`

返回快照详情和字段级来源。

---

## 4. 文档与知识生产

### POST `/documents`

上传文档，`multipart/form-data`。

字段：

- `file`；
- `source_type`；
- `tenant_id`；
- `metadata`（JSON 字符串）。

响应：

```json
{
  "data": {
    "document_id": "D001",
    "job_id": "JOB001",
    "status": "queued"
  }
}
```

### GET `/documents/{document_id}`

返回文档元数据、解析状态、审核状态。

### GET `/jobs/{job_id}`

```json
{
  "data": {
    "job_id": "JOB001",
    "status": "processing",
    "progress": 45,
    "stage": "evidence_extraction",
    "message": "正在抽取证据",
    "created_at": "2026-01-01T10:00:00Z"
  }
}
```

状态：

```text
queued
parsing
extracting
normalizing
review_pending
indexing
completed
failed
```

### POST `/documents/{document_id}/reprocess`

重新解析/重新抽取，需权限和原因。

---

## 5. 证据与 Claim

### GET `/evidence/search`

参数：

```text
q=EGFR 间质性肺病 奥希替尼
disease=NSCLC
biomarker=EGFR
evidence_level=high
year_from=2024
page=1
page_size=20
```

响应：

```json
{
  "data": {
    "items": [
      {
        "evidence_id": "E001",
        "claim_id": "C001",
        "quote": "...",
        "summary": "...",
        "evidence_level": "high",
        "source_id": "S001",
        "score": 0.91,
        "highlights": ["..."]
      }
    ],
    "total": 1
  }
}
```

### GET `/evidence/{evidence_id}`

返回证据详情、原文、关系、审核状态。

### GET `/claims/{claim_id}`

返回 Claim 详情和关联证据列表。

### GET `/claims/{claim_id}/bundle`

参数：

```text
version=latest
include=items,relations,conflicts
```

响应：

```json
{
  "data": {
    "bundle_id": "B001",
    "claim_id": "C001",
    "bundle_version": 1,
    "consistency": "mixed",
    "confidence": "medium",
    "items": [
      {
        "evidence_id": "E001",
        "role": "primary",
        "relation": "supports",
        "quote": "...",
        "evidence_level": "high"
      }
    ],
    "conflicts": [],
    "uncertainties": []
  }
}
```

### GET `/evidence/{evidence_id}/provenance`

返回原文定位：

```json
{
  "source_id": "S001",
  "page_no": 12,
  "section_path": "Results > Efficacy",
  "char_start": 1024,
  "char_end": 1056,
  "quote_hash": "...",
  "object_url": "/api/v1/files/S001?page=12"
}
```

---

## 6. 决策接口

### POST `/decisions`

发起一次临床决策。

```json
{
  "case_id": "CASE001",
  "snapshot_id": "PS001",
  "question": "EGFR突变阳性、合并间质性肺病，一线治疗如何选择？",
  "options": {
    "include_frontier": true,
    "language": "zh-CN"
  }
}
```

响应：

```json
{
  "data": {
    "decision_id": "DEC001",
    "status": "queued",
    "stream_url": "/api/v1/decisions/DEC001/stream"
  }
}
```

### GET `/decisions/{decision_id}/stream`

SSE 事件：

```text
event: progress
data: {"stage":"retrieval","progress":20}

event: evidence
data: {"bundle_ids":["B001","B002"]}

event: agent
data: {"agent":"medical_oncology","status":"completed"}

event: report
data: {"report_id":"R001","status":"draft"}

event: error
data: {"code":"INSUFFICIENT_EVIDENCE","message":"证据不足"}
```

### GET `/decisions/{decision_id}`

返回状态、阶段、报告 ID、错误信息。

### GET `/decisions/{decision_id}/trace`

返回可审计链路：

```json
{
  "decision_id": "DEC001",
  "snapshot_id": "PS001",
  "evidence_bundle_ids": ["B001"],
  "model_calls": [
    {
      "stage": "agent_oncology",
      "model": "qwen-plus",
      "prompt_version": "v1",
      "latency_ms": 1200
    }
  ],
  "rule_hits": ["RENAL_DOSE_001"],
  "report_id": "R001"
}
```

---

## 7. 报告接口

### GET `/reports/{report_id}`

返回结构化报告：

```json
{
  "data": {
    "report_id": "R001",
    "decision_id": "DEC001",
    "status": "draft",
    "summary": "...",
    "standard_paths": [],
    "frontier_paths": [],
    "validation_tasks": [],
    "safety_findings": [],
    "conflicts": [],
    "citations": [],
    "uncertainties": [],
    "created_at": "..."
  }
}
```

### POST `/reports/{report_id}/sign`

医生签字。

```json
{
  "decision": "accepted",
  "comment": "同意，补充基因检测后复评",
  "signed_by": "U001"
}
```

`decision`：

```text
accepted
modified
rejected
needs_more_info
```

### POST `/reports/{report_id}/feedback`

提交反馈：

```json
{
  "rating": 4,
  "adopted": true,
  "modified_sections": ["frontier_paths"],
  "reason": "患者拒绝化疗",
  "outcome": {
    "status": "pending"
  }
}
```

---

## 8. 管理接口

| 方法 | 路径 | 功能 |
|---|---|---|
| GET | `/admin/models` | 模型列表 |
| POST | `/admin/models/{id}/enable` | 启用模型 |
| GET | `/admin/rules` | 规则列表 |
| POST | `/admin/rules/{id}/publish` | 发布规则 |
| GET | `/admin/indexes` | 索引状态 |
| POST | `/admin/indexes/rebuild` | 重建索引 |
| GET | `/admin/audit-logs` | 审计日志 |
| GET | `/admin/health` | 健康检查 |

索引重建请求：

```json
{
  "index": "evidence",
  "target_version": "v2",
  "reason": "embedding model upgrade"
}
```

---

## 9. 文件访问

### GET `/files/{source_id}`

返回受控的原文下载/预览地址。

规则：

- 必须鉴权；
- 记录访问审计；
- 支持 range 请求；
- 不暴露对象存储真实地址；
- 敏感文件使用短期签名 URL。

---

## 10. API 安全要求

1. 所有接口默认需要认证；
2. 资源级权限校验；
3. 租户隔离；
4. 输入长度和文件类型限制；
5. 防止提示词注入；
6. 禁止在 URL 中传患者敏感信息；
7. 日志脱敏；
8. 限流；
9. 审计写操作；
10. 下载和导出单独授权。

---

## 11. 评审修订补充（V0.2）

### 11.1 决策接口增加消融配置

`POST /decisions` 增加可选字段：

```json
{
  "ablation_config": {
    "disabled_agents": ["respiratory"],
    "disabled_modules": ["scenario_simulation"],
    "prompt_variant": "v1",
    "model_variant": "default",
    "random_seed": 42,
    "run_label": "ablation_no_respiratory"
  }
}
```

该字段仅对具有实验权限的用户开放，生产临床决策默认关闭。

### 11.2 SSE 重连游标

SSE 事件增加 `id` 字段，客户端重连时发送：

```http
Last-Event-ID: <last_event_id>
```

服务端从该事件之后继续推送；若无法恢复，返回新的运行快照并提示客户端重新同步。

### 11.3 无推荐报告的签字语义

报告签字 `decision` 枚举增加：

```text
acknowledged_no_recommendation
```

用于系统因证据不足、安全审查失败或关键信息缺失而未给出推荐时，医生确认已阅读但不代表认可治疗方案。

### 11.4 文件访问版权拦截

`GET /files/{source_id}` 必须检查 `license_status`：

- `open`：按授权返回完整原文；
- `internal`：仅内部用户可见；
- `restricted`：只返回 quote 片段或元数据；
- `unknown`：禁止返回完整原文；
- 所有下载写审计日志。

### 11.5 强制审计

所有写操作接口必须自动写入 `audit_log`，包括：

- 创建、修改、删除；
- 报告签字；
- 文件下载；
- 权限变更；
- 模型/规则/索引变更；
- 数据导出。


