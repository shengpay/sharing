---
name: shengpay_platform_sharing_return_apply_api_integration
description: 盛付通平台商分账回退接口的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 发起回退接口 (SKILL)

本 Skill 用于对已成功分给接收方的资金发起分账回退。回退属于异步流程，返回受理成功不代表资金已经退回。

## 1. 基础接口信息

- **集测环境接口:** `https://mchapitest.shengpay.com/return/applyReturn`
- **生产环境接口:** `https://mchapi.shengpay.com/return/applyReturn`
- **请求方式:** `POST application/json (UTF-8)`

## 2. 请求参数规范 (Request Parameters)

* `*mchId` [String(10)]: 盛付通商户号。
* `*mchSharingNo` [String(64)]: 原商户分账单号。
* `*mchReturnNo` [String(64)]: 商户回退单号，在平台商系统内唯一，用作幂等键。
* `notifyUrl` [String(256)]: 退分账结果通知地址。
* `*returnReceiverType` [String(16)]: 回退方类型，`B`=商户，`C`=钱包会员。
* `*returnReceiverId` [String(64)]: `B` 时为商户号；`C` 时为钱包会员 `unionId`。
* `*returnAmount` [int]: 回退金额，单位分。
* `*returnDescription` [String(64)]: 回退描述。
* `*nonceStr` [String]: 随机字符串。
* `*signType` [String]: 官网示例使用 `RSA`。
* `*sign` [String]: 商户请求签名。

## 3. 响应参数规范 (Response Parameters)

* `returnCode`、`returnMsg`、`resultCode`、`errorCode`、`errorCodeDes`。
* `mchId` [String(10)]: 商户号。
* `mchSharingNo` [String(64)]: 原商户分账单号。
* `mchReturnNo` [String(64)]: 商户回退单号。
* `returnNo` [String(64)]: 盛付通回退单号。
* `nonceStr`、`signType`、`sign`: 盛付通响应签名。

## 4. AI 执行步骤

1. **确认原分账状态**：仅对允许回退且已成功的分账记录发起。
2. **锁定回退方**：`returnReceiverType` 与 `returnReceiverId` 必须对应原分账接收方。
3. **校验金额**：`returnAmount` 以分为单位，不能大于该接收方可回退金额。
4. **签名发送**：使用相同业务重试时复用 `mchReturnNo`。
5. **响应验签**：只有验签通过且 `resultCode=SUCCESS` 才视为受理成功。
6. **终态闭环**：调用回退查询或接收退分账结果通知；网络异常和 `SYSTEM_ERROR` 不得直判失败。

## 5. 状态与异常处理

- 受理成功：本地回退单状态设为处理中，记录 `returnNo`。
- `SYSTEM_ERROR` 或超时：调用 `/return/queryReturn` 查询 `mchReturnNo`。
- `DUPLICATE_ORDER`：查询原 `mchReturnNo` 的回退结果，不要生成新单号。
- `NOT_ALLOW_RETURN`：确认原分账是否达到可回退状态。
- `RETURN_OVERDUE`：确认回退金额是否超过该接收方已分账金额。

## 6. 接入避坑指南与最佳实践

1. 回退单号与分账单号是两个不同的幂等键，不能混用。
2. 回退请求返回成功仅为受理结果，用户侧金额到账必须等待终态。
3. 部分回退和多次回退应在商户侧记录累计回退金额，防止重复申请。
4. 回退方必须与原分账接收方一致，不能随意指定其他接收方。
5. 通知和查询同时使用时，终态写入必须幂等。

## 7. 请求 Payload 示例

```json
{
  "mchId": "30057906",
  "mchSharingNo": "SHARE202610150001",
  "mchReturnNo": "RETURN202610150001",
  "notifyUrl": "https://merchant.example.com/return/notify",
  "returnReceiverType": "B",
  "returnReceiverId": "88021534",
  "returnAmount": 100,
  "returnDescription": "business return",
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
  "mchReturnNo": "RETURN202610150001",
  "returnNo": "R657996019342508032",
  "nonceStr": "PLACEHOLDER_RESPONSE_NONCE_0000",
  "signType": "RSA",
  "sign": "<响应签名占位符>"
}
```

## 9. 错误码

| 错误码 | 描述 | 处理建议 |
| --- | --- | --- |
| `SIGN_ERROR` | 签名错误 | 验证签名算法和密钥 |
| `PARAMS_ERROR` | 参数问题 | 检查回退方和金额字段 |
| `SYSTEM_ERROR` | 系统错误 | 调用回退查询确认结果 |
| `NO_AUTH` | 无此接口权限 | 申请接口权限 |
| `ORDER_NOT_EXIST` | 订单不存在 | 检查原分账单号 |
| `DUPLICATE_ORDER` | 重复的订单 | 检查 `mchReturnNo` |
| `NOT_ALLOW_RETURN` | 原分账订单不允许回退 | 确认原分账状态 |
| `RETURN_OVERDUE` | 回退金额超限 | 调整回退金额 |


