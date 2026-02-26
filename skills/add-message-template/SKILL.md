# 添加消息模板

## 概述
此技能用于为现有的验证码类型添加新的消息模板（如新增语言支持、新增国家支持等）。它不涉及代码修改，只需要数据库操作和供应商配置。

## 使用方法
```
/add-template --type=<验证码类型> --language=<语言> --country=<国家> --content=<模板内容>
```

### 参数说明
- `type`: 验证码类型（如: LOGIN_OTP, CHANGE_MOBILE）
- `language`: 语言代码（如: zh, en, th, vi）
- `country`: 国家代码（如: CN, US, TH, VN）
- `content`: 模板内容（可选，有默认值）

## 支持的语言和国家

### 语言代码
| 代码 | 语言 |
|------|------|
| zh | 中文 |
| en | 英文 |
| th | 泰语 |
| vi | 越南语 |
| id | 印尼语 |
| ms | 马来语 |

### 国家代码
| 代码 | 国家 |
|------|------|
| CN | 中国 |
| US | 美国 |
| TH | 泰国 |
| VN | 越南 |
| ID | 印度尼西亚 |
| MY | 马来西亚 |
| SG | 新加坡 |
| PH | 菲律宾 |

## 执行步骤

### 步骤 1: 检查验证码类型是否存在
```sql
SELECT code, description
FROM t_msg_template_definition
WHERE captcha_type = '{{type}}'
LIMIT 1;
```

如果返回为空，说明验证码类型不存在，请先使用 `/new-captcha-type` 创建。

### 步骤 2: 添加模板定义
执行以下 SQL 添加新的模板定义：

```sql
INSERT INTO t_msg_template_definition (
    id,
    template_name,
    business_type,
    captcha_type,
    message_type,
    language,
    country_code,
    description,
    template_content,
    create_time,
    update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{type}} - {{language}}-{{country}}',
    'IMWE',
    '{{type}}',
    'OTP',
    '{{language}}',
    '{{country}}',
    '{{type}} 验证码模板 ({{language}}/{{country}})',
    '{{content}}',
    NOW(),
    NOW()
);
```

**默认模板内容**（如果未提供）：

| 语言 | 默认内容 |
|------|----------|
| zh | 您的验证码是：{code}，5分钟内有效，请勿泄露给他人。 |
| en | Your verification code is: {code}. Valid for 5 minutes. Do not share with others. |
| th | รหัสยืนยันของคุณคือ: {code} มีอายุ 5 นาที กรุณาไม่เปิดเผยให้ผู้อื่น |
| vi | Mã xác minh của bạn là: {code}. Có hiệu lực trong 5 phút. Vui lòng không chia sẻ với người khác. |

### 步骤 3: 获取模板定义 ID
```sql
SELECT id
FROM t_msg_template_definition
WHERE captcha_type = '{{type}}'
  AND language = '{{language}}'
  AND country_code = '{{country}}'
  AND business_type = 'IMWE';
```

记录返回的 `id` 值。

### 步骤 4: 查询现有供应商绑定（参考）
```sql
SELECT provider_code, external_id, priority
FROM t_msg_template_binding
WHERE template_def_id IN (
    SELECT id FROM t_msg_template_definition
    WHERE captcha_type = '{{type}}'
    AND business_type = 'IMWE'
    LIMIT 1
)
ORDER BY priority;
```

这会显示该验证码类型下现有供应商的绑定情况，作为新绑定的参考。

### 步骤 5: 绑定供应商模板
为每个供应商添加绑定：

```sql
-- Telesign 绑定
INSERT INTO t_msg_template_binding (
    id,
    template_def_id,
    provider_code,
    external_id,
    external_title,
    param_map_json,
    priority,
    active_status,
    render_mode,
    create_time,
    update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{template_def_id}}',
    'telesign',
    'PENDING_TEMPLATE_ID',  -- 待供应商后台审核后更新
    '{{type}} - {{language}} (Telesign)',
    '{"code":"code"}',
    10,
    1,
    0,
    NOW(),
    NOW()
);

-- 腾讯云绑定
INSERT INTO t_msg_template_binding (
    id,
    template_def_id,
    provider_code,
    external_id,
    external_title,
    param_map_json,
    priority,
    active_status,
    render_mode,
    create_time,
    update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{template_def_id}}',
    'tencent',
    'PENDING_TEMPLATE_ID',
    '{{type}} - {{language}} (Tencent)',
    '{"code":"code"}',
    20,
    1,
    0,
    NOW(),
    NOW()
);

-- 阿里云绑定
INSERT INTO t_msg_template_binding (
    id,
    template_def_id,
    provider_code,
    external_id,
    external_title,
    param_map_json,
    priority,
    active_status,
    render_mode,
    create_time,
    update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{template_def_id}}',
    'aliyun',
    'PENDING_TEMPLATE_ID',
    '{{type}} - {{language}} (Aliyun)',
    '{"code":"code"}',
    30,
    1,
    0,
    NOW(),
    NOW()
);
```

