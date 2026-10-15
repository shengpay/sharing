---
name: shengpay_platform_sharing_apply_api_integration
description: 盛付通平台商分账发起接口的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 发起分账接口 (SKILL)

本 Skill 用于协助外部商户接入【平台商分账 - 发起分账接口】，将一个已支付订单中约定的资金分配给商户或钱包会员。

> 发起分账返回 `resultCode=SUCCESS` 仅代表分账请求被受理。接收方的最终分账结果必须以分账查询结果或已验签的分账结果通知为准。

## 1. 基础接口信息

- **集测环境接口:** `https://mchapitest.shengpay.com/sharing/applySharing`
- **生产环境接口:** `https://mchapi.shengpay.com/sharing/applySharing`
- **请求方式:** `POST`
- **数据格式:** `application/json`
- **字符编码:** `UTF-8`

## 2. 请求参数规范 (Request Parameters)

带 `*` 标识为必填参数。

* **分账主体与订单：**
  * `*mchId` [String(10)]: 盛付通商户号。
  * `*mchSharingNo` [String(64)]: 商户分账单号，在平台商系统内唯一，用作幂等键。
  * `*transactionId` [String(64)]: 盛付通支付订单号。
  * `*totalAmount` [int]: 分账总金额，必须等于全部接收方 `amount` 之和，单位分。
  * `notifyUrl` [String(256)]: 分账结果通知地址。
  * `devId` [String(64)]: 开发者账号；接收方列表中包含 `receiverType=C` 时必传。
* **接收方列表：**
  * `*receivers` [String(1024)]: 分账接收方列表 JSON 字符串，最多 5 项。
  * `*receiverType` [String(10)]: `B`=商户接收方，`C`=钱包会员接收方。
  * `*receiverId` [String(64)]: `B` 时为商户号；`C` 时为钱包会员 `unionId`。
  * `*amount` [int]: 当前接收方分账金额，单位分。
  * `*description` [String(64)]: 分账描述。
* **安全参数：**
  * `*nonceStr` [String]: 随机字符串。
  * `*signType` [String]: 官网示例使用 `RSA`。
  * `*sign` [String]: 使用商户私钥生成的签名。

### receivers JSON 示例

```json
"[{\"amount\":200,\"description\":\"partner share\",\"receiverId\":\"88021534\",\"receiverType\":\"B\"}]"
```

注意 `receivers` 本身是字符串，不是数组对象。使用 Java 时先序列化 `List<Receiver>`，再把结果字符串放入请求。

## 3. 响应参数规范 (Response Parameters)

* **通信与业务结果：**
  * `*returnCode` [String]: `SUCCESS` / `FAIL`。
  * `returnMsg` [String]: 通信失败描述。
  * `*resultCode` [String]: 业务受理结果，`SUCCESS` / `FAIL`。
  * `errorCode` / `errorCodeDes` [String]: 业务失败错误码和描述。
* **订单标识：**
  * `mchId` [String(10)]: 商户号。
  * `mchSharingNo` [String(64)]: 商户分账单号。
  * `transactionId` [String(64)]: 盛付通支付订单号。
  * `sharingNo` [String(64)]: 盛付通分账单号。
* **安全鉴权：**
  * `nonceStr`、`signType`、`sign`: 响应随机串和盛付通签名。

## 4. AI 执行步骤

1. **校验业务前置**：确认支付订单已支付、已结算并允许分账，接收方已满足要求。
2. **构建并校验金额**：所有金额均为分；先校验 `totalAmount == sum(receiver.amount)`。
3. **构建接收方 JSON 字符串**：最多 5 项；含 `receiverType=C` 时校验 `devId`。
4. **签名并发送**：使用商户号、随机串、`signType=RSA` 和全部非空业务字段生成签名。
5. **响应验签**：使用盛付通公钥验签，再判断 `returnCode` 和 `resultCode`。
6. **终态闭环**：受理成功后依据 `mchSharingNo` 接收通知或调用分账查询；网络异常或 `SYSTEM_ERROR` 时使用相同 `mchSharingNo` 查询。

## 5. 状态与异常处理

- `resultCode != SUCCESS`：根据 `errorCode` 修正请求；不要自动生成新的 `mchSharingNo` 重试同一业务。
- `SYSTEM_ERROR`：调用 `/sharing/querySharing` 查询 `mchSharingNo`。
- 网络超时：记录分账请求为处理中，调用查询接口确认，不得直接置为失败。
- `DUPLICATE_ORDER`：确认是否重复使用 `mchSharingNo`，并查询已有分账单状态。
- `NOT_ENOUGH`：重新查询订单待分账金额并调整分账金额。
- `RECEIVER_INVALID`：确认商户接收方已添加，钱包接收方参数正确。

## 6. 接入避坑指南与最佳实践

1. `totalAmount` 与接收方金额之和必须完全一致，否则会因参数错误失败。
2. 一次最多 5 个接收方，接收方列表长度和结构必须在商户侧提前校验。
3. 相同业务重试必须复用同一个 `mchSharingNo`，不能因超时生成新单号。
4. 响应必须验签，不能只判断 HTTP 200。
5. 钱包会员接收方必须传 `unionId` 对应的 `receiverId`，列表含 `C` 时还必须传 `devId`。
6. 通知可能重复到达，业务侧必须按 `mchSharingNo` 幂等处理。

## 7. 请求 Payload 示例

```json
{
  "mchId": "30057906",
  "mchSharingNo": "SHARE202610150001",
  "transactionId": "M20261015123456789012345678",
  "totalAmount": 200,
  "notifyUrl": "https://merchant.example.com/sharing/notify",
  "receivers": "[{\"amount\":200,\"description\":\"partner share\",\"receiverId\":\"88021534\",\"receiverType\":\"B\"}]",
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
  "transactionId": "M20261015123456789012345678",
  "sharingNo": "S657993611963006976",
  "nonceStr": "PLACEHOLDER_RESPONSE_NONCE_0000",
  "signType": "RSA",
  "sign": "<响应签名占位符>"
}
```

## 9. 错误码

| 错误码 | 描述 | 处理建议 |
| --- | --- | --- |
| `SIGN_ERROR` | 签名错误 | 验证签名算法、排序和密钥 |
| `PARAMS_ERROR` | 参数问题 | 检查字段类型、长度和必填性 |
| `SYSTEM_ERROR` | 系统错误 | 调用分账查询确认业务结果 |
| `NO_AUTH` | 无此接口权限 | 申请接口权限 |
| `NO_SHARING_AUTH` | 无分账权限 | 先开通分账产品 |
| `RECEIVER_INVALID` | 分账接收方非法 | 分账前添加并确认接收方 |
| `NOT_ENOUGH` | 可分账余额不足 | 查询剩余待分金额并调整 |
| `DUPLICATE_ORDER` | 重复的订单 | 检查 `mchSharingNo` 是否重复 |
| `ORDER_NOT_EXIST` | 订单不存在 | 检查 `transactionId` |
| `ORDER_NOT_READY` | 订单未结算 | 等待结算后再分账 |
| `NOT_SHARING_ORDER` | 非分账订单 | 使用支持分账的下单参数 |
| `ORDER_STATUS_ABNORMAL` | 订单状态异常 | 先确认支付订单状态 |
| `RATIO_AMOUNT_OVERDUE` | 超出分账比例上限 | 调整接收方分账金额 |
| `SHARING_PERIOD_OVERDUE` | 订单已超过可分账周期 | 确认订单时间，不再重试 |


