---
name: shengpay_platform_sharing_return_result_notify_api_integration
description: 盛付通平台商退分账结果异步通知的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 退分账结果通知 (SKILL)

本 Skill 用于接收分账回退完成后由盛付通推送到商户 `notifyUrl` 的退分账结果通知。

## 1. 基础规范与机制

- **投递方式:** 盛付通向发起回退时指定的 `notifyUrl` 发送 `POST` 请求。
- **数据格式:** `application/json`
- **字符编码:** `UTF-8`
- **通知确认:** 成功处理本次通知后，直接返回纯文本大写 `SUCCESS`。
- **重试机制:** 返回非 `SUCCESS` 或处理失败时，盛付通会持续发送相同通知，累计最多 10 次且间隔逐步增加。
- **安全要求:** 必须使用盛付通公钥验证 `sign`。

## 2. 通知参数 (Notify Parameters)

* **通信与业务结果：**
  * `*returnCode` [String]: `SUCCESS` / `FAIL`。
  * `returnMsg` [String]: 通信消息。
  * `*resultCode` [String]: 业务结果，`SUCCESS` / `FAIL`。
  * `errorCode` / `errorCodeDes` [String]: 错误码和描述。
* **退分账主体：**
  * `mchId` [String(10)]: 商户号。
  * `mchSharingNo` [String(64)]: 原商户分账单号。
  * `mchReturnNo` [String(64)]: 商户回退单号。
  * `returnNo` [String(64)]: 盛付通回退单号。
  * `returnStatus` [String(10)]: `SUCCESS`=成功，`FAIL`=失败。
* **回退明细：**
  * `returnReceiverType` [String(16)]: `B`=商户，`C`=钱包会员。
  * `returnReceiverId` [String(64)]: 商户号或钱包会员 `unionId`。
  * `returnAmount` [int]: 回退金额，单位分。
  * `finishTime` [String(32)]: 回退完成时间，格式通常为 `yyyyMMddHHmmss`。
  * `failReason` [String(64)]: 失败原因，仅失败时有值。
* **安全鉴权：**
  * `*nonceStr` [String]: 随机字符串。
  * `*signType` [String]: `RSA`。
  * `*sign` [String]: 盛付通通知签名。

## 3. AI 执行步骤

1. **读取原始 JSON 到 Map**：保留全部字段用于验签。
2. **验证签名**：使用盛付通公钥验证通知真实性。
3. **幂等判断**：按 `mchReturnNo` 查询本地回退单；已处理终态时直接返回 `SUCCESS`。
4. **校验业务映射**：确认 `mchSharingNo`、`returnReceiverType`、`returnReceiverId` 与本地回退请求一致。
5. **更新终态**：处理 `returnStatus`、`returnAmount`、`finishTime` 和 `failReason`。
6. **返回 `SUCCESS`**：仅在本轮业务处理和持久化成功后返回。

## 4. 接入避坑指南与最佳实践

1. `returnStatus` 是回退终态判断字段，不能只看 `returnCode` 或 `resultCode`。
2. 通知可能重复，必须按 `mchReturnNo` 幂等。
3. 回退金额必须与本地累计回退金额核对，防止重复回退或金额异常。
4. 失败通知应记录 `failReason`，用于对账和人工处理。
5. 验签失败返回 `FAIL`，不要更新回退业务状态。
6. 通知缺失时使用 `/return/queryReturn` 主动查询兜底。

## 5. 通知 Payload 示例

```json
{
  "resultCode": "SUCCESS",
  "returnCode": "SUCCESS",
  "returnMsg": "OK",
  "mchId": "30057906",
  "mchSharingNo": "SHARE202610150001",
  "mchReturnNo": "RETURN202610150001",
  "returnNo": "R628953070625619968",
  "returnStatus": "SUCCESS",
  "finishTime": "20261015135458",
  "returnReceiverType": "B",
  "returnReceiverId": "88021534",
  "returnAmount": 100,
  "nonceStr": "PLACEHOLDER_NOTIFY_NONCE_000000",
  "signType": "RSA",
  "sign": "<通知签名占位符>"
}
```

## 6. 返回示例

```text
SUCCESS
```


