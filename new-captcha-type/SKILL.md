# 新增验证码类型

## 概述
此技能用于在 IMWE 消息中心中新增一个验证码场景类型。它会引导你完成所有必需的步骤，包括：
1. 添加枚举类型
2. 配置数据库模板
3. 绑定供应商模板
4. 测试验证

## 使用方法
```
/new-captcha-type --code=<验证码代码> --name=<场景名称> --template=<模板代码>
```

### 参数说明
- `code`: 验证码类型代码（如: VERIFY_IDENTITY, BIND_EMAIL）
- `name`: 场景中文名称（如: 实名认证, 绑定邮箱）
- `template`: 模板代码（可选，默认与 code 相同）

## 执行步骤

### 步骤 1: 添加验证码类型枚举
在 `imwe-message-center-api/src/main/java/com/imwe/message/constant/CaptchaTypeEnum.java` 中添加枚举：

```java
/**
 * {{场景中文名称}}
 */
{{CAPS_CODE}}("{{template_code}}"),
```

**注意**：
- `{{CAPS_CODE}}` 是全大写下划线命名，如: `VERIFY_IDENTITY`
- `{{template_code}}` 是模板代码，通常与枚举名一致
- 将新枚举添加到合适的位置，保持逻辑顺序

### 步骤 2: 添加数据库模板定义
执行以下 SQL 添加模板定义（需要为每个支持的国家和语言添加）：

```sql
-- 中文模板（中国）
INSERT INTO t_msg_template_definition (
    id, template_name, business_type, captcha_type, message_type,
    language, country_code, description, template_content, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{场景名称}}模板',
    'IMWE',
    '{{template_code}}',
    'OTP',
    'zh',
    'CN',
    '{{场景名称}}验证码模板',
    '您的验证码是：{code}，5分钟内有效，请勿泄露给他人。',
    NOW(),
    NOW()
);

-- 英文模板（美国）
INSERT INTO t_msg_template_definition (
    id, template_name, business_type, captcha_type, message_type,
    language, country_code, description, template_content, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{场景名称}}模板',
    'IMWE',
    '{{template_code}}',
    'OTP',
    'en',
    'US',
    '{{场景名称}} OTP Template',
    'Your verification code is: {code}. Valid for 5 minutes. Do not share with others.',
    NOW(),
    NOW()
);

-- 其他语言模板按需添加...
```

**模板变量说明**：
- `{code}` - 验证码占位符，系统会自动替换
- 可以添加其他自定义变量，如 `{userName}`, `{orderId}` 等

### 步骤 3: 查询模板定义 ID
执行 SQL 获取刚插入的模板定义 ID：

```sql
SELECT id, template_name, language, country_code
FROM t_msg_template_definition
WHERE captcha_type = '{{template_code}}'
  AND business_type = 'IMWE';
```

记录返回的 `id` 值，下一步需要用到。

### 步骤 4: 绑定供应商模板
执行以下 SQL 绑定供应商模板（为每个供应商添加）：

```sql
-- Telesign 短信绑定
INSERT INTO t_msg_template_binding (
    id, template_def_id, provider_code, external_id, external_title,
    param_map_json, priority, active_status, render_mode, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{template_def_id}}',  -- 步骤3中查询到的ID
    'telesign',
    'TEMPLATE_CODE_{{CAPS_CODE}}',
    '{{场景名称}}验证',
    '{"code":"code"}',
    10,
    1,
    0,
    NOW(),
    NOW()
);

-- 腾讯云短信绑定（如果使用）
INSERT INTO t_msg_template_binding (
    id, template_def_id, provider_code, external_id, external_title,
    param_map_json, priority, active_status, render_mode, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{template_def_id}}',
    'tencent',
    'TEMPLATE_CODE_{{CAPS_CODE}}',
    '{{场景名称}}验证',
    '{"code":"code"}',
    20,
    1,
    0,
    NOW(),
    NOW()
);

-- 其他供应商按需添加...
```

**参数说明**：
- `template_def_id`: 步骤3中查询到的模板定义ID
- `external_id`: 供应商侧的模板ID（需要在供应商后台申请）
- `param_map_json`: 变量映射JSON，`{"code":"code"}` 表示系统变量 `{code}` 对应供应商模板中的 `{code}`
- `priority`: 优先级，数字越小优先级越高
- `active_status`: 1=启用，0=禁用
- `render_mode`: 0=模板模式，1=内容透传模式

