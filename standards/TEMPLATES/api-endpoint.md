# 模板: API 端点文档

**用途**: 记录单个API端点的详细规格  
**目标读者**: Coder, Tester, Designer  
**维护者**: @coder

---

## 端点信息

| 字段 | 值 |
|------|-----|
| **Endpoint** | `POST /api/strategies/` |
| **Method** | `POST` |
| **Summary** | 创建新策略 |
| **Owner** | @coder |
| **Status** | `stable` / `beta` / `deprecated` |
| **Since** | 2026-02-15 |

---

## 描述

简短描述端点的业务用途和上下文。

---

## 请求

### Headers

| 名称 | 类型 | 必需 | 描述 |
|------|------|------|------|
| `Authorization` | string | ✅ | `Bearer <access_token>` |
| `Content-Type` | string | ✅ | `application/json` |

### Body Schema

```json
{
  "type": "object",
  "required": ["name", "parameters"],
  "properties": {
    "name": {
      "type": "string",
      "minLength": 1,
      "maxLength": 100,
      "description": "策略名称"
    },
    "description": {
      "type": "string",
      "description": "策略描述 (可选)"
    },
    "parameters": {
      "type": "object",
      "description": "策略参数 (JSON)",
      "example": {
        "indicators": ["SMA", "RSI"],
        "timeframe": "1d"
      }
    }
  }
}
```

**示例**:

```json
{
  "name": "My RSI Strategy",
  "description": "基于RSI超买超卖的策略",
  "parameters": {
    "rsi_period": 14,
    "oversold": 30,
    "overbought": 70
  }
}
```

---

## 响应

### 成功 (200 Created)

```json
{
  "id": 123,
  "name": "My RSI Strategy",
  "user_id": 42,
  "parameters": { ... },
  "created_at": "2026-03-03T01:23:45Z",
  "updated_at": "2026-03-03T01:23:45Z"
}
```

### 错误

| HTTP 状态码 | error_code | 描述 | 示例 |
|-------------|------------|------|------|
| `400 Bad Request` | `VALIDATION_ERROR` | 请求参数验证失败 | `{"error":"VALIDATION_ERROR","message":"name不能为空"}` |
| `401 Unauthorized` | `INVALID_TOKEN` | 认证失败 | `{"error":"INVALID_TOKEN","message":"Token expired"}` |
| `403 Forbidden` | `INSUFFICIENT_SCOPE` | 权限不足 | `{"error":"INSUFFICIENT_SCOPE","message":"Cannot create strategy"}` |
| `409 Conflict` | `STRATEGY_LIMIT_REACHED` | 达到创建上限 | `{"error":"STRATEGY_LIMIT_REACHED","message":"最多创建10个策略"}` |
| `500 Internal Server Error` | `INTERNAL_ERROR` | 服务器内部错误 | `{"error":"INTERNAL_ERROR","message":"Unexpected error"}` |

---

## 业务规则

- 每个用户最多创建 10 个策略
- 策略名称在同一用户下必须唯一
- `parameters` 必须符合策略类型的 JSON Schema (见 `schemas/strategies.json`)

---

## 数据模型

关联的数据库表: `strategies`

| 列名 | 类型 | 描述 |
|------|------|------|
| `id` | INT (PK) | 主键，自增 |
| `user_id` | INT (FK) | 关联 `users.id` |
| `name` | VARCHAR(100) | 策略名称 |
| `parameters` | JSON | 策略参数 |
| `created_at` | DATETIME | 创建时间 |
| `updated_at` | DATETIME | 更新时间 |

---

## 测试

### 单元测试

```python
def test_create_strategy_success(client, admin_user):
    response = client.post("/api/strategies", json={
        "name": "Test",
        "parameters": {"indicator": "SMA", "period": 20}
    }, headers={"Authorization": f"Bearer {admin_user.token}"})
    assert response.status_code == 201
    assert response.json()["id"] is not None
```

### 集成测试

- [ ] 创建策略后数据库存在记录
- [ ] 同名策略创建失败 (返回409)
- [ ] 达到上限后创建失败 (返回429或409)

### E2E测试

```typescript
test('user can create strategy from UI', async ({ page }) => {
  await page.goto('/strategies/new')
  await page.fill('[name=name]', 'My Strategy')
  await page.click('button[type=submit]')
  await expect(page).toHaveURL('/strategies/123')
})
```

---

## 安全

- ✅ 认证: JWT Bearer token (通过 `Authorization` header)
- ✅ 授权: `@require_login` 装饰器检查用户身份
- ✅ 输入验证: Pydantic schema 验证请求体
- ✅ SQL注入防护: SQLAlchemy ORM (参数化查询)
- ⚠️ 速率限制: 未实现 (建议: 10次/分钟)

---

## 性能

- **DB查询**: 预期 < 50ms (使用 `EXPLAIN` 分析)
- **响应时间**: P95 < 200ms
- **并发**: 支持 100 RPS (需压测验证)

---

## 变更历史

| 日期 | 版本 | 变更内容 | 作者 |
|------|------|----------|------|
| 2026-03-03 | v1.0 | 创建端点文档模板 | @writer |
| 2026-02-15 | v0.9 | 初版策略创建API | @coder |

---

## 相关链接

- API契约（OpenAPI）：`../architecture/API_CONTRACT_V1.yaml`
- [API概览](../development/API_README.md) - 所有端点快速导航
- [策略服务代码](../../app/api/services/strategy_service.py) - 实现代码
- [策略DAO](../../app/domains/strategies/dao/strategy_dao.py) - 数据访问层

---

**使用本模板**: 复制到 `development/api/` 目录，重命名为 `strategy.md` (按端点命名)，填写实际内容，并在 `API_README.md` 添加链接。