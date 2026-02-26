# 代码规范检查

## 概述
此技能用于检查代码是否符合 IMWE 消息中心项目的编码规范。基于现有代码风格提取的规范要求，确保代码一致性和可维护性。

## 使用方法
```
/code-style --file=<文件路径> --type=<检查类型>
```

### 参数说明
- `file`: 要检查的文件路径（可选，默认检查当前修改的文件）
- `type`: 检查类型（all, naming, structure, annotation, log, format，默认: all）

## 规范检查项

### 0. 代码规模规范

#### 类行数限制
- ✅ **推荐**：单个类不超过 **300 行**
- ⚠️ **警告**：单个类不超过 **500 行**
- ❌ **禁止**：单个类超过 **500 行** 需要拆分
- **拆分建议**：超过 500 行的类应按职责拆分为多个类

```java
// ✅ 正确：职责单一，行数合理
@Slf4j
@Service
public class CaptchaSendService {
    // 核心发送逻辑，约 200-400 行
}

// ❌ 错误：类过大，需要拆分
@Slf4j
@Service
public class MegaService {
    // 600+ 行代码，包含多个职责
    // 应该拆分为：ValidationService, SendingService, LogService 等
}
```

#### 方法行数限制
- ✅ **推荐**：单个方法不超过 **50 行**
- ⚠️ **警告**：单个方法不超过 **80 行**
- ❌ **禁止**：单个方法超过 **80 行** 需要拆分
- **拆分建议**：超过 80 行的方法应提取私有方法

```java
// ✅ 正确：方法简洁，职责单一
public SendResult sendCaptcha(CaptchaSendRequest request) {
    validateParams(request);      // 10 行
    checkIdempotent(request);     // 15 行
    return doSend(request);       // 20 行
}

// ❌ 错误：方法过长，需要拆分
public SendResult sendCaptcha(CaptchaSendRequest request) {
    // 100+ 行代码，包含多个步骤
    // 应该拆分为：validateParams(), checkIdempotent(), doSend() 等
}
```

#### 复杂度控制
- 单个方法嵌套层级不超过 **3 层**
- 单个方法的参数个数不超过 **5 个**（超过考虑使用参数对象）
- 单个方法的变量个数不超过 **10 个**

### 类名后缀约定

#### 后缀规范一览表

| 后缀 | 用途 | 位置 | 示例 |
|------|------|------|------|
| **Param** | API 请求参数 | `api/param/` | `CaptchaSendParam`, `UserInfoParam` |
| **Result** | API 响应结果 | `api/result/` | `SendCaptchaResult`, `VerifyResult` |
| **Model** | 数据库实体模型 | `core/dao/entity/` | `MsgSendLogModel`, `MsgTemplateModel` |
| **Dto** | 数据传输对象 | `core/dto/` | `MessageContextDto`, `UserInfoDto` |
| **Request** | 通用请求对象（同 Param） | `api/param/` | `CaptchaSendRequest`, `CheckRequest` |
| **Response** | 通用响应对象（同 Result） | `api/result/` | `BaseResponse`, `QueryResponse` |

#### API 层参数命名
```java
// ✅ 正确：API 请求参数使用 Param 或 Request 后缀
public class CaptchaSendParam {
    private String account;
    private String captchaType;
}

public class UserInfoParam {
    private String userId;
    private String userName;
}

// ✅ 正确：API 响应结果使用 Result 或 Response 后缀
public class SendCaptchaResult {
    private boolean success;
    private String messageId;
}

public class UserInfoResult {
    private String userId;
    private String userName;
}

// ❌ 错误：使用了不恰当的后缀
public class CaptchaSendRequestDTO {  // API 层不应该用 Dto
}

public class SendCaptchaResponse {    // 应该用 Result
}
```

#### 数据库模型命名
```java
// ✅ 正确：数据库实体使用 Model 后缀
@Data
@Builder
public class MsgSendLogModel implements Serializable {
    private String id;
    private String businessId;
    // ...
}

@Data
@Builder
public class MsgTemplateDefinitionModel implements Serializable {
    private String id;
    private String templateName;
    // ...
}

// ❌ 错误：数据库实体不应使用其他后缀
public class MsgSendLog {           // 应该是 MsgSendLogModel
}

public class MsgSendLogEntity {      // 不应该使用 Entity 后缀
}

public class MsgSendLogDO {         // 不应该使用 DO 后缀
}
```

#### 数据传输对象命名
```java
// ✅ 正确：内部数据流转使用 Dto 后缀
public class MessageContextDto {
    private String businessKey;
    private String target;
    private Map<String, String> params;
}

public class TemplateRenderDto {
    private String templateId;
    private String templateContent;
    private Map<String, String> variables;
}

// ❌ 错误：Dto 使用不当
public class CaptchaSendDto {  // 这是 API 层参数，应该用 Param
}

public class MsgSendLogDto {    // 这是数据库实体，应该用 Model
}
```

#### 数据对象使用场景说明

