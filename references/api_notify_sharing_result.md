---
name: shengpay_platform_sharing_result_notify_api_integration
description: 盛付通平台商分账结果异步通知的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 分账结果通知 (SKILL)

本 Skill 用于接收分账完成后由盛付通推送到商户 `notifyUrl` 的分账结果通知。通知包含每个接收方的最终结果。

## 1. 基础规范与机制

- **投递方式:** 盛付通向发起分账时指定的 `notifyUrl` 发送 `POST` 请求。
- **数据格式:** `application/json`
- **字符编码:** `UTF-8`
- **通知确认:** 成功处理本次通知后，直接返回纯文本大写 `SUCCESS`。
- **重试机制:** 返回非 `SUCCESS` 或处理失败时，盛付通会持续发送相同通知，累计最多 10 次且间隔逐步增加。
- **安全要求:** 必须使用盛付通公钥验证 `sign`，不得跳过验签。

## 2. 通知参数 (Notify Parameters)

* **通信与业务结果：**
  * `*returnCode` [String]: `SUCCESS` / `FAIL`。
  * `returnMsg` [String]: 通信消息。
  * `*resultCode` [String]: 业务结果，`SUCCESS` / `FAIL`。
  * `errorCode` / `errorCodeDes` [String]: 错误码和描述。
* **分账主体：**
  * `mchId` [String(10)]: 商户号。
  * `mchSharingNo` [String(64)]: 商户分账单号。
  * `status` [String(10)]: 分账单状态，官网通知中为 `FINISHED`。
  * `sharingNo` [String(64)]: 盛付通分账单号。
  * `transactionId` [String(64)]: 盛付通支付订单号。
* **接收方结果：**
  * `receivers` [String(1024)]: 接收方结果 JSON 字符串。
  * `receiverType` [String(10)]: `B`=商户，`C`=钱包会员。
  * `receiverId` [String(64)]: 商户号或钱包会员 `unionId`。
  * `amount` [int]: 分账金额，单位分。
  * `description` [String(64)]: 分账原因描述。
  * `sharingStatus` [String(16)]: `SUCCESS`=成功，`FAIL`=失败。
  * `finishTime` [String(16)]: 完成时间，格式通常为 `yyyyMMddHHmmss`。
  * `failCode` [String(32)]: 分账失败枚举。
  * `failReason` [String(64)]: 分账失败描述。
* **安全鉴权：**
  * `*nonceStr` [String]: 随机字符串。
  * `*signType` [String]: `RSA`。
  * `*sign` [String]: 盛付通通知签名。

## 3. AI 执行步骤

1. **读取原始 JSON 到 Map**：不要一开始就用 DTO 反序列化，避免盛付通新增字段导致验签失败。
2. **验证签名**：取出 `sign`，使用盛付通公钥和剩余全部非空字段验签。
3. **幂等判断**：按 `mchSharingNo` 查询本地分账单；已处理终态时直接返回 `SUCCESS`。
4. **解析 receivers**：将 JSON 字符串反序列化，按 `receiverType + receiverId` 匹配本地接收方。
5. **更新业务终态**：逐项处理 `sharingStatus`、`finishTime`、`failCode`、`failReason`。
6. **返回 `SUCCESS`**：业务提交成功且幂等保护完成后，返回纯文本 `SUCCESS`。

## 4. 接入避坑指南与最佳实践

1. 通知内 `resultCode=SUCCESS` 表示本次通知业务结果有效，最终资金结果仍需检查每个接收方的 `sharingStatus`。
2. `receivers` 是 JSON 字符串，内部每个接收方可能有独立的成功或失败状态。
3. 通知可能重复或乱序，必须使用数据库唯一约束、状态机或分布式幂等保障。
4. 验签失败应返回 `FAIL` 并记录原始报文摘要，不应处理业务。
5. 不要让 Web 框架自动包装返回值为 JSON；确认响应体严格为 `SUCCESS`。
6. 建议同时保留分账查询兜底，防止通知长期缺失。

## 5. 通知 Payload 示例

```json
{
  "mchId": "30057906",
  "mchSharingNo": "SHARE202610150001",
  "nonceStr": "PLACEHOLDER_NOTIFY_NONCE_000000",
  "receivers": "[{\"amount\":200,\"description\":\"partner share\",\"finishTime\":\"20261015111205\",\"receiverId\":\"88021534\",\"receiverType\":\"B\",\"sharingStatus\":\"SUCCESS\"}]",
  "resultCode": "SUCCESS",
  "returnCode": "SUCCESS",
  "returnMsg": "OK",
  "sharingNo": "S898528644829040640",
  "sign": "<通知签名占位符>",
  "signType": "RSA",
  "status": "FINISHED",
  "transactionId": "M20261015123456789012345678"
}
```

## 6. 返回示例

```text
SUCCESS
```