### 步骤 5: 供应商后台配置（重要）
1. **登录供应商后台**（如 Telesign、腾讯云等）
2. **申请短信模板**，内容与数据库中的 `template_content` 一致
3. **获取模板ID/代码**，更新到 `t_msg_template_binding.external_id` 字段

```sql
-- 更新供应商模板ID
UPDATE t_msg_template_binding
SET external_id = '{{供应商后台申请的模板ID}}'
WHERE provider_code = 'telesign'
  AND template_def_id = '{{template_def_id}}';
```

### 步骤 6: 验证配置
执行以下 SQL 验证配置是否正确：

```sql
-- 查询完整的模板配置
SELECT
    td.template_name,
    td.language,
    td.country_code,
    td.template_content,
    tb.provider_code,
    tb.external_id,
    tb.priority,
    tb.active_status
FROM t_msg_template_definition td
LEFT JOIN t_msg_template_binding tb ON td.id = tb.template_def_id
WHERE td.captcha_type = '{{template_code}}'
  AND td.business_type = 'IMWE'
ORDER BY td.language, td.country_code, tb.priority;
```

### 步骤 7: 测试验证
1. **启动应用**，设置 Mock 模式：
   ```yaml
   msg:
     captcha:
       mock-code: "888888"  # 固定验证码，方便测试
   ```

2. **调用发送接口**：
   ```bash
   curl -X POST "http://localhost:18206/rpc/imwe-message-center-service/msg/captcha/send" \
     -H "Content-Type: application/json" \
     -d '{
       "account": "+1234567890",
       "captchaType": "{{CAPS_CODE}}",
       "accountType": "PHONE",
       "sendType": "SMS",
       "businessId": "test_'$(date +%s)'",
       "businessType": "test",
       "ip": "127.0.0.1"
     }'
   ```

3. **验证日志**，确认：
   - 模板解析成功
   - 供应商选择正确
   - 消息发送成功（Mock 模式不真实发送）

4. **验证验证码**：
   ```bash
   curl -X POST "http://localhost:18206/rpc/imwe-message-center-service/msg/captcha/verify" \
     -H "Content-Type: application/json" \
     -d '{
       "captchaId": "步骤返回的captchaId",
       "account": "+1234567890",
       "code": "888888"
     }'
   ```

## 检查清单
- [ ] CaptchaTypeEnum 枚举已添加
- [ ] 中文模板已添加（数据库）
- [ ] 英文模板已添加（数据库）
- [ ] 其他语言模板已添加（如需要）
- [ ] 供应商模板绑定已添加
- [ ] 供应商后台模板已申请
- 供应商模板ID已更新到数据库
- [ ] Mock 模式测试通过
- [ ] 实际发送测试通过（关闭 Mock 模式）

## 变量映射说明

### 系统内置变量
| 变量名 | 说明 | 示例值 |
|--------|------|--------|
| `{code}` | 验证码 | 123456 |
| `{userName}` | 用户名 | 张三 |
| `{orderId}` | 订单ID | 123456789 |

### 自定义变量
如果模板需要自定义变量，需要：

1. **在模板内容中使用变量**：
   ```sql
   '您的订单 {orderId} 的验证码是：{code}，5分钟内有效。'
   ```

2. **在 param_map_json 中配置映射**：
   ```sql
   '{"code":"code","orderId":"order_id"}'
   ```

3. **发送时传递参数**：
   ```json
   {
     "templateParams": {
       "orderId": "123456789"
     }
   }
   ```

## 常见问题

### Q1: 模板解析失败
**A**: 检查以下几点：
1. `CaptchaTypeEnum` 中的 `templateCode` 是否与数据库中的 `captcha_type` 一致
2. 数据库中的 `business_type` 是否匹配
3. 模板绑定表的 `active_status` 是否为 1

### Q2: 供应商发送失败
**A**: 检查以下几点：
1. 供应商后台是否已申请并审核通过模板
2. `external_id` 是否与供应商后台的模板ID一致
3. `param_map_json` 变量映射是否正确

### Q3: 验证码收不到
**A**: 排查步骤：
1. 查看应用日志，确认是否真正发送
2. 检查供应商账户余额
3. 检查手机号格式是否正确
4. 检查是否被风控拦截

## 注意事项
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

## 参考示例
- LOGIN_OTP - 注册/登录验证码
- CHANGE_MOBILE - 换绑手机号验证
- RESET_PWD - 找回密码验证
