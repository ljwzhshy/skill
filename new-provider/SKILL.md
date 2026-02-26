# 新增短信供应商

## 概述
此技能用于在 IMWE 消息中心中新增一个短信供应商。它会引导你完成所有必需的步骤，包括：
1. 添加枚举类型
2. 创建 Handler 类
3. 创建 Client 类
4. 更新配置文件
5. 数据库配置

## 使用方法
```
/new-provider --provider=<供应商代码> --name=<供应商名称> --type=<SMS|EMAIL>
```

### 参数说明
- `provider`: 供应商代码（如: twilio, tencent, aliyun）
- `name`: 供应商中文名称（如: 腾讯云短信）
- `type`: 供应商类型（SMS-短信, EMAIL-邮件，默认: SMS）

## 执行步骤

### 步骤 1: 添加供应商枚举
在 `imwe-message-center-core/src/main/java/com/imwe/message/core/enums/ProviderTypeEnum.java` 中添加枚举：

```java
/**
 * {{供应商名称}}
 */
{{ENUM_NAME}}("{{provider_code}}", "{{供应商中文名称}}"),
```

### 步骤 2: 创建 Handler 类
在 `imwe-message-center-core/src/main/java/com/imwe/message/core/handler/impl/` 目录下创建 `{{HandlerName}}SmsHandler.java`：

```java
package com.imwe.message.core.handler.impl;

import com.imwe.message.core.client.{{ClientName}}Client;
import com.imwe.message.core.client.request.{{RequestName}};
import com.imwe.message.core.client.response.ClientResponse;
import com.imwe.message.core.constants.MessageErrorCode;
import com.imwe.message.core.enums.ProviderTypeEnum;
import com.imwe.message.core.handler.AbstractMessageHandler;
import com.imwe.message.core.model.MessageContext;
import com.imwe.message.core.model.SendResult;
import com.imwe.message.core.service.cahce.SenderIdCacheService;
import com.imwe.message.core.util.DesensitizeUtil;
import com.imwe.framework.task.ITaskProducer;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

/**
 * {{供应商名称}} 短信处理器
 *
 * @author {{author}}
 */
@Slf4j
@Component
public class {{HandlerName}}SmsHandler extends AbstractMessageHandler {

    private final {{ClientName}}Client client;
    private final SenderIdCacheService senderIdCacheService;
    private final ITaskProducer taskProducer;

    public {{HandlerName}}SmsHandler(@Autowired {{ClientName}}Client client,
                                     @Autowired SenderIdCacheService senderIdCacheService,
                                     @Autowired(required = false) ITaskProducer taskProducer) {
        this.client = client;
        this.senderIdCacheService = senderIdCacheService;
        this.taskProducer = taskProducer;
        log.info("{{供应商名称}} configuration loaded - enabled: {}", client.isHealthy());
    }

    @Override
    protected SendResult doSend(MessageContext context) {
        if (!client.isHealthy()) {
            log.warn("{{供应商名称}} configuration not ready or disabled");
            return SendResult.fail(ProviderTypeEnum.{{ENUM_NAME}},
                    MessageErrorCode.CONFIG_ERROR.getCode(),
                    MessageErrorCode.CONFIG_ERROR.getMessage());
        }

        // 获取消息内容
        String message = context.getTemplateContent();
        if (message == null || message.isBlank()) {
            message = context.getParams().get("message");
        }

        if (message == null || message.isBlank()) {
            log.warn("SMS content is empty - target: {}", DesensitizeUtil.maskPhone(context.getTarget()));
            return SendResult.fail(ProviderTypeEnum.{{ENUM_NAME}},
                    MessageErrorCode.EMPTY_MESSAGE.getCode(),
                    MessageErrorCode.EMPTY_MESSAGE.getMessage(),
                    false);
        }

        // 获取 senderId（可选）
        String senderId = senderIdCacheService
                .resolveSenderId(ProviderTypeEnum.{{ENUM_NAME}}, context.getCountryCode())
                .getSenderId();

        // 构建请求
        {{RequestName}} request = {{RequestName}}.builder()
                .phoneNumber(context.getTarget())
                .message(message)
                .senderId(senderId)
                .externalId(context.getBusinessKey())
                .build();

        log.debug("Sending {{供应商名称}} SMS - target: {}", DesensitizeUtil.maskPhone(context.getTarget()));

        // 调用客户端
        ClientResponse response = client.sendSms(request);

        // 转换结果
        return convertToSendResult(response, context.getTarget());
    }

    private SendResult convertToSendResult(ClientResponse response, String target) {
        if (response.isSuccess()) {
            return SendResult.success(ProviderTypeEnum.{{ENUM_NAME}}, response.getReferenceId());
        }

        log.warn("{{供应商名称}} sending failed - target: {}, error: {}",
                DesensitizeUtil.maskPhone(target), response.getErrorDescription());

        return SendResult.fail(
                ProviderTypeEnum.{{ENUM_NAME}},
                MessageErrorCode.PROVIDER_VALIDATION_FAILED.getCode(),
                MessageErrorCode.PROVIDER_VALIDATION_FAILED.getMessage() + ": " + response.getErrorDescription(),
                response.isRetryable()
        );
    }

    @Override
    public String getProviderType() {
        return ProviderTypeEnum.{{ENUM_NAME}}.getCode();
    }

    @Override
    protected boolean isParamInvalid(MessageContext context) {
        return !client.isHealthy() || super.isParamInvalid(context);
    }

    @Override
    public boolean isHealthy() {
        return client.isHealthy();
    }

    @Override
    public SendResult asyncSend(MessageContext context) {
        // TODO: 实现异步发送逻辑
        return SendResult.success(ProviderTypeEnum.{{ENUM_NAME}}, "");
    }
}
```