```
┌─────────────────────────────────────────────────────────────┐
│                       API 接口层                              │
│  CaptchaSendParam ──→ CaptchaSendResult                     │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       业务逻辑层                              │
│  MessageContextDto ──→ (内部流转) ──→ ProviderRequestDto      │
└──────────────────────────┬──────────────────────────────────┘
                           │
                           ▼
┌─────────────────────────────────────────────────────────────┐
│                       数据访问层                              │
│  MsgSendLogModel ──→ MsgTemplateDefinitionModel             │
└─────────────────────────────────────────────────────────────┘
```

#### 特殊场景说明

| 场景 | 使用后缀 | 说明 |
|------|----------|------|
| Feign 接口参数 | Param 或 Request | Feign 客户端接口定义 |
| Feign 接口返回 | Result 或 Response | Feign 客户端接口定义 |
| MyBatis 参数 | Param | Mapper 方法参数，如 @Param("id") |
| 查询条件对象 | Query 或 Criteria | 复杂查询条件封装 |
| 枚举类型 | Enum / Type | 如：CaptchaTypeEnum, ProviderTypeEnum |
| 常量类 | Constants | 如：MessageConstants, RedisKeyConstants |
| 配置类 | Properties / Config | 如：CaptchaProperties, ProviderConfig |
| 异常类 | Exception | 如：RiskControlException, MessageException |
| 工具类 | Util / Helper | 如：DesensitizeUtil, PhoneNumberUtil |
| 过滤器/拦截器 | Filter / Interceptor | 如：BlacklistFilter, LogInterceptor |

#### 命名一致性检查清单
- [ ] API 请求参数使用 `Param` 或 `Request` 后缀
- [ ] API 响应结果使用 `Result` 或 `Response` 后缀
- [ ] 数据库实体使用 `Model` 后缀
- [ ] 内部数据流转使用 `Dto` 后缀
- [ ] 没有混用 `Entity`、`DO`、`VO` 等后缀
- [ ] 枚举使用 `Enum` 或 `Type` 后缀
- [ ] 常量类使用 `Constants` 后缀

### 1. 命名规范

#### 类命名
- ✅ **正确**：`CaptchaController`, `MessageSenderExecutor`, `RiskControlEngine`
- ❌ **错误**：`captchaController`, `message_sender`, `RiskControl`
- **规则**：帕斯卡命名法（PascalCase），首字母大写

#### 方法命名
- ✅ **正确**：`sendCaptcha()`, `checkCaptcha()`, `parsePhoneNumber()`, `validateParams()`
- ❌ **错误**：`SendCaptcha()`, `check_captcha()`, `ParsePhone()`
- **规则**：驼峰命名法（camelCase），首字母小写，动词开头

#### 变量命名
- ✅ **正确**：`captchaCode`, `maxVerifyAttempts`, `retryCount`, `businessKey`
- ❌ **错误**：`captcha_code`, `MaxVerifyAttempts`, `retry_count`, `business_key`
- **规则**：驼峰命名法（camelCase），首字母小写

#### 常量命名
- ✅ **正确**：`CAPTCHA_VERIFY_LOCK_EXPIRE_SECONDS`, `DEFAULT_PRIORITY`, `MAX_RETRY_COUNT`
- ❌ **错误**：`captcha_verify_lock_expire_seconds`, `defaultPriority`, `max_retry_count`
- **规则**：全大写加下划线（UPPER_SNAKE_CASE）

#### 包命名
- ✅ **正确**：`com.imwe.message.core.service`
- ❌ **错误**：`com.imwe.message.Core.Service`, `com.imwe.message.core.services`
- **规则**：全小写，点分隔，单数形式

### 2. 注解规范

#### 类注解顺序
```java
// 1. Lombok
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor

// 2. Spring 注解
@Component
@Service
@Controller
@Repository

// 3. 其他注解
@Slf4j
```

#### 字段注解
```java
// 依赖注入
@Autowired
private SomeService someService;

// 构造器注入（推荐）
private final SomeService someService;

// 敏感数据加密
@EnableEncrypt
private String account;

// 构建器默认值
@Builder.Default
private String status = "INIT";
```

### 3. 类结构规范

#### 标准类结构
```java
// 1. 包声明
package com.xxx.xxx;

// 2. 导入（按字母顺序）
import com.xxx.*;
import org.xxx.*;
import java.xxx.*;
import javax.xxx.*;

// 3. 类文档注释
/**
 * 类描述
 * <p>详细说明</p>
 *
 * @author 作者名
 * @since 1.0.0
 */

// 4. 类注解
@Slf4j
@Service
public class XxxService {

    // 5. 常量
    private static final int CONSTANT_NAME = 100;

    // 6. 成员变量
    private final SomeService someService;

    // 7. 构造方法
    public XxxService(SomeService someService) {
        this.someService = someService;
    }

    // 8. 公共方法
    public Result doSomething() {
        // ...
    }

    // 9. 私有方法
    private void helperMethod() {
        // ...
    }

    // 10. 内部类
    private static class InnerClass {
        // ...
    }
}
```

