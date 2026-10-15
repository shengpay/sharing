# Java 实现指南

本指南面向外部商户，示例使用 Spring Boot、OkHttp 和 Jackson。所有字段、路径和状态以同级 `api_*.md` 文档为准。

## 1. 依赖

### Maven

```xml
<dependencies>
    <dependency>
        <groupId>com.squareup.okhttp3</groupId>
        <artifactId>okhttp</artifactId>
        <version>4.12.0</version>
    </dependency>
    <dependency>
        <groupId>com.fasterxml.jackson.core</groupId>
        <artifactId>jackson-databind</artifactId>
        <version>2.17.0</version>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
</dependencies>
```

### Gradle

```groovy
implementation 'com.squareup.okhttp3:okhttp:4.12.0'
implementation 'com.fasterxml.jackson.core:jackson-databind:2.17.0'
implementation 'org.springframework.boot:spring-boot-starter-web'
```

## 2. 配置

```java
import org.springframework.boot.context.properties.ConfigurationProperties;
import org.springframework.stereotype.Component;

@Component
@ConfigurationProperties(prefix = "shengpay")
public class ShengpayConfig {
    /** 盛付通商户号 */
    private String merchantNo;
    /** 商户 1024 位 RSA 私钥，PKCS8，Base64，不含 PEM 头尾 */
    private String privateKey;
    /** 盛付通 RSA 公钥，用于响应和通知验签 */
    private String publicKey;
    /** 集测：https://mchapitest.shengpay.com；生产：https://mchapi.shengpay.com */
    private String baseUrl;
    /** 分账结果通知地址 */
    private String sharingNotifyUrl;
    /** 退分账结果通知地址 */
    private String returnNotifyUrl;
    /** 钱包接收方场景使用的开发者账号 */
    private String devId;

    public String getMerchantNo() { return merchantNo; }
    public void setMerchantNo(String merchantNo) { this.merchantNo = merchantNo; }
    public String getPrivateKey() { return privateKey; }
    public void setPrivateKey(String privateKey) { this.privateKey = privateKey; }
    public String getPublicKey() { return publicKey; }
    public void setPublicKey(String publicKey) { this.publicKey = publicKey; }
    public String getBaseUrl() { return baseUrl; }
    public void setBaseUrl(String baseUrl) { this.baseUrl = baseUrl; }
    public String getSharingNotifyUrl() { return sharingNotifyUrl; }
    public void setSharingNotifyUrl(String sharingNotifyUrl) { this.sharingNotifyUrl = sharingNotifyUrl; }
    public String getReturnNotifyUrl() { return returnNotifyUrl; }
    public void setReturnNotifyUrl(String returnNotifyUrl) { this.returnNotifyUrl = returnNotifyUrl; }
    public String getDevId() { return devId; }
    public void setDevId(String devId) { this.devId = devId; }
}
```

### application.yml

```yaml
shengpay:
  merchant-no: ${SHENGPAY_MCH_ID}
  private-key: ${SHENGPAY_PRIVATE_KEY}
  public-key: ${SHENGPAY_PUBLIC_KEY}
  base-url: https://mchapitest.shengpay.com
  sharing-notify-url: https://merchant.example.com/sharing/notify
  return-notify-url: https://merchant.example.com/return/notify
  dev-id: ${SHENGPAY_DEV_ID:}
```

生产环境把 `base-url` 改为 `https://mchapi.shengpay.com`。私钥和公钥禁止写入代码仓库。

## 3. RSA 签名与验签

