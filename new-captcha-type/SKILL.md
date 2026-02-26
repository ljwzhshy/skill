---
name: new-captcha-type
description: 在 IMWE 消息中心中新增验证码场景类型，包括添加枚举、配置数据库模板、绑定供应商模板和测试验证
author: IMWE 消息中心团队
version: 1.0.0
tags: [captcha, database, sms, imwe]
tools: [file_read, file_write, sql]
---

# 新增验证码类型

### 核心规则/执行步骤

1. **添加验证码类型枚举**：在 `CaptchaTypeEnum.java` 中添加新的枚举值
   - 使用全大写加下划线命名，如 `VERIFY_IDENTITY`
   - 枚举值与模板代码保持一致

2. **添加数据库模板定义**：为每个支持的国家和语言添加模板
   - 中文模板（中国）：语言 `zh`，国家 `CN`
   - 英文模板（美国）：语言 `en`，国家 `US`
   - 其他语言按需添加

3. **绑定供应商模板**：为每个供应商添加模板绑定
   - Telesign、腾讯云、阿里云等
   - 设置优先级和激活状态

4. **供应商后台配置**：登录供应商后台申请短信模板，获取模板ID后更新数据库

5. **测试验证**：使用 Mock 模式测试流程，验证配置正确性

### 示例

**命令使用：**
```bash
/new-captcha-type --code=VERIFY_IDENTITY --name=实名认证
```

**步骤1 - 添加枚举：**
```java
/**
 * 实名认证
 */
VERIFY_IDENTITY("verify_identity"),
```

**步骤2 - 添加数据库模板：**
```sql
INSERT INTO t_msg_template_definition (
    id, template_name, business_type, captcha_type, message_type,
    language, country_code, description, template_content, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '实名认证模板',
    'IMWE',
    'verify_identity',
    'OTP',
    'zh',
    'CN',
    '实名认证验证码模板',
    '您的验证码是：{code}，5分钟内有效，请勿泄露给他人。',
    NOW(),
    NOW()
);
```

**步骤3 - 绑定供应商：**
```sql
INSERT INTO t_msg_template_binding (
    id, template_def_id, provider_code, external_id, external_title,
    param_map_json, priority, active_status, render_mode, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{template_def_id}}',
    'telesign',
    'TEMPLATE_CODE_VERIFY_IDENTITY',
    '实名认证验证',
    '{"code":"code"}',
    10,
    1,
    0,
    NOW(),
    NOW()
);
```

### 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `code` | String | 是 | 验证码类型代码（如: VERIFY_IDENTITY, BIND_EMAIL） |
| `name` | String | 是 | 场景中文名称（如: 实名认证, 绑定邮箱） |
| `template` | String | 否 | 模板代码（可选，默认与 code 相同） |

### 检查清单

- [ ] CaptchaTypeEnum 枚举已添加
- [ ] 中文模板已添加（数据库）
- [ ] 英文模板已添加（数据库）
- [ ] 其他语言模板已添加（如需要）
- [ ] 供应商模板绑定已添加
- [ ] 供应商后台模板已申请
- [ ] 供应商模板ID已更新到数据库
- [ ] Mock 模式测试通过
- [ ] 实际发送测试通过（关闭 Mock 模式）

### 注意事项

1. **命名规范**：
   - 枚举名使用全大写加下划线：`VERIFY_IDENTITY`
   - 模板代码通常与枚举名一致
   - 数据库中的 `business_type` 要与调用方传入的一致

2. **多语言支持**：
   - 至少配置中文和英文两种语言
   - 不同国家可能需要不同的模板
   - 模板内容要符合当地法规

3. **供应商审核**：
   - 大多数供应商需要审核短信模板
   - 审核时间可能需要1-3个工作日
   - 提前申请，避免影响上线

4. **测试建议**：
   - 先使用 Mock 模式测试流程
   - 再使用小量真实发送测试
   - 确认无误后再开放给用户使用

### 常见问题

**Q1: 模板解析失败？**
检查以下几点：
1. `CaptchaTypeEnum` 中的 `templateCode` 是否与数据库中的 `captcha_type` 一致
2. 数据库中的 `business_type` 是否匹配
3. 模板绑定表的 `active_status` 是否为 1

**Q2: 供应商发送失败？**
检查以下几点：
1. 供应商后台是否已申请并审核通过模板
2. `external_id` 是否与供应商后台的模板ID一致
3. `param_map_json` 变量映射是否正确

**Q3: 验证码收不到？**
排查步骤：
1. 查看应用日志，确认是否真正发送
2. 检查供应商账户余额
3. 检查手机号格式是否正确
4. 检查是否被风控拦截

### 参考示例

- LOGIN_OTP - 注册/登录验证码
- CHANGE_MOBILE - 换绑手机号验证
- RESET_PWD - 找回密码验证