### 步骤 6: 批量添加多语言模板（可选）
如果需要同时添加多种语言，可以使用以下脚本：

```sql
-- 批量添加模板定义
INSERT INTO t_msg_template_definition (id, template_name, business_type, captcha_type, message_type, language, country_code, description, template_content, create_time, update_time) VALUES
(REPLACE(UUID(), '-', ''), '{{type}} - en-US', 'IMWE', '{{type}}', 'OTP', 'en', 'US', '{{type}} Template (en-US)', 'Your verification code is: {code}. Valid for 5 minutes.', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - en-GB', 'IMWE', '{{type}}', 'OTP', 'en', 'GB', '{{type}} Template (en-GB)', 'Your verification code is: {code}. Valid for 5 minutes.', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - zh-TW', 'IMWE', '{{type}}', 'OTP', 'zh', 'TW', '{{type}} Template (zh-TW)', '您的驗證碼是：{code}，5分鐘內有效，請勿洩露給他人。', NOW(), NOW());
```

### 步骤 7: 供应商后台配置
1. **登录各供应商后台**
2. **申请新的短信模板**，使用步骤2中的 `template_content`
3. **记录模板ID**，更新到数据库：

```sql
-- 批量更新供应商模板ID
UPDATE t_msg_template_binding
SET external_id = CASE
    WHEN provider_code = 'telesign' THEN '{{telesign_template_id}}'
    WHEN provider_code = 'tencent' THEN '{{tencent_template_id}}'
    WHEN provider_code = 'aliyun' THEN '{{aliyun_template_id}}'
END
WHERE template_def_id = '{{template_def_id}}';

-- 或者单独更新
UPDATE t_msg_template_binding
SET external_id = '{{supplier_template_id}}'
WHERE template_def_id = '{{template_def_id}}'
  AND provider_code = '{{provider_code}}';
```

### 步骤 8: 验证配置
```sql
-- 查看完整的模板配置
SELECT
    td.id as template_id,
    td.template_name,
    td.language,
    td.country_code,
    td.template_content,
    tb.id as binding_id,
    tb.provider_code,
    tb.external_id,
    tb.priority,
    tb.active_status
FROM t_msg_template_definition td
LEFT JOIN t_msg_template_binding tb ON td.id = tb.template_def_id
WHERE td.captcha_type = '{{type}}'
  AND td.language = '{{language}}'
  AND td.country_code = '{{country}}'
  AND td.business_type = 'IMWE';
```

### 步骤 9: 测试验证
使用 Mock 模式测试：

```bash
curl -X POST "http://localhost:18206/rpc/imwe-message-center-service/msg/captcha/send" \
  -H "Content-Type: application/json" \
  -d '{
    "account": "+1234567890",
    "captchaType": "{{type}}",
    "accountType": "PHONE",
    "sendType": "SMS",
    "locale": "{{language}}_{{country}}",
    "businessId": "test_'$(date +%s)'",
    "businessType": "test",
    "ip": "127.0.0.1"
  }'
```

**注意**：`locale` 参数格式为 `语言_国家`，如 `zh_CN`, `en_US`, `th_TH` 等。

## 快速添加脚本