```java
import java.nio.charset.StandardCharsets;
import java.security.KeyFactory;
import java.security.PrivateKey;
import java.security.PublicKey;
import java.security.Signature;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;
import java.util.Base64;
import java.util.Map;
import java.util.TreeSet;
import java.util.stream.Collectors;

public final class ShengpaySignUtil {

    private ShengpaySignUtil() {
    }

    public static String sign(Map<String, Object> params, String privateKeyBase64) throws Exception {
        String content = buildSignString(params);
        PrivateKey privateKey = loadPrivateKey(privateKeyBase64);

        Signature signature = Signature.getInstance("SHA1withRSA");
        signature.initSign(privateKey);
        signature.update(content.getBytes(StandardCharsets.UTF_8));
        return Base64.getEncoder().encodeToString(signature.sign());
    }

    public static boolean verify(Map<String, Object> params, String signValue,
                                 String publicKeyBase64) throws Exception {
        if (signValue == null || signValue.isBlank()) {
            return false;
        }

        String content = buildSignString(params);
        PublicKey publicKey = loadPublicKey(publicKeyBase64);
        Signature signature = Signature.getInstance("SHA1withRSA");
        signature.initVerify(publicKey);
        signature.update(content.getBytes(StandardCharsets.UTF_8));
        return signature.verify(Base64.getDecoder().decode(signValue));
    }

    public static String buildSignString(Map<String, Object> params) {
        Map<String, Object> filtered = params.entrySet().stream()
                .filter(entry -> entry.getValue() != null)
                .filter(entry -> !"sign".equals(entry.getKey()))
                .filter(entry -> !"".equals(String.valueOf(entry.getValue())))
                .collect(Collectors.toMap(Map.Entry::getKey, Map.Entry::getValue));

        return new TreeSet<>(filtered.keySet()).stream()
                .map(key -> key + "=" + filtered.get(key))
                .collect(Collectors.joining("&"));
    }

    private static PrivateKey loadPrivateKey(String value) throws Exception {
        String normalized = normalizeKey(value);
        byte[] encoded = Base64.getDecoder().decode(normalized);
        return KeyFactory.getInstance("RSA")
                .generatePrivate(new PKCS8EncodedKeySpec(encoded));
    }

    private static PublicKey loadPublicKey(String value) throws Exception {
        String normalized = normalizeKey(value);
        byte[] encoded = Base64.getDecoder().decode(normalized);
        return KeyFactory.getInstance("RSA")
                .generatePublic(new X509EncodedKeySpec(encoded));
    }

    private static String normalizeKey(String value) {
        if (value == null) {
            throw new IllegalArgumentException("RSA key must not be null");
        }
        return value
                .replace("-----BEGIN PRIVATE KEY-----", "")
                .replace("-----END PRIVATE KEY-----", "")
                .replace("-----BEGIN RSA PRIVATE KEY-----", "")
                .replace("-----END RSA PRIVATE KEY-----", "")
                .replace("-----BEGIN PUBLIC KEY-----", "")
                .replace("-----END PUBLIC KEY-----", "")
                .replaceAll("\\s+", "");
    }
}
```

盛付通当前文档示例使用 `signType=RSA`，对应 `SHA1withRSA + Base64`。请求签名使用商户私钥，响应和通知验签使用盛付通公钥。

## 4. HTTP 客户端

```java
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import okhttp3.MediaType;
import okhttp3.OkHttpClient;
import okhttp3.Request;
import okhttp3.RequestBody;
import okhttp3.Response;
import okhttp3.ResponseBody;

import java.io.IOException;
import java.security.SecureRandom;
import java.util.LinkedHashMap;
import java.util.Map;
import java.util.concurrent.TimeUnit;

public class ShengpaySharingClient {
    private static final MediaType JSON_TYPE =
            MediaType.parse("application/json; charset=UTF-8");
    private static final String CHARS =
            "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz0123456789";

    private final ShengpayConfig config;
    private final ObjectMapper objectMapper;
    private final OkHttpClient httpClient;
    private final SecureRandom random = new SecureRandom();

    public ShengpaySharingClient(ShengpayConfig config, ObjectMapper objectMapper) {
        this.config = config;
        this.objectMapper = objectMapper;
        this.httpClient = new OkHttpClient.Builder()
                .connectTimeout(5, TimeUnit.SECONDS)
                .readTimeout(30, TimeUnit.SECONDS)
                .build();
    }

    public Map<String, Object> post(String path, Map<String, Object> businessParams)
            throws Exception {
        Map<String, Object> params = new LinkedHashMap<>(businessParams);
        params.put("mchId", config.getMerchantNo());
        params.put("nonceStr", generateNonceStr());
        params.put("signType", "RSA");
        params.put("sign", ShengpaySignUtil.sign(params, config.getPrivateKey()));

        String body = objectMapper.writeValueAsString(params);
        Request request = new Request.Builder()
                .url(config.getBaseUrl() + path)
                .post(RequestBody.create(body, JSON_TYPE))
                .build();

        try (Response response = httpClient.newCall(request).execute()) {
            ResponseBody responseBody = response.body();
            if (!response.isSuccessful() || responseBody == null) {
                throw new IOException("Shengpay HTTP error: " + response.code());
            }

            Map<String, Object> result = objectMapper.readValue(
                    responseBody.string(),
                    new TypeReference<Map<String, Object>>() {
                    });

            String sign = String.valueOf(result.get("sign"));
            if (!result.containsKey("sign")
                    || !ShengpaySignUtil.verify(result, sign, config.getPublicKey())) {
                throw new SecurityException("Shengpay response signature verification failed");
            }
            return result;
        }
    }

    public String generateNonceStr() {
        StringBuilder builder = new StringBuilder(32);
        for (int i = 0; i < 32; i++) {
            builder.append(CHARS.charAt(random.nextInt(CHARS.length())));
        }
        return builder.toString();
    }

    public ObjectMapper getObjectMapper() {
        return objectMapper;
    }

    public ShengpayConfig getConfig() {
        return config;
    }
}
```

