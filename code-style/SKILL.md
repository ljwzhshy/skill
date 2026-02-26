---
name: code-style
description: 检查代码是否符合 IMWE 消息中心项目的编码规范，确保代码一致性和可维护性
author: IMWE 消息中心团队
version: 1.0.0
tags: [code-review, java, style, quality]
tools: [file_read, grep]
---

# 代码规范检查

### 核心规则/执行步骤

**检查以下规范类别（按 type 参数选择）：**

1. **代码规模规范**：
   - 单个类不超过 500 行（推荐 300 行）
   - 单个方法不超过 80 行（推荐 50 行）
   - 嵌套层级不超过 3 层

2. **类名后缀约定**：
   - API 请求参数使用 `Param` 或 `Request` 后缀
   - API 响应结果使用 `Result` 或 `Response` 后缀
   - 数据库实体使用 `Model` 后缀
   - 内部数据流转使用 `Dto` 后缀

3. **命名规范**：
   - 类名：帕斯卡命名法（PascalCase），首字母大写
   - 方法名：驼峰命名法（camelCase），首字母小写，动词开头
   - 变量名：驼峰命名法（camelCase），首字母小写
   - 常量名：全大写加下划线（UPPER_SNAKE_CASE）

4. **异常处理规范**：
   - 异常统一向上抛出，由框架统一处理
   - 不捕获异常后返回错误码
   - 保持异常链完整

5. **Mapper 方法命名规范**：
   - 查询方法使用 `find` / `query` / `load` 前缀
   - 更新方法使用 `update` + `ByXxx` 格式
   - 插入方法使用 `insert` 或 `save` 前缀
   - 删除方法使用 `delete` 或 `remove` 前缀

### 示例

**命令使用：**
```bash
/code-style --file=src/main/java/com/imwe/message/controller/CaptchaController.java
/code-style --type=naming      # 只检查命名
/code-style --type=structure   # 只检查结构
```

**类名后缀示例（正确）：**
```java
// ✅ 正确：API 层参数
public class CaptchaSendParam {
    private String account;
    private String captchaType;
}

// ✅ 正确：API 层响应
public class SendCaptchaResult {
    private boolean success;
    private String messageId;
}

// ✅ 正确：数据库实体
@Data
@Builder
public class MsgSendLogModel implements Serializable {
    private String id;
    private String businessId;
}

// ✅ 正确：内部数据流转
public class MessageContextDto {
    private String businessKey;
    private Map<String, String> params;
}
```

**类名后缀示例（错误）：**
```java
// ❌ 错误：API 层不应该用 Dto
public class CaptchaSendRequestDTO {
}

// ❌ 错误：数据库实体应该用 Model
public class MsgSendLogEntity {
}

// ❌ 错误：不应该混用后缀
public class MsgSendLogDO {
}
```

**异常处理示例（正确）：**
```java
// ✅ 正确：异常向上抛出
public void sendCaptcha(CaptchaSendRequest request) {
    if (StringUtils.isBlank(request.getAccount())) {
        throw new BusinessException(MessageErrorCode.INVALID_PARAM, "账号不能为空");
    }
    if (isBlacklisted(request.getAccount())) {
        throw new RiskControlException(MessageErrorCode.ACCOUNT_BLACKLISTED);
    }
    return doSend(request);
}
```

**异常处理示例（错误）：**
```java
// ❌ 错误：捕获异常后返回错误码
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

**Mapper 方法命名示例：**
```java
// ✅ 正确：查询方法
MsgSendLogModel findById(@Param("id") String id);
List<MsgSendLogModel> queryByStatus(@Param("status") String status);

// ✅ 正确：更新方法
int updateById(@Param("model") MsgSendLogModel model);
int updateStatusById(@Param("id") String id, @Param("status") String status);

// ✅ 正确：插入方法
int insert(@Param("model") MsgSendLogModel model);
int batchInsert(@Param("list") List<MsgSendLogModel> list);

// ✅ 正确：删除方法
int deleteById(@Param("id") String id);
int deleteByIds(@Param("ids") List<String> ids);

// ❌ 错误：使用了其他前缀
MsgSendLogModel getById(@Param("id") String id);           // 应该用 findById
int modifyById(@Param("model") MsgSendLogModel model);      // 应该用 updateById
int add(@Param("model") MsgSendLogModel model);              // 应该用 insert
```

### 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `file` | String | 否 | 要检查的文件路径（可选，默认检查当前修改的文件） |
| `type` | String | 否 | 检查类型（all, naming, structure, annotation, log, format，默认: all） |

### 检查类型说明

| 类型 | 说明 |
|------|------|
| `all` | 检查所有规范 |
| `naming` | 只检查命名规范（类名、方法名、变量名、常量名） |
| `structure` | 只检查类结构和后缀规范 |
| `annotation` | 只检查注解规范 |
| `log` | 只检查日志规范 |
| `format` | 只检查代码格式 |

### 代码审查清单

**结构性检查：**
- [ ] 类结构符合规范（常量 → 变量 → 构造器 → 方法）
- [ ] 导入语句已整理并去重
- [ ] 无未使用的导入
- [ ] 无调试代码（System.out.println, TODO 等）
- [ ] 无注释掉的代码块
- [ ] 单个类不超过 500 行
- [ ] 单个方法不超过 80 行

**命名检查：**
- [ ] 类名使用 PascalCase
- [ ] 方法名和变量名使用 camelCase
- [ ] 常量名使用 UPPER_SNAKE_CASE
- [ ] 命名具有描述性，不使用缩写（除通用缩写）
- [ ] API 请求参数使用 `Param` 或 `Request` 后缀
- [ ] API 响应结果使用 `Result` 或 `Response` 后缀
- [ ] 数据库实体使用 `Model` 后缀
- [ ] 内部数据流转使用 `Dto` 后缀
- [ ] 没有混用 `Entity`、`DO`、`VO` 等后缀

**注解检查：**
- [ ] 必要的注解已添加（@Service, @Component 等）
- [ ] @Slf4j 已添加（如需日志）
- [ ] @EnableEncrypt 已添加（敏感字段）

**日志检查：**
- [ ] 敏感信息已脱敏（手机号、身份证等）
- [ ] 使用正确的日志级别
- [ ] 异常日志包含异常对象
- [ ] 关键业务流程有日志记录

**异常处理检查：**
- [ ] 异常被正确抛出（不捕获后返回错误码）
- [ ] 异常信息包含足够的上下文
- [ ] 资源正确释放（锁、连接等）

### Mapper 命名规则总结

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

### 注意事项

1. **注释语言**：
   - 类注释：中文
   - 方法注释：中文
   - 行内注释：中文
   - 代码变量：英文

2. **日志记录**：
   - 使用 SLF4J 占位符：`log.info("Value: {}", value)`
   - 不要字符串拼接：`log.info("Value: " + value)`
   - 敏感信息脱敏：`DesensitizeUtil.maskPhone(phone)`

3. **异常处理**：
   - 异常统一向上抛出，由框架全局异常处理器统一处理
   - 保持异常链完整，使用 cause 参数传递原始异常
   - 资源清理使用 try-finally 或 try-with-resources

### 相关文档

- [Google Java Style Guide](https://google.github.io/styleguide/javaguide.html)
- [Alibaba Java Coding Guidelines](https://github.com/alibaba/p3c)
