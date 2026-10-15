---
name: shengpay_platform_sharing_auto_unfreeze_notify_api_integration
description: 盛付通平台商分账自动解冻异步通知的接入引导与规范
user-invocable: true
---

# 盛付通平台商分账 - 自动解冻通知 (SKILL)

本 Skill 用于接收订单超过可分账时间后发生的自动解冻通知。官网当前仅提供通知字段和报文示例，未提供签名字段及明确的确认返回说明。

> **协议边界：** 本通知官网示例中没有 `nonceStr`、`signType`、`sign`，也没有说明要求商户返回 `SUCCESS`。不得虚构验签逻辑、重试承诺或成功确认协议；如果业务安全要求较高，应向盛付通技术支持确认当前环境的完整接入协议。

## 1. 基础规范

- **投递方式:** 盛付通向商户接收地址发送 `POST` 请求。
- **数据格式:** `application/json`
- **字符编码:** `UTF-8`
- **业务含义:** 订单已超过可分账时间，尚未使用的分账资金自动解冻。

## 2. 通知参数 (Notify Parameters)

* **分账主体：**
  * `mchId` [String(10)]: 商户号。
  * `mchSharingNo` [String(64)]: 商户分账单号。
  * `status` [String(10)]: 分账单状态，官网示例为 `FINISHED`。
  * `sharingNo` [String(64)]: 盛付通分账单号。
  * `transactionId` [String(64)]: 盛付通支付订单号。
* **接收方信息：**
  * `receivers` [String(1024)]: 接收方结果 JSON 字符串。
  * `receiverType` [String(10)]: 接收方类型，官网字段说明为 `B`=商户。
  * `receiverId` [String(64)]: 商户接收方标识。
  * `amount` [int]: 金额，单位分。
  * `description` [String(64)]: 自动解冻说明。
  * `sharingStatus` [String(16)]: 官网示例为 `SUCCESS`。
  * `finishTime` [String(16)]: 完成时间，格式通常为 `yyyyMMddHHmmss`。

## 3. AI 执行步骤

1. **读取并保存原始报文**：不要丢弃未知字段，便于后续排查和官网协议核对。
2. **不执行未定义验签**：当前通知没有 `sign` 字段，代码不得伪造验签通过，也不得在文档中声称已验签。
3. **按业务键幂等**：使用 `mchSharingNo` 或更具体的 `sharingNo + transactionId` 防止重复处理。
4. **解析 receivers**：将 JSON 字符串转换为接收方列表。
5. **更新自动解冻状态**：记录解冻金额、完成时间和接收方信息。
6. **谨慎选择响应**：由于官网未说明确认返回协议，返回策略应根据商户网关或技术支持确认；不要在接口注释中声称返回 `SUCCESS` 可控制重试。

## 4. 接入避坑指南与最佳实践

1. 自动解冻表示资金未按原计划分配，可能影响后续分账，应通知业务监控或对账流程。
2. `receivers` 为 JSON 字符串，内部字段需单独反序列化。
3. 不要将自动解冻通知当作分账成功通知处理。
4. 当前文档缺少签名与响应确认说明，生产接入前应确认是否存在环境差异或补充安全方案。
5. 日志中避免输出私钥、完整签名或敏感商户资料。

## 5. 通知 Payload 示例

```json
{
  "mchId": "88042234",
  "mchSharingNo": "SHARE202610150002",
  "receivers": "[{\"amount\":29925,\"description\":\"超过可分账时间，自动解冻分账\",\"finishTime\":\"20261015173004\",\"receiverId\":\"88042214\",\"receiverType\":\"B\",\"sharingStatus\":\"SUCCESS\"}]",
  "sharingNo": "SP1095037986565935104",
  "status": "FINISHED",
  "transactionId": "M20261015105904522942504970"
}
```

## 6. 响应说明

官网页面未提供自动解冻通知的 HTTP 响应确认规则，因此本 Skill 不提供“返回 `SUCCESS` 即确认成功”的承诺。实际接收端应以盛付通技术支持确认的协议为准，并确保重复通知不会重复解冻或重复记账。