`post` 使用 Map 接收响应，先验签再返回，避免强类型对象丢弃盛付通新增字段。

## 5. 请求 DTO

```java
import java.util.ArrayList;
import java.util.List;

public class SharingReceiver {
    public String receiverType;
    public String receiverId;
    public int amount;
    public String description;

    public SharingReceiver() {
    }

    public SharingReceiver(String receiverType, String receiverId,
                           int amount, String description) {
        this.receiverType = receiverType;
        this.receiverId = receiverId;
        this.amount = amount;
        this.description = description;
    }

    public static List<SharingReceiver> single(SharingReceiver receiver) {
        List<SharingReceiver> receivers = new ArrayList<>();
        receivers.add(receiver);
        return receivers;
    }
}
```

`SharingReceiver` 是 `receivers` 数组的成员，不是请求顶层对象。发送前必须序列化为 JSON 字符串。

## 6. 平台商分账 Service

```java
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.stereotype.Service;

import java.util.LinkedHashMap;
import java.util.List;
import java.util.Map;

@Service
public class PlatformSharingService {
    private static final Logger log = LoggerFactory.getLogger(PlatformSharingService.class);

    private final ShengpaySharingClient client;
    private final ObjectMapper objectMapper;

    public PlatformSharingService(ShengpaySharingClient client) {
        this.client = client;
        this.objectMapper = client.getObjectMapper();
    }

    /** POST /sharing/applySharing */
    public Map<String, Object> applySharing(String mchSharingNo,
                                            String transactionId,
                                            int totalAmount,
                                            List<SharingReceiver> receivers)
            throws Exception {
        validateReceivers(totalAmount, receivers);

        Map<String, Object> params = new LinkedHashMap<>();
        params.put("mchSharingNo", mchSharingNo);
        params.put("transactionId", transactionId);
        params.put("totalAmount", totalAmount);
        params.put("notifyUrl", client.getConfig().getSharingNotifyUrl());
        params.put("receivers", objectMapper.writeValueAsString(receivers));
        if (receivers.stream().anyMatch(item -> "C".equals(item.receiverType))) {
            if (isBlank(client.getConfig().getDevId())) {
                throw new IllegalArgumentException("devId is required when receiverType=C");
            }
            params.put("devId", client.getConfig().getDevId());
        }

        try {
            Map<String, Object> result = client.post("/sharing/applySharing", params);
            if ("SYSTEM_ERROR".equals(result.get("errorCode"))) {
                return querySharing(mchSharingNo);
            }
            return result;
        } catch (Exception exception) {
            log.error("分账请求结果不明确，使用相同 mchSharingNo 查询，value={}",
                    mchSharingNo, exception);
            return querySharing(mchSharingNo);
        }
    }

    /** POST /sharing/querySharing */
    public Map<String, Object> querySharing(String mchSharingNo) throws Exception {
        Map<String, Object> params = new LinkedHashMap<>();
        params.put("mchSharingNo", mchSharingNo);
        return client.post("/sharing/querySharing", params);
    }

    /** POST /return/applyReturn */
    public Map<String, Object> applyReturn(String mchSharingNo,
                                           String mchReturnNo,
                                           String returnReceiverType,
                                           String returnReceiverId,
                                           int returnAmount,
                                           String returnDescription)
            throws Exception {
        Map<String, Object> params = new LinkedHashMap<>();
        params.put("mchSharingNo", mchSharingNo);
        params.put("mchReturnNo", mchReturnNo);
        params.put("notifyUrl", client.getConfig().getReturnNotifyUrl());
        params.put("returnReceiverType", returnReceiverType);
        params.put("returnReceiverId", returnReceiverId);
        params.put("returnAmount", returnAmount);
        params.put("returnDescription", returnDescription);

        try {
            Map<String, Object> result = client.post("/return/applyReturn", params);
            if ("SYSTEM_ERROR".equals(result.get("errorCode"))) {
                return queryReturn(mchReturnNo);
            }
            return result;
        } catch (Exception exception) {
            log.error("回退请求结果不明确，使用相同 mchReturnNo 查询，value={}",
                    mchReturnNo, exception);
            return queryReturn(mchReturnNo);
        }
    }

    /** POST /return/queryReturn */
    public Map<String, Object> queryReturn(String mchReturnNo) throws Exception {
        Map<String, Object> params = new LinkedHashMap<>();
        params.put("mchReturnNo", mchReturnNo);
        return client.post("/return/queryReturn", params);
    }

    /** POST /sharing/receiver/add */
    public Map<String, Object> addMerchantReceiver(String receiverId) throws Exception {
        return changeReceiver("/sharing/receiver/add", receiverId);
    }

    /** POST /sharing/receiver/remove */
    public Map<String, Object> removeMerchantReceiver(String receiverId) throws Exception {
        return changeReceiver("/sharing/receiver/remove", receiverId);
    }

    /** POST /sharing/queryOrderSharableAmount */
    public Map<String, Object> queryOrderSharableAmount(String transactionId)
            throws Exception {
        Map<String, Object> params = new LinkedHashMap<>();
        params.put("transactionId", transactionId);
        return client.post("/sharing/queryOrderSharableAmount", params);
    }

    private Map<String, Object> changeReceiver(String path, String receiverId)
            throws Exception {
        Map<String, Object> receiver = new LinkedHashMap<>();
        receiver.put("receiverType", "B");
        receiver.put("receiverId", receiverId);

        Map<String, Object> params = new LinkedHashMap<>();
        params.put("receiver", objectMapper.writeValueAsString(receiver));
        return client.post(path, params);
    }

    private void validateReceivers(int totalAmount, List<SharingReceiver> receivers) {
        if (receivers == null || receivers.isEmpty()) {
            throw new IllegalArgumentException("receivers must not be empty");
        }
        if (receivers.size() > 5) {
            throw new IllegalArgumentException("receivers must not exceed 5 items");
        }

        int sum = 0;
        for (SharingReceiver receiver : receivers) {
            if (!"B".equals(receiver.receiverType)
                    && !"C".equals(receiver.receiverType)) {
                throw new IllegalArgumentException("receiverType must be B or C");
            }
            if (receiver.amount <= 0) {
                throw new IllegalArgumentException("receiver amount must be positive");
            }
            sum += receiver.amount;
        }
        if (sum != totalAmount) {
            throw new IllegalArgumentException("totalAmount must equal sum(receiver.amount)");
        }
    }

    private boolean isBlank(String value) {
        return value == null || value.trim().isEmpty();
    }
}
```