### 4. 日志规范

#### 日志级别使用
```java
// ERROR - 系统错误，需要立即处理
log.error("Failed to process request - id: {}", id, exception);

// WARN - 警告信息，业务异常
log.warn("Parameter validation failed - account: {}, error: {}", account, error);

// INFO - 重要业务流程
log.info("Message sent successfully - businessKey: {}, provider: {}", key, provider);

// DEBUG - 调试信息
log.debug("Captcha generated - code: {}, validity: {} seconds", code, validity);

// TRACE - 详细跟踪信息（默认不开启）
log.trace("Processing step 1 - value: {}", value);
```

#### 敏感信息脱敏
```java
// ✅ 正确 - 使用脱敏工具
log.info("Account: {}", DesensitizeUtil.maskPhone(account));

// ❌ 错误 - 直接输出敏感信息
log.info("Account: {}", account);
```

### 5. 异常处理规范

#### 统一异常抛出原则
- ✅ **正确**：异常统一向上抛出，由框架统一处理
- ❌ **错误**：在业务层捕获异常后直接吞掉或返回错误码

```java
// ✅ 正确：异常向上抛出，由框架统一处理
public void sendCaptcha(CaptchaSendRequest request) {
    // 参数校验失败，直接抛出异常
    if (StringUtils.isBlank(request.getAccount())) {
        throw new BusinessException(MessageErrorCode.INVALID_PARAM);
    }

    // 业务逻辑失败，直接抛出异常
    if (isBlacklisted(request.getAccount())) {
        throw new RiskControlException(MessageErrorCode.ACCOUNT_BLACKLISTED);
    }

    // 系统异常，包装后抛出
    try {
        doSend(request);
    } catch (Exception e) {
        throw new SystemException(MessageErrorCode.SYSTEM_ERROR, e);
    }
}

// ❌ 错误：捕获异常后返回错误码（不推荐）
public Result sendCaptcha(CaptchaSendRequest request) {
    try {
        // 业务逻辑
        return Result.success();
    } catch (BusinessException e) {
        // 不应该在这里处理，应该向上抛出
        return Result.fail(e.getErrorCode(), e.getMessage());
    }
}
```

#### 异常分层处理

| 层级 | 异常处理策略 | 说明 |
|------|-------------|------|
| **Controller 层** | 不捕获异常，直接抛出 | 由全局异常处理器统一处理 |
| **Service 层** | 不捕获异常，直接抛出 | 业务异常和系统异常都向上抛出 |
| **Manager 层** | 不捕获异常，直接抛出 | 保持异常传播链完整 |
| **框架层** | 全局异常处理器 | 统一捕获并转换为标准响应 |

```java
// ========== Controller 层示例 ==========

@Slf4j
@RestController
@RequestMapping("/rpc/xxx-service/xxx")
public class CaptchaController {

    @Autowired
    private CaptchaService captchaService;

    /**
     * 发送验证码
     *
     * @param request 请求参数
     * @return 响应结果
     */
    @PostMapping("/send")
    public BaseResponse<SendResult> send(@RequestBody CaptchaSendParam request) {
        // 直接调用 Service，不捕获异常
        SendResult result = captchaService.sendCaptcha(request);
        return BaseResponse.success(result);
    }
}

// ========== Service 层示例 ==========

@Slf4j
@Service
public class CaptchaService {

    public SendResult sendCaptcha(CaptchaSendParam request) {
        // 参数校验失败，直接抛出
        if (StringUtils.isBlank(request.getAccount())) {
            throw new BusinessException(MessageErrorCode.INVALID_PARAM, "账号不能为空");
        }

        // 业务逻辑失败，直接抛出
        if (isBlacklisted(request.getAccount())) {
            throw new RiskControlException(MessageErrorCode.ACCOUNT_BLACKLISTED);
        }

        // 继续处理...
        return doSend(request);
    }
}

// ========== 框架全局异常处理器示例 ==========

@Slf4j
@RestControllerAdvice
public class GlobalExceptionHandler {

    /**
     * 业务异常处理
     */
    @ExceptionHandler(BusinessException.class)
    public BaseResponse<?> handleBusinessException(BusinessException e) {
        log.warn("Business exception - code: {}, message: {}", e.getErrorCode(), e.getMessage());
        return BaseResponse.fail(e.getErrorCode(), e.getMessage());
    }

    /**
     * 风控异常处理
     */
    @ExceptionHandler(RiskControlException.class)
    public BaseResponse<?> handleRiskControlException(RiskControlException e) {
        log.warn("Risk control exception - code: {}, message: {}", e.getErrorCode(), e.getMessage());
        return BaseResponse.fail(e.getErrorCode(), e.getMessage());
    }

    /**
     * 系统异常处理
     */
    @ExceptionHandler(Exception.class)
    public BaseResponse<?> handleSystemException(Exception e) {
        log.error("System exception occurred", e);
        return BaseResponse.error(MessageErrorCode.SYSTEM_ERROR);
    }
}
```

