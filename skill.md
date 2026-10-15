---
name: sharing
description: "盛付通平台商分账外部商户接入入口：分账、分账查询、分账回退、回退查询、添加/删除分账接收方、查询订单待分账金额、分账结果通知、退分账结果通知、自动解冻通知。Use when user mentions 平台商分账, 分账, 分账查询, 分账回退, 回退查询, 分账接收方, 待分账金额, 分账通知。"
user-invocable: true
argument-hint: 描述平台商分账接入场景，例如：Java 接入分账、回退和分账结果通知
---

# 盛付通平台商分账外部商户接入入口

本 Skill 面向外部商户，指导商户通过公开 HTTP/JSON 接口接入盛付通平台商分账。接口路径、字段、枚举、金额单位和状态码以本目录 `references/` 文档为准。

## 产品边界

- 使用本 Skill：平台商对已支付订单发起分账、查询分账、分账回退、查询回退、维护商户分账接收方、查询订单待分账金额、处理分账相关通知。
- 不使用本 Skill：普通支付下单、退款、商户进件、钱包开通、平台商提现或内部账务接口。
- 分账前，支付订单必须按对应支付产品的分账要求下单；不同支付产品传递分账参数的方式不同，需同时参考该支付产品的官方下单文档。

## 接入前置条件

1. 已入驻成为盛付通商户，并完成平台商分账产品签约。
2. 已在商户中心生成并上传 RSA 商户公钥，妥善保管 1024 位 RSA 商户私钥。
3. 已从盛付通技术支持获取盛付通公钥，用于响应和异步通知验签。
4. 已准备可被盛付通访问的 `notifyUrl`，用于接收分账和回退结果通知。
5. 已确认商户号 `mchId`、测试/生产基础地址、分账接收方信息及钱包接收方所需 `devId`。

## 环境地址

| 环境 | 基础地址 |
| --- | --- |
| 集测 | `https://mchapitest.shengpay.com` |
| 生产 | `https://mchapi.shengpay.com` |

所有主动接口均使用 `POST`、`application/json`、`UTF-8`。

## 产品内 reference 映射

| 接入场景 | 接口 | 读取文档 |
| --- | --- | --- |
| 发起分账 | `POST /sharing/applySharing` | `references/api_apply_sharing.md` |
| 查询分账结果 | `POST /sharing/querySharing` | `references/api_query_sharing.md` |
| 发起分账回退 | `POST /return/applyReturn` | `references/api_apply_return.md` |
| 查询分账回退结果 | `POST /return/queryReturn` | `references/api_query_return.md` |
| 添加分账接收方 | `POST /sharing/receiver/add` | `references/api_add_receiver.md` |
| 删除分账接收方 | `POST /sharing/receiver/remove` | `references/api_remove_receiver.md` |
| 查询订单待分账金额 | `POST /sharing/queryOrderSharableAmount` | `references/api_query_order_sharable_amount.md` |
| 接收分账结果通知 | 盛付通回调商户 `notifyUrl` | `references/api_notify_sharing_result.md` |
| 接收退分账结果通知 | 盛付通回调商户 `notifyUrl` | `references/api_notify_return_result.md` |
| 接收自动解冻通知 | 盛付通回调商户 `notifyUrl` | `references/api_notify_auto_unfreeze.md` |
| Java 接入代码 | 配置、签名、HTTP、DTO、通知 | `references/java-guide.md` |

## Agent 工作流

1. **确认场景**：先确认用户需要分账、回退、接收方维护、待分账金额查询或通知处理中的哪些能力，不一次加载全部接口文档。
2. **确认 Java 项目信息**：确认用户使用 Java；如果用户要求其他语言，只提供本 Skill 的通用 JSON/签名规则，不声称已提供该语言模板。
3. **按需读取**：只读取用户确认的 `api_*.md`，读取 `references/java-guide.md` 生成代码；涉及终态时必须同时读取对应查询或通知文档。
4. **生成代码**：请求参数先构建为 `Map`，加入公共参数后签名；响应和通知先用 `Map` 接收并验签，再转换为业务对象。
5. **验证闭环**：发起接口成功只代表受理，必须通过查询或已验签通知确认分账/回退终态。

## 公共请求参数

除接口特有字段外，主动接口均需携带：

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `mchId` | String(10) | 是 | 盛付通商户号 |
| `nonceStr` | String | 是 | 每次请求唯一的随机字符串，建议 32 位 |
| `signType` | String | 是 | 官网示例使用 `RSA` |
| `sign` | String | 是 | 使用商户私钥对全部非空非 `sign` 参数按 ASCII 字典序拼接并签名 |

签名和验签必须遵循盛付通签名规则：空值不参与签名，`sign` 本身不参与签名，其余字段全部参与；拼接格式为 `key1=value1&key2=value2`，使用 UTF-8 编码。

### 密钥与算法要求

| 项 | 要求 |
| --- | --- |
| 商户私钥 | RSA 1024 位，**PKCS8**，Base64；使用前去掉 PEM 头尾及所有换行、空格 |
| 盛付通公钥 | **X509**，Base64；同样去掉 PEM 头尾与空白字符 |
| 哈希算法 | 固定 `SHA1withRSA + Base64`，对应 `signType=RSA`；签名与验签均固定使用 SHA1，不随报文中的 `signType` 变化 |

