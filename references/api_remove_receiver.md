---
name: shengpay_platform_sharing_remove_receiver_api_integration
description: 盛付通平台商分账删除接收方接口的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 删除分账接收方接口 (SKILL)

本 Skill 用于删除已添加的商户接收方。当前官网接口明确支持的接收方类型为商户 `receiverType=B`。

## 1. 基础接口信息

- **集测环境接口:** `https://mchapitest.shengpay.com/sharing/receiver/remove`
- **生产环境接口:** `https://mchapi.shengpay.com/sharing/receiver/remove`
- **请求方式:** `POST application/json (UTF-8)`

## 2. 请求参数规范 (Request Parameters)

* `*mchId` [String(10)]: 盛付通商户号。
* `*receiver` [String(1024)]: 分账接收方 JSON 字符串。
* `*receiverType` [String(2)]: 分账接收方类型，接口固定值 `B`。
* `*receiverId` [String(10)]: 接收方商户号。
* `*nonceStr` [String]: 随机字符串。
* `*signType` [String]: 官网示例使用 `RSA`。
* `*sign` [String]: 商户请求签名。

### receiver JSON 示例

```json
"{\"receiverType\":\"B\",\"receiverId\":\"880100552\"}"
```

## 3. 响应参数规范 (Response Parameters)

* `returnCode`、`returnMsg`、`resultCode`、`errorCode`、`errorCodeDes`。
* `mchId` [String(10)]: 商户号。
* `receiver` [String(1024)]: 分账接收方 JSON 字符串。
* `receiverId` [String(10)]: 接收方商户号。
* `receiverType` [String(2)]: 固定为 `B`。
* `nonceStr`、`signType`、`sign`: 盛付通响应签名。

## 4. AI 执行步骤

1. **确认业务影响**：删除后该商户接收方不能再用于新的分账。
2. **构建 receiver 字符串**：传入要删除的 `receiverType=B` 和 `receiverId`。
3. **签名发送并验签响应**。
4. **幂等更新本地关系**：删除成功后更新本地状态；不存在时结合业务决定是否视为已完成。
5. **处理后续分账风险**：已有分账单可能不受影响，新分账不得再使用已删除接收方。

## 5. 接入避坑指南与最佳实践

1. 删除前应确认不存在待处理的新分账计划，避免后续请求返回 `RECEIVER_INVALID`。
2. 删除接口不代表自动回退历史分账资金，两者是独立能力。
3. 重复删除可能返回 `RECEIVER_NOT_EXIST`，业务侧可按幂等目标决定是否忽略。
4. `receiver` 必须序列化为 JSON 字符串，不能直接发送对象。
5. 当前接口只记录 `receiverType=B`，不得将钱包会员接收方套用此请求。

## 6. 请求 Payload 示例

```json
{
  "mchId": "30057906",
  "receiver": "{\"receiverType\":\"B\",\"receiverId\":\"880100552\"}",
  "nonceStr": "PLACEHOLDER_NONCE_32_CHARS_000000",
  "signType": "RSA",
  "sign": "<RSA签名占位符>"
}
```

## 7. 响应 Payload 示例

```json
{
  "returnCode": "SUCCESS",
  "returnMsg": "OK",
  "resultCode": "SUCCESS",
  "mchId": "30057906",
  "receiver": "{\"receiverType\":\"B\",\"receiverId\":\"880100552\"}",
  "receiverId": "880100552",
  "receiverType": "B",
  "nonceStr": "PLACEHOLDER_RESPONSE_NONCE_0000",
  "signType": "RSA",
  "sign": "<响应签名占位符>"
}
```

## 8. 错误码

| 错误码 | 描述 | 处理建议 |
| --- | --- | --- |
| `SIGN_ERROR` | 签名错误 | 校验签名算法和参数 |
| `PARAMS_ERROR` | 参数问题 | 检查 `receiver` 字符串和字段 |
| `SYSTEM_ERROR` | 系统错误 | 使用相同参数再次调用 |
| `NO_AUTH` | 无此接口权限 | 申请接口权限 |
| `NO_SHARING_AUTH` | 无分账权限 | 先开通分账 |
| `RECEIVER_NOT_EXIST` | 分账接收方不存在 | 核对接收方或按幂等处理 |


