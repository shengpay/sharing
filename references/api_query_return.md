---
name: shengpay_platform_sharing_return_query_api_integration
description: 盛付通平台商分账回退结果查询接口的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 回退查询接口 (SKILL)

本 Skill 用于查询分账回退的最终状态，也是在回退通知缺失、网络超时或 `SYSTEM_ERROR` 后的主动查询兜底能力。

## 1. 基础接口信息

- **集测环境接口:** `https://mchapitest.shengpay.com/return/queryReturn`
- **生产环境接口:** `https://mchapi.shengpay.com/return/queryReturn`
- **请求方式:** `POST application/json (UTF-8)`

## 2. 请求参数规范 (Request Parameters)

* `*mchId` [String(10)]: 盛付通商户号。
* `*mchReturnNo` [String(64)]: 发起回退时使用的商户回退单号。
* `*nonceStr` [String]: 随机字符串。
* `*signType` [String]: 官网示例使用 `RSA`。
* `*sign` [String]: 商户请求签名。

## 3. 响应参数规范 (Response Parameters)

* **通信与业务结果：**
  * `returnCode`、`returnMsg`、`resultCode`、`errorCode`、`errorCodeDes`。
* **回退主体：**
  * `mchId` [String(10)]: 商户号。
  * `mchSharingNo` [String(64)]: 原商户分账单号。
  * `mchReturnNo` [String(64)]: 商户回退单号。
  * `returnNo` [String(64)]: 盛付通回退单号。
  * `returnStatus` [String(10)]: `SUCCESS`=成功，`FAIL`=失败，`ING`=处理中。
* **回退明细：**
  * `returnReceiverType` [String(16)]: `B`=商户，`C`=钱包会员。
  * `returnReceiverId` [String(64)]: 商户号或钱包会员 `unionId`。
  * `returnAmount` [int]: 回退金额，单位分。
  * `finishTime` [String(32)]: 回退完成时间，格式通常为 `yyyyMMddHHmmss`。
  * `failReason` [String(64)]: 失败原因，仅失败时有值。
* **安全鉴权：**
  * `nonceStr`、`signType`、`sign`。

## 4. AI 执行步骤

1. **确认查询键**：只使用 `mchReturnNo` 查询回退结果。
2. **签名请求**：加入公共参数并生成 RSA 签名。
3. **响应验签**：验证盛付通响应签名后再读取业务字段。
4. **判断状态**：`ING` 表示继续处理；`SUCCESS` 表示回退成功；`FAIL` 表示终态失败。
5. **幂等更新**：按 `mchReturnNo` 更新本地回退单，保留 `returnNo`、`finishTime` 和 `failReason`。

## 5. 状态处理建议

| `returnStatus` | 含义 | 建议 |
| --- | --- | --- |
| `ING` | 处理中 | 等待通知或按退避策略再次查询 |
| `SUCCESS` | 回退成功 | 更新本地终态并停止轮询 |
| `FAIL` | 回退失败 | 记录失败原因并转人工或业务补偿流程 |

## 6. 接入避坑指南与最佳实践

1. 查询接口的 `SYSTEM_ERROR` 不代表回退失败，应使用相同 `mchReturnNo` 稍后重试。
2. 不要用分账单号查询单笔回退结果，查询键必须是 `mchReturnNo`。
3. 查询可能早于通知到达，必须与通知使用同一幂等更新逻辑。
4. 轮询频率应合理，建议采用递增间隔，终态后立即停止。
5. 回退失败原因不能仅展示给用户，需记录用于对账和人工处理。

## 7. 请求 Payload 示例

```json
{
  "mchId": "30057906",
  "mchReturnNo": "RETURN202610150001",
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
  "mchId": "30057906",
  "mchSharingNo": "SHARE202610150001",
  "returnStatus": "SUCCESS",
  "finishTime": "20261015135458",
  "mchReturnNo": "RETURN202610150001",
  "returnNo": "R628953070625619968",
  "returnReceiverType": "B",
  "returnReceiverId": "88021534",
  "returnAmount": 100,
  "nonceStr": "PLACEHOLDER_RESPONSE_NONCE_0000",
  "signType": "RSA",
  "sign": "<响应签名占位符>"
}
```

## 9. 错误码

| 错误码 | 描述 | 处理建议 |
| --- | --- | --- |
| `SIGN_ERROR` | 签名错误 | 校验签名算法和参数 |
| `PARAMS_ERROR` | 参数问题 | 检查 `mchReturnNo` |
| `SYSTEM_ERROR` | 系统错误 | 使用相同参数重试 |
| `NO_AUTH` | 无此接口权限 | 申请接口权限 |
| `ORDER_NOT_EXIST` | 订单不存在 | 确认回退单号 |