### 添加单个语言模板
```sql
-- 复制此脚本，替换 {{variable}}
INSERT INTO t_msg_template_definition (
    id, template_name, business_type, captcha_type, message_type,
    language, country_code, description, template_content, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{type}} - {{language}}-{{country}}',
    'IMWE', '{{type}}', 'OTP',
    '{{language}}', '{{country}}',
    '{{type}} 验证码模板',
    '{{content}}',
    NOW(), NOW()
);

-- 获取插入的ID（保存为 {{template_def_id}}）
SELECT id FROM t_msg_template_definition
WHERE captcha_type = '{{type}}'
  AND language = '{{language}}'
  AND country_code = '{{country}}'
  AND business_type = 'IMWE'
ORDER BY create_time DESC LIMIT 1;

-- 绑定供应商（使用上面的ID）
INSERT INTO t_msg_template_binding (
    id, template_def_id, provider_code, external_id, external_title,
    param_map_json, priority, active_status, render_mode, create_time, update_time
) VALUES (
    REPLACE(UUID(), '-', ''),
    '{{template_def_id}}', 'telesign', 'PENDING_ID',
    '{{type}} - Telesign', '{"code":"code"}', 10, 1, 0, NOW(), NOW()
);
```

### 批量添加常用语言
```sql
-- 为 {{type}} 添加常用语言模板
INSERT INTO t_msg_template_definition (id, template_name, business_type, captcha_type, message_type, language, country_code, description, template_content, create_time, update_time) VALUES
(REPLACE(UUID(), '-', ''), '{{type}} - en-US', 'IMWE', '{{type}}', 'OTP', 'en', 'US', '{{type}} (en-US)', 'Your code is: {code}', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - zh-TW', 'IMWE', '{{type}}', 'OTP', 'zh', 'TW', '{{type}} (zh-TW)', '您的驗證碼是：{code}', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - th-TH', 'IMWE', '{{type}}', 'OTP', 'th', 'TH', '{{type}} (th-TH)', 'รหัสยืนยันของคุณคือ: {code}', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - vi-VN', 'IMWE', '{{type}}', 'OTP', 'vi', 'VN', '{{type}} (vi-VN)', 'Mã của bạn là: {code}', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - id-ID', 'IMWE', '{{type}}', 'OTP', 'id', 'ID', '{{type}} (id-ID)', 'Kode Anda adalah: {code}', NOW(), NOW());
```

## 检查清单
- [ ] 模板定义已添加到数据库
- [ ] 模板内容符合目标语言习惯
- [ ] 供应商绑定已添加
- [ ] 供应商后台模板已申请
- [ ] 供应商模板ID已更新到数据库
- [ ] Mock 模式测试通过
- [ ] 实际发送测试通过

## 模板内容规范

### 中文模板
```
您的验证码是：{code}，5分钟内有效，请勿泄露给他人。
```
- 简洁明了
- 说明有效期
- 安全提醒

### 英文模板
```
Your verification code is: {code}. Valid for 5 minutes. Do not share with others.
```
- 使用 "verification code" 而非 "OTP"
- 说明有效期
- 安全提醒

### 其他语言注意事项
1. **泰语**：使用礼貌用语，注意尊称
2. **越南语**：使用正式表达
3. **印尼语**：简洁为主
4. **马来语**：使用通用表达

## 变量使用

### 单变量模板
```
您的验证码是：{code}，5分钟内有效。
```
```json
{"code":"code"}
```

### 多变量模板
```
您的订单 {orderId} 的验证码是：{code}，5分钟内有效。
```
```json
{"code":"code","orderId":"order_id"}
```

### 发送时传递参数
```bash
curl -X POST "..." -d '{
  "templateParams": {
    "orderId": "123456"
  }
}'
```

## 常见问题

### Q1: 模板解析失败
**A**: 检查 `locale` 参数格式是否正确：`语言_国家`，如 `zh_CN`

### Q2: 找不到模板
**A**: 确认以下参数是否匹配：
1. `captchaType` - 验证码类型
2. `locale` - 语言和国家
3. `businessType` - 业务类型

### Q3: 供应商审核不通过
**A**: 常见原因：
1. 模板内容包含禁止词汇
2. 变量格式不符合供应商要求
3. 模板过于相似被拒绝

**解决方案**：
- 修改模板内容
- 调整变量格式
- 使用不同的模板内容

## 注意事项
1. **变量格式**：不同供应商对变量格式要求不同
   - Telesign: `{code}`
   - 腾讯云: `{1}`, `{2}`
   - 阿里云: `${code}`

2. **模板审核**：
   - 多数供应商需要审核
   - 审核时间1-3个工作日
   - 提前申请避免影响上线

3. **测试顺序**：
   - Mock 模式测试
   - 单个供应商测试
   - 供应商切换测试
   - 全量测试

4. **回滚准备**：
   - 保留旧模板配置
   - 准备快速回滚脚本
   - 监控新模板使用情况