#### 异常抛出规范

```java
// ✅ 正确：抛出具体业务异常
throw new BusinessException(MessageErrorCode.INVALID_PARAM, "手机号格式错误");
throw new RiskControlException(MessageErrorCode.FREQUENCY_LIMIT);
throw new MessageException(MessageErrorCode.TEMPLATE_NOT_FOUND);

// ✅ 正确：包装系统异常后抛出
try {
    callExternalApi();
} catch (Exception e) {
    throw new SystemException(MessageErrorCode.EXTERNAL_API_ERROR, e);
}

// ❌ 错误：直接抛出 Exception
throw new Exception("参数错误");  // 应该抛出具体异常类型

// ❌ 错误：捕获后只记录日志不抛出
try {
    doSomething();
} catch (Exception e) {
    log.error("Error occurred", e);
    // 异常被吞掉，应该继续抛出
}

// ❌ 错误：返回 null 代替抛出异常
public Result doSomething() {
    if (invalid) {
        return null;  // 应该抛出异常
    }
    return Result.success();
}
```

#### 异常链保持完整

```java
// ✅ 正确：保持异常链完整
try {
    sendToProvider(provider, message);
} catch (ProviderException e) {
    // 包装原始异常，保留异常链
    throw new MessageException(
        MessageErrorCode.PROVIDER_SEND_FAILED,
        "发送失败: " + provider.getName(),
        e  // 传递原始异常
    );
}

// ❌ 错误：丢失原始异常信息
try {
    sendToProvider(provider, message);
} catch (ProviderException e) {
    // 重新抛出新异常，但丢失了原始异常信息
    throw new MessageException(MessageErrorCode.PROVIDER_SEND_FAILED);
}
```

#### 特殊场景处理

```java
// 场景1：需要清理资源时使用 try-finally
public void processWithResource() {
    Lock lock = null;
    try {
        lock = redisLock.lock(key);
        // 业务逻辑，异常会向上抛出
        doProcess();
    } finally {
        // 确保资源释放
        if (lock != null) {
            lock.unlock();
        }
    }
    // 不捕获异常，让异常向上传播
}

// 场景2：需要转换异常类型
public void convertException() {
    try {
        externalService.call();
    } catch (ExternalServiceException e) {
        // 转换为内部异常类型后抛出
        throw new BusinessException(
            MessageErrorCode.EXTERNAL_SERVICE_ERROR,
            "外部服务调用失败",
            e  // 保留原始异常
        );
    }
}

// 场景3：批量处理中的异常
public void batchProcess(List<Item> items) {
    for (Item item : items) {
        try {
            processItem(item);
        } catch (Exception e) {
            // 记录失败项，继续处理下一项
            log.error("Failed to process item: {}", item.getId(), e);
            failedItems.add(item);
        }
    }
    // 批量处理完成后，如果有失败项，抛出异常
    if (!failedItems.isEmpty()) {
        throw new BatchProcessException(
            MessageErrorCode.BATCH_PROCESS_FAILED,
            "部分项处理失败: " + failedItems.size() + "/" + items.size()
        );
    }
}
```

#### 异常抛出检查清单
- [ ] 业务异常直接抛出，不捕获
- [ ] 系统异常包装后抛出
- [ ] 不吞掉异常（捕获后只记录日志）
- [ ] 保持异常链完整（使用 cause 参数）
- [ ] 使用具体的异常类型，不直接抛出 Exception
- [ ] 资源清理使用 try-finally 或 try-with-resources
- [ ] 不返回 null 代替异常抛出

#### 标准异常处理（已废弃，仅供参考）
以下方式**不推荐**使用，仅用于理解旧代码：

```java
// ❌ 不推荐：在业务层处理异常并返回错误码
public Result doSomething(Request request) {
    try {
        // 业务逻辑
        return Result.success();
    } catch (BusinessException e) {
        log.warn("Business exception - key: {}, error: {}", key, e.getMessage());
        return Result.fail(e.getErrorCode(), e.getMessage());
    } catch (Exception e) {
        log.error("System exception - key: {}", key, e);
        return Result.error(e);
    }
}
```

#### 异常日志记录
- 包含关键业务参数（脱敏后）
- 记录异常类型和消息
- 使用 {} 占位符，不要字符串拼接
- 异常对象作为最后一个参数
- **全局异常处理器统一记录日志，业务层不需要记录后再次抛出**

### 6. 注释规范

#### 类注释
```java
/**
 * 验证码发送服务
 * <p>
 * 核心服务，整合所有模块：
 * <ul>
 *   <li>参数校验</li>
 *   <li>幂等性控制</li>
 *   <li>风控检查</li>
 *   <li>消息发送</li>
 * </ul>
 * </p>
 *
 * @author luojiawei
 * @since 1.0.0
 */
```

