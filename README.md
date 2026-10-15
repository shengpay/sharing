# 盛付通平台商分账 Skill 包

面向 AI Agent 的盛付通平台商分账接入知识库，覆盖分账、分账查询、分账回退、回退查询、分账接收方维护、订单待分账金额查询与分账相关异步通知。

开发者用自然语言描述目标，Skill 会协助定位接口资料、锁定分账接收方与金额参数映射、生成受控代码示例、给出联调排查与上线检查建议。它不是只生成一次代码的模板包：面对首次接入、存量项目改造、接收方参数核对、签名排查或分账回退闭环，会先识别当前阶段与分账场景（商户接收方 / 钱包会员接收方），再提供相应的最小资料集与下一步建议。

## 覆盖能力

| 能力域 | 接口 | 说明 |
| --- | --- | --- |
| 发起分账 | `POST /sharing/applySharing` | 对已支付订单将约定资金分配给商户或钱包会员；`receivers` 为 JSON 字符串、最多 5 项 |
| 分账查询 | `POST /sharing/querySharing` | 确认分账单与每个接收方的最终结果；`status` 为 `PROCESSING` / `FINISHED`，`sharingStatus` 为 `ING` / `SUCCESS` / `FAIL` |
| 发起分账回退 | `POST /return/applyReturn` | 退回已分账资金；`returnStatus` 是终态来源 |
| 分账回退查询 | `POST /return/queryReturn` | 确认回退终态 |
| 接收方维护 | `POST /sharing/receiver/add`、`POST /sharing/receiver/remove` | 商户接收方增删；`receiver` 为 JSON 字符串 |
| 待分账金额查询 | `POST /sharing/queryOrderSharableAmount` | 查询订单剩余可分账金额 |
| 异步通知 | 回调商户 `notifyUrl` | 分账结果通知、退分账结果通知、自动解冻通知 |

环境地址：

| 环境 | 基础地址 |
| --- | --- |
| 集测 | `https://mchapitest.shengpay.com` |
| 生产 | `https://mchapi.shengpay.com` |

所有主动接口均使用 `POST`、`application/json`、`UTF-8`。

## 快速安装

### 通过 Agent 对话安装

将以下描述复制粘贴到任意 AI，它将引导您完成安装。

```text
根据 https://github.com/shengpay/sharing 安装skill
```


## 使用方式

- **总入口**：`skill.md`（产品边界、接入前置、reference 映射、Agent 工作流、硬检查点）
- **接口口径**：接口路径、请求/响应字段、枚举、金额单位（分）、状态码一律以 `references/` 文档为准
- **签名验签**：分账场景共用盛付通统一签名规则，固定 `SHA1withRSA + Base64`
- **验签铁律**：先用 Map 接收全部原始参数 → 验签通过 → 再反序列化为业务对象

## 目录结构

```text
sharing/
├─ skill.md                    # AI 加载入口：产品边界、reference 路由、签名与通知规则、硬检查点
├─ README.md                   # 本文件
└─ references/                 # 接口口径与代码示例
   ├─ api_apply_sharing.md                    # 发起分账
   ├─ api_query_sharing.md                    # 分账查询
   ├─ api_apply_return.md                     # 发起分账回退
   ├─ api_query_return.md                     # 分账回退查询
   ├─ api_add_receiver.md                     # 添加分账接收方
   ├─ api_remove_receiver.md                  # 删除分账接收方
   ├─ api_query_order_sharable_amount.md      # 查询订单待分账金额
   ├─ api_notify_sharing_result.md            # 分账结果通知
   ├─ api_notify_return_result.md             # 退分账结果通知
   ├─ api_notify_auto_unfreeze.md             # 自动解冻通知
   └─ java-guide.md                           # Java 接入示例（配置、签名、HTTP、DTO、通知）
```

## 签名与分账终态要求

签名：

1. 除 `sign` 外所有参数按 key 的 ASCII 字典序排序，拼接为 `key1=value1&key2=value2&...`，空值（`""` 与 `null`）不参与；
2. 请求加签用商户私钥，响应与通知验签用盛付通公钥；
3. 算法固定 `SHA1withRSA + Base64`；
4. `receivers`、`receiver` 使用 JSON 序列化后的字符串原值参与签名，不要对其内部二次排序；金额一律为分（整数）。

分账终态：

- 发起分账或回退返回 `resultCode=SUCCESS` 仅代表请求受理，不代表资金处理完成；
- 分账单终态以查询 `status=FINISHED` 或已验签的分账结果通知为准，且分账单完成不等于每个接收方都成功，必须逐项检查 `receivers[].sharingStatus`；
- `SYSTEM_ERROR`、网络超时或结果不明确时，用相同 `mchSharingNo` / `mchReturnNo` 查询确认，禁止换新单号重发；
- 有签名的通知必须先验签，处理后返回纯文本 `SUCCESS`（区分大小写），未确认最多重试 10 次、间隔递增；
- 自动解冻通知官网未提供签名字段与确认返回说明，不得虚构其验签、重试或成功返回协议。

## 边界说明

- **使用范围**：平台商对已支付订单发起分账、查询分账、分账回退、查询回退、维护商户分账接收方、查询订单待分账金额、处理分账相关通知
- **不覆盖**：普通支付下单、退款、商户进件、钱包开通、平台商提现或内部账务接口
- 分账前，支付订单必须按对应支付产品的分账要求下单；不同支付产品传递分账参数的方式不同
- 本仓库不包含真实商户密钥，文档中密钥片段均为示例占位符
- 密钥、商户号、回调地址等运行时配置必须外部化注入，禁止硬编码
- 不回答费率、合规、通道准入结论；生产问题不定责，仅整理升级材料

## 版权说明

本技能包内容基于盛付通官方开放平台文档及接入资料整理，仅供盛付通平台商分账接入的技术参考使用，不得擅自篡改、转售或用于与接入盛付通服务无关的其他用途。
