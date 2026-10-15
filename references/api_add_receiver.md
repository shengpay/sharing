---
name: shengpay_platform_sharing_add_receiver_api_integration
description: 盛付通平台商分账添加接收方接口的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 添加分账接收方接口 (SKILL)

本 Skill 用于在发起分账前添加商户接收方。当前官网接口明确支持的接收方类型为商户 `receiverType=B`。

## 1. 基础接口信息

- **集测环境接口:** `https://mchapitest.shengpay.com/sharing/receiver/add`
- **生产环境接口:** `https://mchapi.shengpay.com/sharing/receiver/add`
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

不要把 `receiver` 直接写成 JSON 对象。使用 Java 时先构建接收方 DTO，再序列化为字符串。

## 3. 响应参数规范 (Response Parameters)

* `returnCode`、`returnMsg`、`resultCode`、`errorCode`、`errorCodeDes`。
* `mchId` [String(10)]: 商户号。
* `receiver` [String(1024)]: 分账接收方 JSON 字符串。
* `receiverId` [String(10)]: 接收方商户号。
* `receiverType` [String(2)]: 固定为 `B`。
* `nonceStr`、`signType`、`sign`: 盛付通响应签名。

## 4. AI 执行步骤

1. **确认接收方类型**：当前主动添加接口只接受商户接收方 `B`。
2. **构建 receiver 字符串**：字段至少包含 `receiverType=B` 和 `receiverId`。
3. **签名发送**：按公共签名规则生成请求。
4. **响应验签**：验签后判断 `returnCode` 和 `resultCode`。
5. **持久化关系**：记录商户接收方关系，供后续分账前校验和告警。

## 5. 接入避坑指南与最佳实践

1. 添加接收方是分账前置步骤，未添加的商户接收方在分账时可能返回 `RECEIVER_INVALID`。
2. `receiver` 是 JSON 字符串，转义错误会触发参数异常或业务异常。
3. 重复添加应根据接口返回和自身数据做幂等处理。
4. 接收方关系应带业务含义保存，不要只保存一个无上下文的商户号。
5. 钱包会员 `C` 的添加方式不在当前主动添加接口文档中，不得套用 `B` 的请求结构。

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
| `RECEIVER_ID_NOT_EXIST` | 分账接收方标识不存在 | 确认接收方商户号 |