#### 方法注释
```java
/**
 * 发送验证码
 *
 * @param request 发送请求
 * @return 发送结果
 * @throws RiskControlException 风控异常
 */
public SendResult sendCaptcha(CaptchaSendRequest request) {
    // ...
}
```

#### 字段注释
```java
/**
 * 随机数生成器（用于生成验证码）
 */
private static final SecureRandom RANDOM = new SecureRandom();

/**
 * 验证码验证锁过期时间（秒）
 */
private static final int CAPTCHA_VERIFY_LOCK_EXPIRE_SECONDS = 10;
```

#### 行内注释
```java
// 1. 解析手机号国家代码
parsePhoneNumber(request, context);

// 2. 检查幂等性（5分钟内重复请求直接返回）
if (!idempotentService.check(context)) {
    return Result.duplicate();
}

// TODO: 实现异步发送逻辑
// FIXME: 修复并发问题
// XXX: 临时方案，需要重构
```

### 7. 代码格式规范

#### 缩进和空格
- 使用 **4 个空格**缩进（不使用 Tab）
- 运算符两端加空格：`a + b`, `x == y`
- 逗号后加空格：`method(a, b, c)`
- 左大括号不换行：`if (condition) {`

#### 行长度
- 最大行长度：**120 字符**
- 超出时换行，缩进 8 个空格

#### 空行规则
```java
// 类成员之间空一行
private final ServiceA serviceA;

private final ServiceB serviceB;

// 方法之间空一行
public void method1() {
}

public void method2() {
}

// 逻辑块之间空一行（可选）
// 解析参数
parseRequest(request);

// 验证参数
validateParams(request);
```

### 8. 集合和数组规范

#### 集合初始化
```java
// ✅ 推荐：使用 Guava 或 Java 9+ 方法
private final Map<String, String> config = Map.of();

// ✅ 可接受：使用不可变集合
private static final List<String> ALLOWED_TYPES = List.of("SMS", "EMAIL");

// ❌ 避免：可变集合作为常量
private static final List<String> TYPES = new ArrayList<>();
```

#### 集合判空
```java
// ✅ 正确：使用 Optional 或工具类
List<Item> items = list != null ? list : List.of();

// ✅ 正确：使用 Stream API
items.stream()
    .filter(Objects::nonNull)
    .collect(Collectors.toList());

// ❌ 错误：直接使用可能为 null 的集合
for (Item item : items) {  // NPE 风险
}
```

### 9. 字符串处理规范

#### 字符串拼接
```java
// ✅ 日志：使用占位符
log.info("Value: {}, Count: {}", value, count);

// ✅ 简单拼接：使用 +
String result = "Hello " + name;

// ✅ 复杂拼接：使用 StringBuilder
StringBuilder sb = new StringBuilder();
sb.append("Hello ").append(name).append("!");
```

#### 字符串比较
```java
// ✅ 正确：常量放前面，避免 NPE
if ("EXPECTED".equals(actual)) {
}

// ✅ 正确：使用 Objects.equals()
if (Objects.equals(a, b)) {
}

// ❌ 错误：变量放前面，可能 NPE
if (actual.equals("EXPECTED")) {
}
```

### 10. 日期时间规范

#### 使用 java.time API
```java
// ✅ 正确：使用 Java 8+ 时间 API
private Date createTime;  // 数据库映射用
private LocalDateTime localDateTime;  // 业务逻辑用

// ✅ 正确：时间转换
Date date = Date.from(localDateTime.atZone(ZoneId.systemDefault()).toInstant();

// ❌ 避免：使用旧版 Date API（除数据库映射外）
private java.sql.Date sqlDate;
```

### 11. 依赖注入规范

#### 构造器注入（推荐）
```java
@Service
public class XxxService {
    private final DependencyA depA;
    private final DependencyB depB;

    public XxxService(DependencyA depA, DependencyB depB) {
        this.depA = depA;
        this.depB = depB;
    }
}
```

#### 字段注入（可选）
```java
@Service
public class XxxService {
    @Autowired
    private DependencyA depA;

    @Autowired(required = false)  // 可选依赖
    private DependencyB depB;
}
```

### 12. Builder 模式规范

#### 实体类使用 Lombok Builder
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class MsgSendLogModel implements Serializable {
    private String id;

    @Builder.Default
    private String status = "INIT";  // 带默认值
}
```

#### Builder 使用
```java
// ✅ 正确
MsgSendLogModel log = MsgSendLogModel.builder()
    .id(UUID.randomUUID().toString())
    .businessId(businessKey)
    .status("INIT")
    .build();
```

### 13. 配置类规范

#### ConfigurationProperties
```java
@Data
@ConfigurationProperties(prefix = "msg.captcha")
public class CaptchaProperties {
    private int expireSeconds = 300;  // 提供默认值
    private int length = 6;
    private String mockCode = "-1";

