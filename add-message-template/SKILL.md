---
name: add-message-template
description: 为现有验证码类型添加新的消息模板，支持新增语言和国家，仅涉及数据库操作和供应商配置
author: IMWE 消息中心团队
version: 1.0.0
tags: [template, database, sms, i18n]
tools: [sql]
---

# 添加消息模板

### 核心规则/执行步骤

1. **检查验证码类型是否存在**：确认验证码类型已在系统中定义

2. **添加模板定义**：执行 SQL 添加新的模板定义
   - 设置正确的语言代码（zh, en, th, vi 等）
   - 设置正确的国家代码（CN, US, TH, VN 等）
   - 提供符合目标语言习惯的模板内容

3. **获取模板定义 ID**：查询刚插入的模板定义 ID，后续绑定需要使用

4. **绑定供应商模板**：为每个供应商添加绑定
   - Telesign、腾讯云、阿里云等
   - 设置优先级（数字越小优先级越高）

5. **供应商后台配置**：登录各供应商后台申请新的短信模板，获取模板ID后更新数据库

6. **测试验证**：使用 Mock 模式测试，确认模板解析正确

### 示例

**命令使用：**
```bash
/add-template --type=LOGIN_OTP --language=th --country=TH
```

**支持的语言：**
| 代码 | 语言 |
|------|------|
| zh | 中文 |
| en | 英文 |
| th | 泰语 |
| vi | 越南语 |
| id | 印尼语 |
| ms | 马来语 |

**支持的国家：**
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

**步骤2 - 添加模板定义：**
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
    'LOGIN_OTP - th-TH',
    'IMWE',
    'LOGIN_OTP',
    'OTP',
    'th',
    'TH',
    'LOGIN_OTP 验证码模板 (th/TH)',
    'รหัสยืนยันของคุณคือ: {code} มีอายุ 5 นาที กรุณาไม่เปิดเผยให้ผู้อื่น',
    NOW(),
    NOW()
);
```

**步骤5 - 绑定供应商：**
```sql
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
    'PENDING_TEMPLATE_ID',
    'LOGIN_OTP - th (Telesign)',
    '{"code":"code"}',
    10,
    1,
    0,
    NOW(),
    NOW()
);
```

**更新供应商模板ID：**
```sql
UPDATE t_msg_template_binding
SET external_id = '{{supplier_template_id}}'
WHERE template_def_id = '{{template_def_id}}'
  AND provider_code = '{{provider_code}}';
```

### 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `type` | String | 是 | 验证码类型（如: LOGIN_OTP, CHANGE_MOBILE） |
| `language` | String | 是 | 语言代码（如: zh, en, th, vi） |
| `country` | String | 是 | 国家代码（如: CN, US, TH, VN） |
| `content` | String | 否 | 模板内容（可选，有默认值） |

### 默认模板内容

| 语言 | 默认内容 |
|------|----------|
| zh | 您的验证码是：{code}，5分钟内有效，请勿泄露给他人。 |
| en | Your verification code is: {code}. Valid for 5 minutes. Do not share with others. |
| th | รหัสยืนยันของคุณคือ: {code} มีอายุ 5 นาที กรุณาไม่เปิดเผยให้ผู้อื่น |
| vi | Mã xác minh của bạn là: {code}. Có hiệu lực trong 5 phút. Vui lòng không chia sẻ với người khác. |

### 检查清单

- [ ] 模板定义已添加到数据库
- [ ] 模板内容符合目标语言习惯
- [ ] 供应商绑定已添加
- [ ] 供应商后台模板已申请
- [ ] 供应商模板ID已更新到数据库
- [ ] Mock 模式测试通过
- [ ] 实际发送测试通过

### 注意事项

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

4. **locale 参数格式**：`语言_国家`，如 `zh_CN`, `en_US`, `th_TH`

### 常见问题

**Q1: 模板解析失败？**
检查 `locale` 参数格式是否正确：`语言_国家`，如 `zh_CN`

**Q2: 找不到模板？**
确认以下参数是否匹配：
1. `captchaType` - 验证码类型
2. `locale` - 语言和国家
3. `businessType` - 业务类型

**Q3: 供应商审核不通过？**
常见原因：
1. 模板内容包含禁止词汇
2. 变量格式不符合供应商要求
3. 模板过于相似被拒绝

**解决方案**：
- 修改模板内容
- 调整变量格式
- 使用不同的模板内容

### 批量添加脚本

```sql
-- 批量添加常用语言模板
INSERT INTO t_msg_template_definition (id, template_name, business_type, captcha_type, message_type, language, country_code, description, template_content, create_time, update_time) VALUES
(REPLACE(UUID(), '-', ''), '{{type}} - en-US', 'IMWE', '{{type}}', 'OTP', 'en', 'US', '{{type}} Template (en-US)', 'Your verification code is: {code}. Valid for 5 minutes.', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - zh-TW', 'IMWE', '{{type}}', 'OTP', 'zh', 'TW', '{{type}} Template (zh-TW)', '您的驗證碼是：{code}，5分鐘內有效，請勿洩露給他人。', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - th-TH', 'IMWE', '{{type}}', 'OTP', 'th', 'TH', '{{type}} Template (th-TH)', 'รหัสยืนยันของคุณคือ: {code}', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - vi-VN', 'IMWE', '{{type}}', 'OTP', 'vi', 'VN', '{{type}} Template (vi-VN)', 'Mã của bạn là: {code}', NOW(), NOW()),
(REPLACE(UUID(), '-', ''), '{{type}} - id-ID', 'IMWE', '{{type}}', 'OTP', 'id', 'ID', '{{type}} Template (id-ID)', 'Kode Anda adalah: {code}', NOW(), NOW());
```
