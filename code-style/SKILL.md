---
name: java-code-style
description: >
   Best practices for writing Java backend code, including controllers, services, repositories,
   naming conventions, exception handling, logging, annotations, unit tests, SQL usage,
   thread safety, performance, and secure coding practices.
   Ensure consistent coding style, readable, maintainable, and robust backend code.
author:  Team
version: 1.1.0
tags: [java, backend, code-style, quality, secure, performance]
tools: [file_read, grep]
---

# 核心规则

## 1. 代码规模规范
- 单个类不超过 500 行（推荐 300 行）
- 单个方法不超过 80 行（推荐 50 行）
- 嵌套层级不超过 3 层
- 避免深层继承，推荐组合优先

## 2. 类名后缀约定
- API 请求参数使用 `Param` 或 `Request` 后缀
- API 响应结果使用 `Result` 或 `Response` 后缀
- 数据库实体使用 `Model` 后缀
- 内部 DTO/VO 使用 `Dto` 后缀
- 工具类使用 `Util` 后缀
- 异常类使用 `Exception` 后缀

## 3. 命名规范
- 类名：PascalCase
- 方法名、变量名：camelCase
- 常量名：UPPER_SNAKE_CASE
- 接口名：`I` 开头或无前缀，保持语义清晰
- 包名：小写，按功能分层

## 4. 注解规范
- 服务类：`@Service` 或 `@Component`
- Controller：`@RestController` 或 `@Controller`
- Mapper：`@Mapper`
- 日志类：`@Slf4j`
- 敏感字段加注解：如 `@Encrypt` 或自定义注解
- 参数校验使用 `@NotNull`, `@Size` 等 JSR-303 注解

## 5. 异常处理规范
- 异常统一向上抛出，由全局异常处理器统一处理
- 保持异常链完整（使用 cause 参数）
- 不捕获后返回错误码
- 自定义异常分类（业务异常、系统异常、第三方异常）
- 对外 API 不泄露敏感内部信息

## 6. 日志规范
- 使用 SLF4J 占位符：`log.info("UserId: {}", userId)`
- 不要字符串拼接：`log.info("UserId: " + userId)`
- 异常日志必须包含异常对象：`log.error("Failed", e)`
- 关键业务流程必有日志记录
- 敏感信息脱敏（手机号、身份证号、密码等）

## 7. Mapper 方法命名规范
- 查询：`findBy` / `queryBy` / `loadBy`
- 更新：`updateById` / `updateXxxById`
- 插入：`insert` / `batchInsert`
- 删除：`deleteById` / `deleteByIds`
- 使用明确参数名 `@Param("id")`

## 8. 单元测试规范
- 每个类必须有对应测试类
- 测试方法命名：`methodName_shouldDoSomething_whenCondition`
- 尽量覆盖正向、异常、边界、性能场景
- 使用 Mock 避免依赖真实 DB/服务

## 9. SQL/数据库规范
- 避免 `select *`
- 使用参数化查询，防止 SQL 注入
- 复杂 SQL 建议写存储过程或视图
- 对大表分页查询使用索引、limit/offset 或 keyset pagination
- 批量插入/更新尽量使用批处理

## 10. 线程安全 & 并发
- 避免共享可变状态
- 使用 `ConcurrentHashMap`、`AtomicXXX`、`ThreadLocal` 等安全类
- 对于高并发修改，考虑加锁或事务控制
- 避免死锁、无限循环、阻塞调用

## 11. 性能优化
- 避免频繁创建对象（特别是循环内）
- 对大集合使用 Stream 或并行处理时注意性能
- 尽量使用懒加载
- 避免重复计算，可缓存常用结果

## 12. 安全规范
- 敏感信息加密存储
- 输入参数校验严格（长度、类型、格式）
- 防止 XSS、CSRF、SQL 注入等常见攻击
- 不在日志打印敏感信息

## 13. Controller 参数校验规范
- 对前端请求参数使用 `@Valid` 或 `@Validated` 注解
- DTO/Param 对象中使用 JSR-303 校验注解，如：
   - `@NotNull`
   - `@NotBlank`
   - `@Size(min=, max=)`
   - `@Pattern`
- 对方法参数加 `@Valid`，结合 `@RequestBody` 或 `@RequestParam`
- 校验失败由全局异常处理器统一返回标准错误
## 14. Controller 请求/响应日志规范（AOP）

### 核心规则
- 使用 AOP 拦截 Controller 层请求
- 打印请求参数（DTO）和响应结果
- 对敏感信息进行脱敏（手机号、密码、身份证号等）
- 使用 SLF4J 日志，避免字符串拼接
- 配合 `@Valid` 实现参数校验

### 示例 AOP 实现

```java
@Aspect
@Component
@Slf4j
public class ControllerLoggingAspect {

    @Pointcut("within(@org.springframework.web.bind.annotation.RestController *)")
    public void controllerPointcut() {}

    @Around("controllerPointcut()")
    public Object logAround(ProceedingJoinPoint joinPoint) throws Throwable {
        String methodName = joinPoint.getSignature().toShortString();
        Object[] args = joinPoint.getArgs();

        // 打印请求参数（脱敏处理）
        log.info("Request to {} with args: {}", methodName, maskSensitive(args));

        Object result;
        try {
            result = joinPoint.proceed();
        } catch (Throwable t) {
            log.error("Exception in {}: {}", methodName, t.getMessage(), t);
            throw t;
        }

        // 打印响应结果（脱敏处理）
        log.info("Response from {}: {}", methodName, maskSensitive(result));
        return result;
    }

    private Object maskSensitive(Object obj) {
        // 简单示例：可递归检查 DTO，脱敏手机号/密码
        return obj; // 实际可调用通用脱敏工具
    }
}

### 示例

```java
@RestController
@RequestMapping("/user")
public class UserController {

    private final UserService userService;

    public UserController(UserService userService) {
        this.userService = userService;
    }

    // ✅ 使用 @Valid 对请求参数进行校验
    @PostMapping("/register")
    public RegisterResult registerUser(@RequestBody @Valid UserRegisterParam request) {
        return userService.registerUser(request);
    }
}

// ✅ DTO/Param 示例
public class UserRegisterParam {

    @NotBlank(message = "用户名不能为空")
    private String username;

    @NotBlank(message = "密码不能为空")
    @Size(min = 6, max = 20, message = "密码长度必须在6~20位")
    private String password;

    @Email(message = "邮箱格式不正确")
    private String email;

    // getter/setter
}

---

# 示例代码

```java
// ✅ Controller 不写业务逻辑
public class UserRegisterParam {
    private String username;
    private String password;
}
public class RegisterResult {
    private boolean success;
    private String userId;
}

// ✅ Service 层处理业务逻辑
public void registerUser(RegisterRequest request) {
    if (StringUtils.isBlank(request.getUsername())) {
        throw new BusinessException("USERNAME_EMPTY");
    }
    return doRegister(request);
}

// ✅ Mapper 查询
UserModel findById(@Param("id") String id);
List<UserModel> queryByStatus(@Param("status") String status);

// ✅ Mapper 插入/更新/删除
int insert(@Param("model") UserModel model);
int updateStatusById(@Param("id") String id, @Param("status") String status);
int deleteById(@Param("id") String id);

// ✅ 异常处理
try {
    registerUser(request);
} catch (BusinessException e) {
    log.error("User registration failed: {}", request.getUsername(), e);
    throw e;
}
