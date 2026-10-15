---
name: shengpay_platform_sharing_query_api_integration
description: 盛付通平台商分账结果查询接口的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 分账查询接口 (SKILL)

本 Skill 用于查询分账请求及每个接收方的最终处理结果，也是分账通知缺失或超时后的主动查询兜底接口。

## 1. 基础接口信息

- **集测环境接口:** `https://mchapitest.shengpay.com/sharing/querySharing`
- **生产环境接口:** `https://mchapi.shengpay.com/sharing/querySharing`
- **请求方式:** `POST application/json (UTF-8)`

## 2. 请求参数规范 (Request Parameters)

* `*mchId` [String(10)]: 盛付通商户号。
* `*mchSharingNo` [String(64)]: 发起分账时使用的商户分账单号。
* `*nonceStr` [String]: 随机字符串。
* `*signType` [String]: 官网示例使用 `RSA`。
* `*sign` [String]: 商户请求签名。

## 3. 响应参数规范 (Response Parameters)

* **通信与业务结果：**
  * `returnCode`、`returnMsg`、`resultCode`、`errorCode`、`errorCodeDes`。
* **分账主体：**
  * `mchId` [String(10)]: 商户号。
  * `mchSharingNo` [String(64)]: 商户分账单号。
  * `sharingNo` [String(64)]: 盛付通分账单号。
  * `transactionId` [String(64)]: 盛付通支付订单号。
  * `status` [String(10)]: 分账单状态，`PROCESSING`=处理中，`FINISHED`=处理完成。
* **接收方结果：**
  * `receivers` [String(1024)]: 接收方结果 JSON 字符串。
  * `receiverType` [String(10)]: `B`=商户，`C`=钱包会员。
  * `receiverId` [String(64)]: 商户号或钱包会员 `unionId`。
  * `amount` [int]: 分账金额，单位分。
  * `description` [String(64)]: 分账原因描述。
  * `sharingStatus` [String(16)]: `ING`=处理中，`SUCCESS`=成功，`FAIL`=失败。
  * `finishTime` [String(16)]: 分账完成时间，格式通常为 `yyyyMMddHHmmss`。
  * `failCode` [String(32)]: 分账失败枚举。
  * `failReason` [String(64)]: 分账失败描述。
* **安全鉴权：**
  * `nonceStr`、`signType`、`sign`：盛付通响应签名。

## 4. AI 执行步骤

1. **确认查询键**：只使用 `mchSharingNo` 查询，不使用支付订单号代替。
2. **发起签名请求**：公共参数加入后生成 RSA 签名。
3. **响应验签并解析 receivers**：先验签，再把 `receivers` JSON 字符串反序列化为接收方列表。
4. **判断分账单状态**：`PROCESSING` 时继续等待和轮询；`FINISHED` 后逐项判断 `sharingStatus`。
5. **更新本地业务**：终态更新必须幂等，保留 `sharingNo`、`finishTime`、`failCode` 和 `failReason`。

## 5. 状态处理建议

| 层级 | 状态 | 处理 |
| --- | --- | --- |
| 分账单 `status` | `PROCESSING` | 保持处理中，按退避策略查询或等待通知 |
| 分账单 `status` | `FINISHED` | 分账单处理结束，继续判断每个接收方结果 |
| 接收方 `sharingStatus` | `ING` | 当前接收方未终态，继续查询 |
| 接收方 `sharingStatus` | `SUCCESS` | 当前接收方分账成功 |
| 接收方 `sharingStatus` | `FAIL` | 当前接收方失败，记录 `failCode/failReason` |

## 6. 接入避坑指南与最佳实践

1. 查询接口也会返回 `SYSTEM_ERROR`，此时应使用相同 `mchSharingNo` 稍后重试，不能生成新分账单。
2. 分账单 `FINISHED` 不等于每个接收方都成功，必须检查嵌套 `receivers`。
3. 不要按接收方列表位置关联业务，应使用 `receiverType + receiverId` 建立映射。
4. 轮询应设置合理间隔和上限，避免高频调用触发风控。
5. 通知与查询可能同时到达，终态更新必须保持幂等。

## 7. 请求 Payload 示例

```json
{
  "mchId": "30057906",
  "mchSharingNo": "SHARE202610150001",
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
  "sharingNo": "S628952730081689600",
  "transactionId": "M20261015123456789012345678",
  "status": "FINISHED",
  "receivers": "[{\"amount\":200,\"description\":\"partner share\",\"finishTime\":\"20261015135457\",\"receiverId\":\"88021534\",\"receiverType\":\"B\",\"sharingStatus\":\"SUCCESS\"}]",
  "nonceStr": "PLACEHOLDER_RESPONSE_NONCE_0000",
  "signType": "RSA",
  "sign": "<响应签名占位符>"
}
```

## 9. 错误码

| 错误码 | 描述 | 处理建议 |
| --- | --- | --- |
| `SIGN_ERROR` | 签名错误 | 校验签名算法和参数 |
| `PARAMS_ERROR` | 参数问题 | 检查 `mchSharingNo` 格式 |
| `SYSTEM_ERROR` | 系统错误 | 使用相同参数重试 |
| `NO_AUTH` | 无此接口权限 | 申请接口权限 |
| `ORDER_NOT_EXIST` | 订单不存在 | 检查 `mchSharingNo` |