    // Getter/Setter 由 Lombok @Data 生成
}
```

### 14. 异步和并发规范

#### 异步方法
```java
@Async
public CompletableFuture<Result> asyncMethod(Request request) {
    // 异步逻辑
    return CompletableFuture.completedFuture(result);
}
```

#### 分布式锁
```java
String lockKey = "lock:" + businessKey;
String lockValue = UUID.randomUUID().toString();
boolean locked = redisLock.lock(lockKey, lockValue, 10);
try {
    // 业务逻辑
} finally {
    redisLock.unlock(lockKey, lockValue);
}
```

### 15. 实体类规范

#### 实体类结构
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class XxxModel implements Serializable {
    private static final long serialVersionUID = 1L;

    // 主键
    private String id;

    // 业务字段
    private String field1;

    // 审计字段
    private Date createTime;
    private Date updateTime;
}
```

### 16. Controller 规范

#### REST API 定义
```java
@Slf4j
@RestController
@RequestMapping("/rpc/xxx-service/xxx")
public class XxxController {

    @Autowired
    private XxxService xxxService;

    /**
     * 方法描述
     *
     * @param request 请求参数
     * @return 响应结果
     */
    @PostMapping("/action")
    public Result doAction(@RequestBody Request request) {
        log.info("Received request - param: {}", request.getParam());
        return xxxService.doAction(request);
    }
}
```

### 17. Service 规范

#### Service 层结构
```java
@Slf4j
@Service
public class XxxService {
    // 1. 常量
    private static final int CONSTANT = 100;

    // 2. 依赖（使用 final）
    private final DependencyA depA;

    // 3. 构造器
    public XxxService(DependencyA depA) {
        this.depA = depA;
    }

    // 4. 公共方法
    public Result doSomething(Request request) {
        // 业务逻辑
    }

    // 5. 私有辅助方法
    private void helperMethod() {
    }
}
```

### 18. Mapper 规范

#### Mapper 方法命名规范

##### 查询方法命名
```java
// ✅ 正确：使用 find / query / load 前缀
XxxModel findById(@Param("id") String id);
XxxModel findByBusinessId(@Param("businessId") String businessId);
List<XxxModel> queryByStatus(@Param("status") String status);
XxxModel loadByUniqueId(@Param("uniqueId") String uniqueId);

// ✅ 正确：批量查询
List<XxxModel> findAll();
List<XxxModel> findByIds(@Param("ids") List<String> ids);
Map<String, XxxModel> findMapByStatus(@Param("status") String status);

// ❌ 错误：使用了其他前缀
XxxModel getById(@Param("id") String id);           // 应该用 findById
XxxModel selectById(@Param("id") String id);         // 应该用 findById
XxxModel getById(@Param("id") String id);           // 应该用 findById
```

##### 更新方法命名
```java
// ✅ 正确：使用 updateBy + 字段名
int updateById(@Param("model") XxxModel model);
int updateByBusinessId(@Param("model") XxxModel model);
int updateStatusById(@Param("id") String id, @Param("status") String status);
int updateRetryCount(@Param("id") String id, @Param("count") int count);

// ✅ 正确：批量更新
int batchUpdate(@Param("list") List<XxxModel> list);
int updateStatusByIds(@Param("ids") List<String> ids, @Param("status") String status);

// ❌ 错误：使用了其他前缀
int modifyById(@Param("model") XxxModel model);      // 应该用 updateById
int changeStatus(@Param("id") String id);             // 应该用 updateStatusById
```

##### 插入方法命名
```java
// ✅ 正确：使用 insert 或 save
int insert(@Param("model") XxxModel model);
int insertSelective(@Param("model") XxxModel model);
int save(@Param("model") XxxModel model);
int saveSelective(@Param("model") XxxModel model);

// ✅ 正确：批量插入
int batchInsert(@Param("list") List<XxxModel> list);
int insertBatch(@Param("list") List<XxxModel> list);

// ❌ 错误：使用了其他前缀
int add(@Param("model") XxxModel model);              // 应该用 insert
int create(@Param("model") XxxModel model);           // 应该用 insert
```

##### 删除方法命名
```java
// ✅ 正确：使用 delete 或 remove 前缀
int deleteById(@Param("id") String id);
int deleteByBusinessId(@Param("businessId") String businessId);
int deleteByIds(@Param("ids") List<String> ids);
int removeById(@Param("id") String id);

// ✅ 正确：逻辑删除（更新状态）
int deleteLogicallyById(@Param("id") String id);
int markAsDeleted(@Param("id") String id);

// ❌ 错误：使用了其他前缀
int remove(@Param("id") String id);                   // 应该用 deleteById
int delById(@Param("id") String id);                   // 应该用 deleteById
```

##### 统计方法命名
```java
// ✅ 正确：使用 count 前缀
int count();
int countByStatus(@Param("status") String status);
long countByCreateTime(@Param("startTime") Date startTime, @Param("endTime") Date endTime);

// ❌ 错误：使用了其他前缀
int getNumber();                                        // 应该用 count
int getTotal();                                         // 应该用 count
```