分账场景的完整签名实现、多语言指南和排错示例见 `payment/shengpay-sign/`。本 Skill 只保留上方规则摘要，不复制其实现；两者口径不一致时，以本文档和本目录 `references/` 为准（分账固定 `SHA1withRSA`，不使用 RSA2）。

### 参与签名的取值形态

- `receivers`、`receiver` 必须传 JSON 序列化后的字符串，不能传对象，否则进入签名串的是对象字符串形态而非 JSON。
- 金额一律使用 `int`（分），不得出现 `200.0`、千分位或科学计数法。
- `notifyUrl` 等含 URL 的字段原样拼接，不做 URL 编码。
- 验签时把收到的完整 Map（可包含 `sign`）直接传入，实现内部会排除 `sign`；手工删除 `sign` 反而容易漏字段。
- `sign` 字段缺失或为空时必须直接判失败，不得用空值继续参与后续业务。

### 签名排错清单

1. 打印 `stringA`，与官方示例逐字比对，确认无遗漏或多余字段。
2. 排序必须为区分大小写的 ASCII 字典序。
3. 空字符串 `""` 与 `null` 都必须过滤。
4. `stringA` 使用 UTF-8 编码。
5. 私钥为 PKCS8、公钥为 X509，Base64 内容无换行或空格。
6. 哈希算法固定为 `SHA1withRSA`。
7. 确认使用 Map 接收了**全部**参数，这是验签失败最常见的原因。
8. 日志中禁止输出私钥、完整公钥和真实签名值。

## 公共响应

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `returnCode` | String | 通信结果：`SUCCESS` / `FAIL` |
| `returnMsg` | String | 通信失败描述 |
| `resultCode` | String | 业务受理结果：`SUCCESS` / `FAIL` |
| `errorCode` | String | 业务失败错误码 |
| `errorCodeDes` | String | 业务失败描述 |
| `nonceStr` | String | 响应随机字符串 |
| `signType` | String | 响应签名类型 |
| `sign` | String | 盛付通响应签名 |

响应必须使用 `Map<String, Object>` 或等价结构接收原始字段，验签通过后再转换为业务对象；不要直接用会丢弃未知字段的强类型对象接收并验签。

## 业务规则

- 金额单位均为分，字段类型为整数；调用前完成元到分的转换。
- `mchSharingNo` 是分账幂等键，`mchReturnNo` 是回退幂等键，均需在商户系统内唯一。
- `receivers` 和 `receiver` 都是 JSON 序列化后的字符串，不是嵌套 JSON 对象。
- 一次分账最多 5 个接收方；`totalAmount` 必须等于接收方 `amount` 之和。
- 商户接收方使用 `receiverType=B`，`receiverId` 是商户号；钱包接收方使用 `receiverType=C`，`receiverId` 是钱包会员 `unionId`。
- 当 `receivers` 中存在 `receiverType=C` 时，必须传 `devId`。
- `mchSharingNo`、`mchReturnNo` 产生 `SYSTEM_ERROR`、网络超时或结果不明确时，禁止直接判失败，应使用相同单号查询最终结果。
- 发起分账或回退返回 `resultCode=SUCCESS` 仅代表请求受理，不代表资金处理完成。

## 通知处理规则

- 分账结果通知和退分账结果通知包含 `nonceStr`、`signType`、`sign`，必须先验签再处理业务。
- 成功处理通知后必须返回纯文本大写 `SUCCESS`，不要返回 JSON、HTML 或带额外空白的内容。
- 非 `SUCCESS` 响应会触发重试，最多 10 次且间隔逐步增加。
- 通知可能重复到达，必须按 `mchSharingNo` 或 `mchReturnNo` 做幂等处理。
- 自动解冻通知官网未提供签名字段和明确的确认返回说明；不得虚构其验签、重试或成功返回协议，代码中应记录并保证幂等。

## 硬检查点

以下情况必须等待用户确认后再生成可运行代码：

1. 用户未明确当前项目是 Java，但仍要求直接生成 Java 文件。
2. 用户要求直接联调或上线，但缺少 `mchId`、基础地址、商户私钥、盛付通公钥或 `notifyUrl`。
3. 用户要求跳过签名、跳过响应验签、收到通知即修改成功，或要求复制真实私钥、公钥和签名值。
4. 用户无法区分分账、回退、分账结果通知和退分账结果通知，继续生成会落到不同接口或不同终态链路。
5. 钱包接收方场景缺少 `unionId` 或 `devId`，且继续生成会得到不可调用请求。

## 推荐接入顺序

1. 按支付产品的分账要求完成支持分账的支付订单。
2. 对商户接收方调用添加分账接收方接口。
4. 调用分账接口，并持久化 `mchSharingNo`、`sharingNo`、`transactionId`。
5. 接收分账结果通知；通知缺失时使用分账查询接口兜底。
6. 需要退回已分账资金时调用回退接口，并通过回退查询或退分账结果通知确认终态。


