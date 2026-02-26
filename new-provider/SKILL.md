---
name: new-provider
description: 在 IMWE 消息中心中新增短信或邮件供应商，包括添加枚举、创建 Handler 和 Client 类、更新配置文件
author: IMWE 消息中心团队
version: 1.0.0
tags: [provider, sms, email, integration]
tools: [file_read, file_write, yaml]
---

# 新增短信供应商

### 核心规则/执行步骤

1. **添加供应商枚举**：在 `ProviderTypeEnum.java` 中添加新的供应商枚举
   - 使用全大写加下划线命名，如 `TWILIO_SMS`
   - 格式：`枚举名("供应商代码", "供应商中文名称")`

2. **创建 Handler 类**：在 `core/handler/impl/` 目录下创建
   - 继承 `AbstractMessageHandler`
   - 实现 `doSend()` 和 `asyncSend()` 方法
   - 使用 `@ConditionalOnProperty` 确保配置可用时才加载

3. **创建 Client 类**：在 `core/client/` 目录下创建
   - 封装供应商 API 调用
   - 实现 `sendSms()` 或 `sendEmail()` 方法
   - 实现 `isHealthy()` 方法检查配置状态

4. **创建请求/响应类**：在 `core/client/request/` 目录下创建
   - 使用 Lombok `@Data` 和 `@Builder`
   - 包含所有必需的请求参数

5. **创建配置属性类**：在 `core/config/` 目录下创建
   - 使用 `@ConfigurationProperties`
   - 包含 enabled, appKey, appSecret, endpoint, priority 等字段

6. **更新 application.yml 配置**：在 `msg.provider.routes` 部分添加供应商配置

7. **数据库配置（可选）**：如需 SenderId 配置，执行 SQL 添加

8. **测试验证**：启动应用检查 Handler 注册，使用 Mock 模式测试

### 示例

**命令使用：**
```bash
/new-provider --provider=twilio --name=腾讯云短信 --type=SMS
```

**步骤1 - 添加枚举：**
```java
/**
 * 腾讯云短信
 */
TWILIO_SMS("twilio", "腾讯云短信"),
```

**步骤2 - Handler 类模板：**
```java
@Slf4j
@Component
public class TwilioSmsHandler extends AbstractMessageHandler {

    private final TwilioClient client;
    private final SenderIdCacheService senderIdCacheService;
    private final ITaskProducer taskProducer;

    public TwilioSmsHandler(@Autowired TwilioClient client,
                            @Autowired SenderIdCacheService senderIdCacheService,
                            @Autowired(required = false) ITaskProducer taskProducer) {
        this.client = client;
        this.senderIdCacheService = senderIdCacheService;
        this.taskProducer = taskProducer;
        log.info("Twilio configuration loaded - enabled: {}", client.isHealthy());
    }

    @Override
    protected SendResult doSend(MessageContext context) {
        if (!client.isHealthy()) {
            return SendResult.fail(ProviderTypeEnum.TWILIO_SMS,
                    MessageErrorCode.CONFIG_ERROR.getCode(),
                    MessageErrorCode.CONFIG_ERROR.getMessage());
        }

        String message = context.getTemplateContent();
        String senderId = senderIdCacheService
                .resolveSenderId(ProviderTypeEnum.TWILIO_SMS, context.getCountryCode())
                .getSenderId();

        TwilioRequest request = TwilioRequest.builder()
                .phoneNumber(context.getTarget())
                .message(message)
                .senderId(senderId)
                .externalId(context.getBusinessKey())
                .build();

        ClientResponse response = client.sendSms(request);
        return convertToSendResult(response, context.getTarget());
    }

    @Override
    public String getProviderType() {
        return ProviderTypeEnum.TWILIO_SMS.getCode();
    }
}
```

**步骤3 - Client 类模板：**
```java
@Slf4j
@Component
@ConditionalOnProperty(prefix = "msg.provider.twilio", name = "enabled", havingValue = "true")
public class TwilioClient {

    private final TwilioProperties properties;
    private final RestTemplate restTemplate;

    public ClientResponse sendSms(TwilioRequest request) {
        try {
            String url = properties.getEndpoint() + "/sms/send";
            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.setBearerAuth(properties.getAppSecret());

            HttpEntity<TwilioRequest> httpEntity = new HttpEntity<>(request, headers);
            ResponseEntity<String> response = restTemplate.exchange(url, HttpMethod.POST, httpEntity, String.class);

            return parseResponse(response.getBody());
        } catch (Exception e) {
            log.error("Twilio API call failed", e);
            return ClientResponse.error(e.getMessage());
        }
    }

    public boolean isHealthy() {
        return properties.isEnabled()
                && properties.getAppKey() != null
                && !properties.getAppKey().isEmpty();
    }
}
```

**步骤5 - 配置类模板：**
```java
@Data
@ConfigurationProperties(prefix = "msg.provider.twilio")
public class TwilioProperties {
    private boolean enabled = true;
    private String appKey;
    private String appSecret;
    private String endpoint;
    private int priority = 10;
}
```

**步骤6 - application.yml 配置：**
```yaml
msg:
  provider:
    routes:
      SMS:
        # Twilio 短信
        - provider: TWILIO_SMS
          appKey: ${twilio.sms.key}
          appSecret: ${twilio.sms.secret}
          endpoint: https://api.twilio.com
          priority: 10
          enabled: true
```

### 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `provider` | String | 是 | 供应商代码（如: twilio, tencent, aliyun） |
| `name` | String | 是 | 供应商中文名称（如: 腾讯云短信） |
| `type` | String | 否 | 供应商类型（SMS-短信, EMAIL-邮件，默认: SMS） |

### 检查清单

- [ ] ProviderTypeEnum 枚举已添加
- [ ] Handler 类已创建
- [ ] Client 类已创建
- [ ] 请求/响应类已创建
- [ ] 配置类已创建
- [ ] application.yml 已更新
- [ ] 数据库配置已添加（如需要）
- [ ] 启动日志确认组件加载成功
- [ ] Mock 模式测试通过

### 注意事项

1. **命名规范**：
   - 枚举名使用全大写加下划线：`TWILIO_SMS`
   - 类名使用驼峰命名：`TwilioSmsHandler`
   - 配置前缀使用小写加横杠：`msg.provider.twilio`

2. **错误处理**：
   - 务必在 doSend 方法中捕获所有异常
   - 正确设置 isRetryable 标志

3. **日志记录**：
   - 使用 DesensitizeUtil.maskPhone() 脱敏手机号
   - 关键步骤都要记录日志

4. **配置验证**：
   - 使用 @ConditionalOnProperty 确保配置可用时才加载
   - isHealthy() 方法必须正确实现

### 参考示例

- TelesignSmsHandler.java
- TelesignClient.java
- MessageProviderProperties.java