##### 存在性检查方法命名
```java
// ✅ 正确：使用 exists / check 前缀
boolean existsById(@Param("id") String id);
boolean checkExists(@Param("id") String id);
boolean isAvailable(@Param("id") String id);
```

#### 完整 Mapper 接口示例

```java
@Mapper
public interface MsgSendLogMapper {

    // ========== 查询方法 ==========

    /**
     * 根据 ID 查询
     *
     * @param id 主键 ID
     * @return 实体对象
     */
    MsgSendLogModel findById(@Param("id") String id);

    /**
     * 根据业务 ID 查询
     *
     * @param businessId 业务 ID
     * @return 实体对象
     */
    MsgSendLogModel findByBusinessId(@Param("businessId") String businessId);

    /**
     * 查询所有记录（分页）
     *
     * @param page 页码
     * @param size 页大小
     * @return 记录列表
     */
    List<MsgSendLogModel> queryByPage(
        @Param("page") int page,
        @Param("size") int size
    );

    /**
     * 批量查询（根据状态）
     *
     * @param status 状态
     * @return 记录列表
     */
    List<MsgSendLogModel> findListByStatus(@Param("status") String status);

    // ========== 插入方法 ==========

    /**
     * 插入记录
     *
     * @param model 实体对象
     * @return 影响行数
     */
    int insert(@Param("model") MsgSendLogModel model);

    /**
     * 批量插入
     *
     * @param list 记录列表
     * @return 影响行数
     */
    int batchInsert(@Param("list") List<MsgSendLogModel> list);

    // ========== 更新方法 ==========

    /**
     * 根据 ID 更新
     *
     * @param model 实体对象
     * @return 影响行数
     */
    int updateById(@Param("model") MsgSendLogModel model);

    /**
     * 更新状态
     *
     * @param id 主键 ID
     * @param status 状态
     * @return 影响行数
     */
    int updateStatusById(
        @Param("id") String id,
        @Param("status") String status
    );

    /**
     * 更新外部消息 ID
     *
     * @param id 主键 ID
     * @param externalMsgId 外部消息 ID
     * @return 影响行数
     */
    int updateExternalMsgIdById(
        @Param("id") String id,
        @Param("externalMsgId") String externalMsgId
    );

    /**
     * 批量更新状态
     *
     * @param ids ID 列表
     * @param status 状态
     * @return 影响行数
     */
    int updateStatusByIds(
        @Param("ids") List<String> ids,
        @Param("status") String status
    );

    // ========== 删除方法 ==========

    /**
     * 根据 ID 删除
     *
     * @param id 主键 ID
     * @return 影响行数
     */
    int deleteById(@Param("id") String id);

    /**
     * 根据 ID 逻辑删除
     *
     * @param id 主键 ID
     * @return 影响行数
     */
    int deleteLogicallyById(@Param("id") String id);

    /**
     * 批量删除
     *
     * @param ids ID 列表
     * @return 影响行数
     */
    int deleteByIds(@Param("ids") List<String> ids);

    // ========== 统计方法 ==========

    /**
     * 统计总数
     *
     * @return 总数
     */
    int count();

    /**
     * 根据状态统计
     *
     * @param status 状态
     * @return 数量
     */
    int countByStatus(@Param("status") String status);

    /**
     * 统计时间范围内的数量
     *
     * @param startTime 开始时间
     * @param endTime 结束时间
     * @return 数量
     */
    int countByTimeRange(
        @Param("startTime") Date startTime,
        @Param("endTime") Date endTime
    );

    // ========== 存在性检查 ==========

    /**
     * 检查 ID 是否存在
     *
     * @param id 主键 ID
     * @return 是否存在
     */
    boolean existsById(@Param("id") String id);

    /**
     * 检查业务 ID 是否存在
     *
     * @param businessId 业务 ID
     * @return 是否存在
     */
    boolean checkExistsByBusinessId(@Param("businessId") String businessId);
}
```

#### Mapper 方法命名规则总结

| 操作类型 | 前缀 | 示例 | 说明 |
|----------|------|------|------|
| 单条查询 | `findBy` | `findById`, `findByBusinessId` | 按 X 查找 |
| 批量查询 | `queryBy` | `queryByStatus`, `queryByPage` | 按 X 条件查询 |
| 列表查询 | `findListBy` | `findListByStatus` | 查找列表 |
| 单条加载 | `loadBy` | `loadByUniqueId` | 按 X 加载 |
| 单条更新 | `updateById` | `updateById` | 更新整条记录 |
| 字段更新 | `updateXxxById` | `updateStatusById` | 更新指定字段 |
| 批量更新 | `updateXxxByIds` | `updateStatusByIds` | 批量更新字段 |
| 单条插入 | `insert` | `insert`, `insertSelective` | 插入记录 |
| 批量插入 | `batchInsert` | `batchInsert`, `insertBatch` | 批量插入 |
| 单条删除 | `deleteById` | `deleteById`, `removeById` | 删除记录 |
| 逻辑删除 | `deleteLogicallyById` | `deleteLogicallyById` | 逻辑删除 |
| 批量删除 | `deleteByIds` | `deleteByIds`, `removeByIds` | 批量删除 |
| 统计数量 | `countBy` | `count`, `countByStatus` | 统计数量 |
| 存在检查 | `existsBy` | `existsById`, `checkExistsBy` | 检查存在 |