### 步骤 3: 创建 Client 类
在 `imwe-message-center-core/src/main/java/com/imwe/message/core/client/` 目录下创建 `{{ClientName}}Client.java`：

```java
package com.imwe.message.core.client;

import com.imwe.message.core.client.request.{{RequestName}};
import com.imwe.message.core.client.response.ClientResponse;
import com.imwe.message.core.config.{{ProviderConfigName}}Properties;
import lombok.extern.slf4j.Slf4j;
import org.springframework.boot.autoconfigure.condition.ConditionalOnProperty;
import org.springframework.http.*;
import org.springframework.stereotype.Component;
import org.springframework.web.client.RestTemplate;

/**
 * {{供应商名称}} 客户端
 *
 * @author {{author}}
 */
@Slf4j
@Component
@ConditionalOnProperty(prefix = "{{config_prefix}}.{{provider_code}}", name = "enabled", havingValue = "true")
public class {{ClientName}}Client {

    private final {{ProviderConfigName}}Properties properties;
    private final RestTemplate restTemplate;

    public {{ClientName}}Client({{ProviderConfigName}}Properties properties, RestTemplate restTemplate) {
        this.properties = properties;
        this.restTemplate = restTemplate;
    }

    /**
     * 发送短信
     */
    public ClientResponse sendSms({{RequestName}} request) {
        try {
            String url = properties.getEndpoint() + "/sms/send";

            HttpHeaders headers = new HttpHeaders();
            headers.setContentType(MediaType.APPLICATION_JSON);
            headers.setBearerAuth(properties.getAppSecret());

            HttpEntity<{{RequestName}}> httpEntity = new HttpEntity<>(request, headers);

            ResponseEntity<String> response = restTemplate.exchange(
                    url,
                    HttpMethod.POST,
                    httpEntity,
                    String.class
            );

            return parseResponse(response.getBody());
        } catch (Exception e) {
            log.error("{{供应商名称}} API call failed", e);
            return ClientResponse.error(e.getMessage());
        }
    }

    private ClientResponse parseResponse(String responseBody) {
        // TODO: 根据供应商API响应格式解析
        return ClientResponse.success("reference-id");
    }

    public boolean isHealthy() {
        return properties.isEnabled()
                && properties.getAppKey() != null
                && !properties.getAppKey().isEmpty();
    }
}
```