### 调用示例

```java
List<SharingReceiver> receivers = SharingReceiver.single(
        new SharingReceiver("B", "88021534", 200, "partner share"));

Map<String, Object> result = service.applySharing(
        "SHARE202610150001",
        "M20261015123456789012345678",
        200,
        receivers
);
```

## 7. 分账结果通知 Controller

```java
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.List;
import java.util.Map;

@RestController
public class SharingNotifyController {
    private static final Logger log = LoggerFactory.getLogger(SharingNotifyController.class);

    private final ShengpayConfig config;
    private final ObjectMapper objectMapper;

    public SharingNotifyController(ShengpayConfig config, ObjectMapper objectMapper) {
        this.config = config;
        this.objectMapper = objectMapper;
    }

    @PostMapping(path = "/sharing/notify", produces = "text/plain;charset=UTF-8")
    public String handle(@RequestBody Map<String, Object> params) {
        try {
            String sign = String.valueOf(params.get("sign"));
            if (!params.containsKey("sign")
                    || !ShengpaySignUtil.verify(params, sign, config.getPublicKey())) {
                log.warn("分账通知验签失败");
                return "FAIL";
            }

            String mchSharingNo = String.valueOf(params.get("mchSharingNo"));
            // TODO: 按 mchSharingNo 查询本地记录；已处理终态时直接返回 SUCCESS。
            // if (sharingRepository.isFinished(mchSharingNo)) return "SUCCESS";

            String receiversJson = String.valueOf(params.get("receivers"));
            List<Map<String, Object>> receivers = objectMapper.readValue(
                    receiversJson, new TypeReference<List<Map<String, Object>>>() {
                    });
            log.info("接收到分账结果，mchSharingNo={}, receiverCount={}",
                    mchSharingNo, receivers.size());

            // TODO: 在事务中写入分账单和接收方终态，然后提交。
            return "SUCCESS";
        } catch (Exception exception) {
            log.error("处理分账通知失败", exception);
            return "FAIL";
        }
    }
}
```

