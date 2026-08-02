# Day 9｜会员 API 与 Tool Calling 详细设计

## 1. 工具不是内部API的原样暴露

面向AI的工具需要围绕一个明确业务动作重新封装：
- 输入少而清晰；
- 安全参数由系统注入；
- 返回稳定的业务事实和原因码；
- 下游再次校验身份、scope和对象归属；
- 不返回模型不需要的内部字段；
- 有超时、重试、幂等和审计规则。

## 2. 推荐工具分层

### 2.1 本人资产只读
- `points_balance.read`
- `points_ledger.search`
- `tier_status.read`
- `benefits_list.read`
- `benefit_detail.read`
- `coupon_list.read`
- `coupon_detail.read`
- `order_summary.read`
- `activity_participation.read`

### 2.2 确定性资格判断
- `coupon_eligibility.check`
- `order_points_eligibility.check`

资格判断放在业务规则服务中，不让模型根据自然语言规则自行计算。

### 2.3 知识与公开数据
- `knowledge_search.search`
- `store_search.search`
- `page_navigation.build`

### 2.4 受控写入
- `case_create.create`
- `human_handoff.create`

首期不提供补积分、补券、退款、账号合并和隐私删除工具。

## 3. 示例Schema

```json
{
  "name": "coupon_eligibility.check",
  "description": "校验当前登录会员的一张优惠券是否可用于指定订单或渠道，并返回稳定原因码。",
  "input": {
    "type": "object",
    "properties": {
      "coupon_ref": {"type": "string", "minLength": 1, "maxLength": 64},
      "order_ref": {"type": ["string", "null"], "maxLength": 64},
      "channel": {"type": "string", "enum": ["app", "h5", "pos", "ecommerce", "theme_park"]}
    },
    "required": ["coupon_ref", "channel"],
    "additionalProperties": false
  },
  "system_injected": ["member_token", "market", "trace_id"]
}
```

模型不能传`member_id`，会员主体由认证上下文注入。

## 4. 统一返回协议

```json
{
  "success": true,
  "trace_id": "tr_xxx",
  "tool_version": "1.2",
  "queried_at": "2026-07-31T10:00:00+08:00",
  "data": {
    "eligible": false,
    "reason_code": "CHANNEL_NOT_SUPPORTED",
    "failed_conditions": ["仅限线下POS使用"]
  },
  "evidence_refs": ["rule_sg_coupon_2026_v3"],
  "redaction_applied": true,
  "error_code": null,
  "retryable": false
}
```

模型只依据`data`和`evidence_refs`解释，不接触下游原始响应。

## 5. 错误分类

必须区分：
- `NOT_FOUND`：本人范围内确实没有记录；
- `NOT_OWNED`：对象不属于本人，不能暴露对象信息；
- `AUTH_EXPIRED`：需要重新登录并清理旧上下文；
- `TIMEOUT`：可有限重试，仍失败不能生成资产；
- `NO_EVIDENCE`：没有有效规则，必须拒答或人工；
- `VERSION_CONFLICT`：知识版本冲突；
- `DATA_CONFLICT`：多个事实源冲突；
- `DUPLICATE`：受控写入命中幂等。

## 6. 幂等和重试

- 只读查询可以对明确可重试错误重试一次；
- 工单创建和人工接管必须携带由服务端生成的`idempotency_key`；
- 相同会话、相同动作和相同业务对象在有效时间窗内只创建一次；
- 不允许模型因为“没看到结果”而自行重复写入。

## 7. 工具验收

每个工具至少验证：
1. 正常调用；
2. 参数缺失与非法；
3. 认证失效；
4. 跨账号对象；
5. 下游超时；
6. 返回协议缺失；
7. 敏感字段裁剪；
8. 国家/语言上下文；
9. 重复调用和幂等；
10. 日志与trace关联。
