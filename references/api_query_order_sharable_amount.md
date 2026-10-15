---
name: shengpay_platform_sharing_query_order_sharable_amount_api_integration
description: 盛付通平台商分账查询订单待分账金额接口的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 查询订单待分账金额接口 (SKILL)

本 Skill 用于在发起分账前查询订单剩余可分账金额，降低因余额不足导致分账失败的风险。

## 1. 基础接口信息

- **集测环境接口:** `https://mchapitest.shengpay.com/sharing/queryOrderSharableAmount`
- **生产环境接口:** `https://mchapi.shengpay.com/sharing/queryOrderSharableAmount`
- **请求方式:** `POST application/json (UTF-8)`

## 2. 请求参数规范 (Request Parameters)

* `*mchId` [String(10)]: 盛付通商户号。
* `*transactionId` [String(64)]: 盛付通支付订单号。
* `*nonceStr` [String]: 随机字符串。
* `*signType` [String]: 官网示例使用 `RSA`。
* `*sign` [String]: 商户请求签名。

## 3. 响应参数规范 (Response Parameters)

* **通信与业务结果：**
  * `returnCode`、`returnMsg`、`resultCode`、`errorCode`、`errorCodeDes`。
* **订单金额：**
  * `mchId` [String(10)]: 商户号。
  * `sharableAmount` [int]: 订单剩余待分金额，单位分。
  * `transactionId` [String(64)]: 盛付通支付订单号。
* **安全鉴权：**
  * `nonceStr`、`signType`、`sign`。

## 4. AI 执行步骤

1. **确认订单支持分账**：使用盛付通支付订单号 `transactionId` 查询。
2. **签名发送**：加入公共参数并生成签名。
3. **响应验签**：验签通过后读取 `sharableAmount`。
4. **前置校验**：分账 `totalAmount` 不得超过查询到的 `sharableAmount`。
5. **并发保护**：多笔分账并发时，本地业务仍需做额度锁定或串行控制，不能只依赖一次查询。

## 5. 状态与异常处理

- `ORDER_NOT_READY`：订单尚未结算，等待后重新查询。
- `NOT_SHARING_ORDER`：订单不是分账订单，需确认下单参数。
- `ORDER_STATUS_ABNORMAL`：订单未支付、已失败或状态不允许分账。
- `SHARING_PERIOD_OVERDUE`：订单超过可分账周期，应停止分账。
- `SYSTEM_ERROR`：使用相同参数重试查询。

## 6. 接入避坑指南与最佳实践

1. `sharableAmount` 是当前时点剩余额度，不是订单初始可分配额度。
2. 查询与真正发起分账之间存在时间差，需要在分账失败时处理 `NOT_ENOUGH`。
3. 多接收方分账时，待分金额必须覆盖所有接收方金额之和。
4. 金额单位为分，禁止混入元或小数。
5. 查询不产生资金变动，可以安全用于分账前校验和人工排查。

## 7. 请求 Payload 示例

```json
{
  "mchId": "30057907",
  "transactionId": "M20261015123456789012345678",
  "nonceStr": "PLACEHOLDER_NONCE_32_CHARS_000000",
  "signType": "RSA",
  "sign": "<RSA签名占位符>"
}
```

## 8. 响应 Payload 示例

```json
{
  "returnCode": "SUCCESS",
  "returnMsg": "OK",
  "resultCode": "SUCCESS",
  "mchId": "30057907",
  "transactionId": "M20261015123456789012345678",
  "sharableAmount": 99,
  "nonceStr": "PLACEHOLDER_RESPONSE_NONCE_0000",
  "signType": "RSA",
  "sign": "<响应签名占位符>"
}
```

## 9. 错误码

| 错误码 | 描述 | 处理建议 |
| --- | --- | --- |
| `SIGN_ERROR` | 签名错误 | 校验签名算法和参数 |
| `PARAMS_ERROR` | 参数问题 | 检查 `transactionId` |
| `SYSTEM_ERROR` | 系统错误 | 使用相同参数重试 |
| `NO_AUTH` | 无此接口权限 | 申请接口权限 |
| `ORDER_NOT_EXIST` | 订单不存在 | 检查支付订单号 |
| `ORDER_NOT_READY` | 订单处理中 | 等待结算后重试 |
| `NOT_SHARING_ORDER` | 非分账订单 | 使用支持分账的下单参数 |
| `ORDER_STATUS_ABNORMAL` | 订单状态异常 | 确认订单支付状态 |
| `SHARING_PERIOD_OVERDUE` | 订单已超过可分账周期 | 不再发起分账 |