生产代码中应把日志后的 `TODO` 替换为事务内的业务处理，并保留幂等校验。

## 8. 退分账结果通知 Controller

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class ReturnNotifyController {
    private static final Logger log = LoggerFactory.getLogger(ReturnNotifyController.class);

    private final ShengpayConfig config;

    public ReturnNotifyController(ShengpayConfig config) {
        this.config = config;
    }

    @PostMapping(path = "/return/notify", produces = "text/plain;charset=UTF-8")
    public String handle(@RequestBody Map<String, Object> params) {
        try {
            String sign = String.valueOf(params.get("sign"));
            if (!params.containsKey("sign")
                    || !ShengpaySignUtil.verify(params, sign, config.getPublicKey())) {
                log.warn("退分账通知验签失败");
                return "FAIL";
            }

            String mchReturnNo = String.valueOf(params.get("mchReturnNo"));
            String returnStatus = String.valueOf(params.get("returnStatus"));
            // TODO: 按 mchReturnNo 幂等更新本地回退单。
            // returnStatus 为 SUCCESS / FAIL，只有验签并持久化成功后才返回 SUCCESS。

            return "SUCCESS";
        } catch (Exception exception) {
            log.error("处理退分账通知失败", exception);
            return "FAIL";
        }
    }
}
```

## 9. 自动解冻通知 Controller

```java
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.PostMapping;
import org.springframework.web.bind.annotation.RequestBody;
import org.springframework.web.bind.annotation.RestController;

import java.util.Map;

@RestController
public class AutoUnfreezeNotifyController {
    private static final Logger log = LoggerFactory.getLogger(AutoUnfreezeNotifyController.class);

    /**
     * 官网当前未提供自动解冻通知的签名和响应确认规则。
     * 此处只记录原始业务字段并做幂等处理，不执行伪造的验签逻辑。
     */
    @PostMapping("/sharing/auto-unfreeze/notify")
    public ResponseEntity<Void> handle(@RequestBody Map<String, Object> params) {
        try {
            String mchSharingNo = String.valueOf(params.get("mchSharingNo"));
            String receivers = String.valueOf(params.get("receivers"));
            log.info("收到自动解冻通知，mchSharingNo={}, receivers={}",
                    mchSharingNo, receivers);

            // TODO: 按 mchSharingNo 或 sharingNo 做幂等处理。
            // 返回策略以盛付通技术支持确认的当前环境协议为准。
            return ResponseEntity.ok().build();
        } catch (Exception exception) {
            log.error("处理自动解冻通知失败", exception);
            return ResponseEntity.badRequest().build();
        }
    }
}
```

## 10. 异常与状态处理

### 分账

- `returnCode`/`resultCode` 必须验签后判断。
- `resultCode=SUCCESS` 仅表示受理成功，后续查询 `status` 和每个接收方的 `sharingStatus`。
- `SYSTEM_ERROR` 或网络异常：使用相同 `mchSharingNo` 调用查询接口。
- `DUPLICATE_ORDER`：查询已有分账单，不要新建分账单号。
- `NOT_ENOUGH`：重新查询待分账金额并调整。

### 回退

- `returnStatus` 是终态来源；发起回退的成功响应不是终态。
- `SYSTEM_ERROR` 或网络异常：使用相同 `mchReturnNo` 调用查询接口。
- 失败时记录 `failReason`，并通过对账或人工流程处理。

### 通知

- 分账和退分账通知必须先验签，再处理事务。
- 通知和主动查询可能同时到达，更新必须使用状态机和幂等约束。
- 两个有签名通知返回纯文本 `SUCCESS`。
- 自动解冻通知按官网现状处理，不声明不存在的签名协议。

## 11. 上线检查

- [ ] 商户号、私钥、盛付通公钥和基础地址全部来自安全配置。
- [ ] 集测和生产地址切换正确。
- [ ] 金额全部使用分且通过整数校验。
- [ ] `mchSharingNo`、`mchReturnNo` 有数据库唯一约束。
- [ ] `receivers` 最多 5 项且总金额一致。
- [ ] 钱包接收方传 `devId`。
- [ ] 响应和通知均验签。
- [ ] 通知返回纯文本 `SUCCESS`。
- [ ] 超时、`SYSTEM_ERROR` 后使用查询接口补单。
- [ ] 日志不输出私钥、完整公钥或真实签名。