#### Mapper 命名检查清单
- [ ] 查询方法使用 `find` / `query` / `load` 前缀
- [ ] 更新方法使用 `update` + `ByXxx` 格式
- [ ] 插入方法使用 `insert` 或 `save` 前缀
- [ ] 删除方法使用 `delete` 或 `remove` 前缀
- [ ] 统计方法使用 `count` 前缀
- [ ] 存在性检查使用 `exists` 或 `check` 前缀
- [ ] 方法命名清晰表达操作意图
- [ ] 批量操作方法名包含 `batch` 或 `List` 或 `Ids`

### 19. 错误码规范

#### 错误码定义
```java
public enum MessageErrorCode {
    SUCCESS("200", "成功"),
    INVALID_PARAM("400", "参数错误"),
    PHONE_COUNTRY_PARSE_FAILED("400001", "手机号国家代码解析失败"),
    CAPTCHA_NOT_EXIST("404001", "验证码不存在"),
    CAPTCHA_INVALID("400002", "验证码错误"),
    CAPTCHA_VERIFY_LIMIT("429001", "验证码验证次数超限");

    private final String code;
    private final String message;
}
```

### 20. 单元测试规范

#### 测试类命名
```
✅ 正确：CaptchaSendServiceTest, MessageSenderExecutorTest
❌ 错误：CaptchaServiceTest, messageSenderTest
```

#### 测试方法命名
```
✅ 正确：testSendCaptcha_Success(), testSendCaptcha_InvalidParam()
❌ 错误：test1(), testSend()
```

## 代码审查清单

在提交代码前，请确认以下事项：

### 结构性检查
- [ ] 类结构符合规范（常量 → 变量 → 构造器 → 方法）
- [ ] 导入语句已整理并去重
- [ ] 无未使用的导入
- [ ] 无调试代码（System.out.println, TODO 等）
- [ ] 无注释掉的代码块
- [ ] **单个类不超过 500 行**
- [ ] **单个方法不超过 80 行**

### 命名检查
- [ ] 类名使用 PascalCase
- [ ] 方法名和变量名使用 camelCase
- [ ] 常量名使用 UPPER_SNAKE_CASE
- [ ] 命名具有描述性，不使用缩写（除通用缩写）
- [ ] **API 请求参数使用 Param 或 Request 后缀**
- [ ] **API 响应结果使用 Result 或 Response 后缀**
- [ ] **数据库实体使用 Model 后缀**
- [ ] **内部数据流转使用 Dto 后缀**
- [ ] **没有混用 Entity、DO、VO 等后缀**

### 注解检查
- [ ] 必要的注解已添加（@Service, @Component 等）
- [ ] @Slf4j 已添加（如需日志）
- [ ] @EnableEncrypt 已添加（敏感字段）

### 日志检查
- [ ] 敏感信息已脱敏（手机号、身份证等）
- [ ] 使用正确的日志级别
- [ ] 异常日志包含异常对象
- [ ] 关键业务流程有日志记录

### 异常处理检查
- [ ] 异常被正确捕获和处理
- [ ] 异常信息包含足够的上下文
- [ ] 资源正确释放（锁、连接等）

### 安全检查
- [ ] 输入参数已验证
- [ ] SQL 注入风险已防范
- [ ] 敏感数据已加密

### 性能检查
- [ ] 无不必要的数据库查询
- [ ] 集合使用得当
- [ ] 缓存使用合理

## 代码审查命令

### 检查单个文件
```
/code-style --file=src/main/java/com/imwe/message/controller/CaptchaController.java
```

### 检查所有修改
```
/git diff --name-only | head -10
```

### 检查特定类型
```
/code-style --type=naming      # 只检查命名
/code-style --type=structure    # 只检查结构
/code-style --type=log          # 只检查日志
```

## 常见问题

### Q1: 代码格式化后还是有问题
**A**: 确保项目中有正确的格式化配置：
- Eclipse/IntelliJ: 导入项目代码风格配置
- 使用项目提供的 .editorconfig

### Q2: 注释需要写中文还是英文
**A**:
- 类注释：中文
- 方法注释：中文
- 行内注释：中文
- 代码变量：英文

### Q3: 日志中如何记录参数
**A**:
- 使用 SLF4J 占位符：`log.info("Value: {}", value)`
- 不要字符串拼接：`log.info("Value: " + value)`
- 敏感信息脱敏：`DesensitizeUtil.maskPhone(phone)`

## 相关文档
- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Alibaba Java Coding Guidelines](https://github.com/alibaba/p3c)
- 项目代码示例：参考现有代码风格