### 步骤 4: 创建请求/响应类
在 `imwe-message-center-core/src/main/java/com/imwe/message/core/client/request/` 目录下创建：

```java
// {{RequestName}}.java
package com.imwe.message.core.client.request;

import lombok.Builder;
import lombok.Data;

/**
 * {{供应商名称}} 短信请求
 */
@Data
@Builder
public class {{RequestName}} {
    private String phoneNumber;
    private String message;
    private String senderId;
    private String externalId;
}
```

### 步骤 5: 创建配置属性类
在 `imwe-message-center-core/src/main/java/com/imwe/message/core/config/` 目录下创建 `{{ProviderConfigName}}Properties.java`：

```java
package com.imwe.message.core.config;

import lombok.Data;
import org.springframework.boot.context.properties.ConfigurationProperties;

/**
 * {{供应商名称}} 配置属性
 */
@Data
@ConfigurationProperties(prefix = "{{config_prefix}}.{{provider_code}}")
public class {{ProviderConfigName}}Properties {
    private boolean enabled = true;
    private String appKey;
    private String appSecret;
    private String endpoint;
    private int priority = 10;
}
```

### 步骤 6: 更新 application.yml 配置
在 `imwe-message-center-service/src/main/resources/application.yml` 的 `msg.provider.routes` 部分添加：

```yaml
msg:
  provider:
    routes:
      SMS:
        # {{供应商名称}}
        - provider: {{ENUM_NAME}}  # 如: TWILIO_SMS
          appKey: ${{{provider_code}}.sms.key}
          appSecret: ${{{provider_code}}.sms.secret}
          endpoint: https://api.{{provider_code}}.com
          priority: 10
          enabled: true
```

### 步骤 7: 数据库配置（可选）
如果供应商需要 SenderId 配置，执行以下 SQL：

```sql
INSERT INTO t_msg_provider_sender_id (id, provider_code, country_code, sender_id, priority, required, description, create_time, update_time)
VALUES ('{{provider_code}}_us', '{{provider_code}}', 'US', 'YOUR_SENDER_ID', 10, TRUE, '美国地区SenderID', NOW(), NOW());
```

### 步骤 8: 测试验证
1. 启动应用，检查 Handler 是否正确注册
2. 使用 Mock 模式测试：`msg.captcha.mock-code=888888`
3. 发送测试短信，验证日志输出

## 检查清单
- [ ] ProviderTypeEnum 枚举已添加
- [ ] Handler 类已创建
- [ ] Client 类已创建
- [ ] 请求/响应类已创建
- [ ] 配置类已创建
- [ ] application.yml 已更新
- [ ] 数据库配置已添加（如需要）
- [ ] 启动日志确认组件加载成功
- [ ] Mock 模式测试通过

## 注意事项
1. **命名规范**：
   - 枚举名使用全大写加下划线：`TWILIO_SMS`
   - 类名使用驼峰命名：`TwilioSmsHandler`
   - 配置前缀使用小写加横杠：`twilio.sms`

2. **错误处理**：
   - 务必在 doSend 方法中捕获所有异常
   - 正确设置 isRetryable 标志

3. **日志记录**：
   - 使用 DesensitizeUtil.maskPhone() 脱敏手机号
   - 关键步骤都要记录日志

4. **配置验证**：
   - 使用 @ConditionalOnProperty 确保配置可用时才加载
   - isHealthy() 方法必须正确实现

## 参考示例
- TelesignSmsHandler.java
- TelesignClient.java
- MessageProviderProperties.java
