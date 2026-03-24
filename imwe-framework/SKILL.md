---
name: imwe-framework
description: 基于 IMWE 框架进行工程化项目开发，包括项目结构、依赖配置、启动类配置等
author: IMWE 消息中心团队
version: 1.0.0
tags: [framework, spring-boot, java, project-setup, configuration]
tools: [file_read, file_write, grep]
---

# IMWE 框架工程化开发

### 核心规则/执行步骤

**使用此 skill 进行以下操作（按 task 参数选择）：**

1. **项目初始化** (init)：
   - 创建标准的三模块 Maven 结构（api/core/service）
   - 配置父 POM 继承 imwe-framework
   - 配置各模块 POM 依赖
   - 创建基础目录结构

2. **依赖管理** (dependency)：
   - 引入 IMWE 框架 Starter（web/mybatis/kafka/consul）
   - 配置依赖版本管理
   - 处理依赖冲突和排除

3. **配置文件** (config)：
   - 配置 application.yml
   - 配置数据源（支持多数据源）
   - 配置 Redis（集群/单机）
   - 配置 Kafka（生产者/消费者）
   - 配置 Consul 服务发现

4. **启动类配置** (application)：
   - 创建标准启动类
   - 配置组件扫描
   - 配置 Mapper 扫描
   - 配置服务发现和 Feign

5. **代码结构** (structure)：
   - 创建 Controller 层
   - 创建 Service 层
   - 创建 Mapper 层
   - 创建 Entity 实体

### 项目结构规范

```
project-root/
├── pom.xml                          # 父 POM，继承 imwe-framework
├── project-api/                     # API 模块：接口定义、DTO、枚举、常量
│   ├── src/main/java/
│   │   └── com/imwe/project/
│   │       ├── api/                 # Facade 接口
│   │       ├── param/               # 请求参数 (Param/Request 后缀)
│   │       ├── result/              # 响应结果 (Result/Response 后缀)
│   │       ├── constant/            # 常量
│   │       └── enums/               # 枚举
│   └── pom.xml
├── project-core/                    # 核心模块：业务逻辑、DAO、配置
│   ├── src/main/java/
│   │   └── com/imwe/project/core/
│   │       ├── service/             # Service 接口和实现
│   │       ├── dao/
│   │       │   ├── mapper/          # MyBatis Mapper 接口
│   │       │   └── entity/          # 数据库实体 (Model 后缀)
│   │       ├── config/              # 配置类
│   │       ├── handler/             # 处理器
│   │       └── util/                # 工具类
│   ├── src/main/resources/
│   │   └── mapper/                  # MyBatis XML 映射文件
│   └── pom.xml
└── project-service/                 # 服务模块：Controller、启动类
    ├── src/main/java/
    │   └── com/imwe/project/
    │       ├── controller/          # Controller
    │       ├── converter/           # 转换器
    │       └── ProjectApplication.java
    ├── src/main/resources/
    │   └── application.yml          # 配置文件
    └── pom.xml
```

### 示例

**命令使用：**
```bash
/imwe-framework --task=init --project=user-center
/imwe-framework --task=dependency --add=kafka
/imwe-framework --task=config --component=redis
```

**父 POM 配置示例：**
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <parent>
        <artifactId>imwe-framework</artifactId>
        <groupId>com.imwe.framework</groupId>
        <version>3.0.0-SNAPSHOT</version>
    </parent>
    <groupId>com.imwe</groupId>
    <artifactId>imwe-project</artifactId>
    <version>3.0.0-SNAPSHOT</version>
    <packaging>pom</packaging>

    <modules>
        <module>imwe-project-api</module>
        <module>imwe-project-core</module>
        <module>imwe-project-service</module>
    </modules>

    <properties>
        <project.build.sourceEncoding>UTF-8</project.build.sourceEncoding>
        <maven.compiler.source>21</maven.compiler.source>
        <maven.compiler.target>21</maven.compiler.target>
        <lombok.version>1.18.30</lombok.version>
        <imwe.framework.version>3.0.4-SNAPSHOT</imwe.framework.version>
    </properties>

    <dependencyManagement>
        <dependencies>
            <dependency>
                <groupId>com.imwe.base</groupId>
                <artifactId>imwe-framework-starter-mybatis</artifactId>
                <version>${imwe.framework.version}</version>
            </dependency>
            <dependency>
                <groupId>com.imwe.base</groupId>
                <artifactId>imwe-framework-starter-kafka</artifactId>
                <version>${imwe.framework.version}</version>
            </dependency>
            <dependency>
                <groupId>com.imwe.base</groupId>
                <artifactId>imwe-framework-starter-consul</artifactId>
                <version>${imwe.framework.version}</version>
            </dependency>
            <dependency>
                <groupId>org.projectlombok</groupId>
                <artifactId>lombok</artifactId>
                <version>${lombok.version}</version>
            </dependency>
        </dependencies>
    </dependencyManagement>
</project>
```

**API 模块 POM 示例：**
```xml
<artifactId>imwe-project-api</artifactId>
<packaging>jar</packaging>

<dependencies>
    <dependency>
        <groupId>com.imwe.framework</groupId>
        <artifactId>imwe-framework-starter</artifactId>
        <version>3.0.0-SNAPSHOT</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.30</version>
        <scope>provided</scope>
    </dependency>
    <dependency>
        <groupId>javax.validation</groupId>
        <artifactId>validation-api</artifactId>
        <version>2.0.1.Final</version>
    </dependency>
</dependencies>

<build>
    <plugins>
        <plugin>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-maven-plugin</artifactId>
            <configuration>
                <skip>true</skip>
            </configuration>
        </plugin>
    </plugins>
</build>
```

**Core 模块 POM 示例：**
```xml
<artifactId>imwe-project-core</artifactId>
<packaging>jar</packaging>

<dependencies>
    <dependency>
        <groupId>com.imwe</groupId>
        <artifactId>imwe-project-api</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.imwe.framework</groupId>
        <artifactId>imwe-framework-starter</artifactId>
        <version>${project.version}</version>
    </dependency>
    <dependency>
        <groupId>com.imwe.base</groupId>
        <artifactId>imwe-framework-starter-mybatis</artifactId>
    </dependency>
    <dependency>
        <groupId>com.imwe.base</groupId>
        <artifactId>imwe-framework-starter-kafka</artifactId>
        <version>3.0.4-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
    </dependency>
</dependencies>
```

**Service 模块 POM 示例：**
```xml
<artifactId>imwe-project-service</artifactId>
<packaging>jar</packaging>

<dependencies>
    <dependency>
        <groupId>com.imwe.framework</groupId>
        <artifactId>imwe-framework-starter</artifactId>
        <version>3.0.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.imwe.base</groupId>
        <artifactId>imwe-framework-starter-consul</artifactId>
        <version>3.0.4-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.imwe</groupId>
        <artifactId>imwe-project-core</artifactId>
        <version>3.0.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>com.imwe</groupId>
        <artifactId>imwe-project-api</artifactId>
        <version>3.0.0-SNAPSHOT</version>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <version>1.18.30</version>
    </dependency>
</dependencies>
```

**application.yml 配置示例：**
```yaml
server:
  port: 18206
  servlet:
    context-path: /

spring:
  application:
    name: imwe-project-service
  cloud:
    consul:
      discovery:
        instance-id: ${spring.application.name}-${spring.cloud.client.ip-address}-${server.port}
        tags: urlprefix-/rpc/imwe-project-service

mybatis:
  mapper-locations: classpath*:mapper/**/*.xml
  type-aliases-package: com.imwe.project.core.dao.entity

lock:
  redisson:
    enabled: true

imwe:
  datasource:
    enabled: true
    defaultSourceName: imwe
    groups:
      main:
        url: jdbc:mysql://${jdbc.host}:3306/db_name?useOldAliasMetadataBehavior=true&autoReconnect=true&allowMultiQueries=true&useSSL=false
        username: ${jdbc.user}
        password: ${jdbc.pwd}

  encrypt:
    mybatis:
      enabled: true
      algorithm: AES
      secret-key: ${imwe.encrypt.secret-key}

  data:
    redis:
      enabled: true
      client-type: LETTUCE
      groups:
        master:
          cluster:
            nodes:
              - ${redis.host}:${redis.port}
          lettuce:
            pool:
              enabled: true
              max-active: 20
              max-idle: 10
              min-idle: 5
```

**启动类示例：**
```java
@EnableDiscoveryClient
@ComponentScan(basePackages = {
    "com.imwe.project"
})
@EnableFeignClients
@MapperScan("com.imwe.project.core.dao.mapper")
@Slf4j
@SpringBootApplication(exclude = {
    DataSourceAutoConfiguration.class,
    MongoAutoConfiguration.class,
    DataSourceTransactionManagerAutoConfiguration.class
})
public class ProjectApplication {

    public static void main(String[] args) {
        SpringApplication.run(ProjectApplication.class, args);
        log.info("Project service started successfully!");
    }
}
```

**Controller 示例：**
```java
@Slf4j
@RestController
@RequestMapping("/rpc/imwe-project-service/api/path")
public class SomeController {

    @Autowired
    private SomeService someService;

    @PostMapping("/action")
    public ResultDTO doAction(@RequestBody SomeParam request) {
        log.info("Received request - param: {}", request.getParam());
        return someService.doAction(request);
    }
}
```

**Service 示例：**
```java
@Slf4j
@Service
public class SomeServiceImpl implements SomeService {

    @Autowired
    private SomeMapper someMapper;

    @Override
    @Transactional(rollbackFor = Exception.class)
    public String save(SomeSaveParam param, String operator) {
        SomeModel model;
        boolean isUpdate = false;

        if (param.getId() != null && !param.getId().isEmpty()) {
            model = someMapper.findById(param.getId());
            if (model == null) {
                throw new BizException(ErrorCode.NOT_FOUND);
            }
            isUpdate = true;
        } else {
            model = new SomeModel();
            model.setId(UUID.randomUUID().toString().replace("-", ""));
        }

        BeanUtils.copyProperties(param, model);
        model.setUpdater(operator);

        if (isUpdate) {
            someMapper.updateById(model);
        } else {
            model.setCreator(operator);
            someMapper.insert(model);
        }

        return model.getId();
    }

    @Override
    public PageResult<SomeResult> queryList(int pageNum, int pageSize) {
        int offset = (pageNum - 1) * pageSize;
        List<SomeModel> models = someMapper.queryList(offset, pageSize);
        int total = someMapper.count();

        List<SomeResult> results = models.stream()
                .map(this::convertToResult)
                .collect(Collectors.toList());

        return PageResult.<SomeResult>builder()
                .records(results)
                .total((long) total)
                .pageNum(pageNum)
                .pageSize(pageSize)
                .build();
    }
}
```

**Mapper 示例：**
```java
@Mapper
public interface SomeMapper {

    int insert(@Param("model") SomeModel model);

    int updateById(@Param("model") SomeModel model);

    int deleteById(@Param("id") String id);

    SomeModel findById(@Param("id") String id);

    List<SomeModel> queryList(@Param("offset") Integer offset, @Param("limit") Integer limit);

    int count();
}
```

**Entity 示例：**
```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class SomeModel implements Serializable {

    private static final long serialVersionUID = 1L;

    private String id;
    private String field;
    private Integer status;
    private String creator;
    private Date createTime;
    private String updater;
    private Date updateTime;
}
```

### 参数说明

| 参数 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `task` | String | 是 | 任务类型（init, dependency, config, application, structure） |
| `project` | String | 否 | 项目名称（init 任务时必填） |
| `add` | String | 否 | 要添加的依赖（mybatis, kafka, consul, redis） |
| `component` | String | 否 | 配置组件（datasource, redis, kafka） |

### 任务类型说明

| 类型 | 说明 |
|------|------|
| `init` | 初始化项目结构 |
| `dependency` | 添加/管理依赖 |
| `config` | 配置组件 |
| `application` | 配置启动类 |
| `structure` | 创建代码结构 |

### 框架 Starter 依赖

| Starter | 说明 | 使用场景 |
|---------|------|----------|
| `imwe-framework-starter` | 基础框架 | 所有项目必需 |
| `imwe-framework-starter-mybatis` | 数据库访问 | 需要数据库操作 |
| `imwe-framework-starter-kafka` | 消息队列 | 需要消息队列 |
| `imwe-framework-starter-consul` | 服务发现 | 微服务注册发现 |

### 配置检查清单

**项目结构检查：**
- [ ] 父 POM 继承 imwe-framework
- [ ] 包含 api/core/service 三个模块
- [ ] 模块间依赖关系正确（service → core → api）
- [ ] 各模块 pom.xml 配置正确

**依赖检查：**
- [ ] API 模块依赖使用 provided 作用域
- [ ] Core 模块依赖 imwe-framework-starter-mybatis
- [ ] Service 模块依赖 imwe-framework-starter-consul
- [ ] Lombok 版本统一

**配置文件检查：**
- [ ] application.yml 配置数据源
- [ ] application.yml 配置 Redis
- [ ] application.yml 配置服务发现
- [ ] MyBatis mapper 路径正确
- [ ] 数据库加密配置正确

**启动类检查：**
- [ ] @EnableDiscoveryClient 注解
- [ ] @ComponentScan 配置正确包路径
- [ ] @MapperScan 配置 Mapper 路径
- [ ] 排除默认数据源自动配置

**代码结构检查：**
- [ ] Controller 使用 @RestController
- [ ] Service 使用 @Service 和 @Transactional
- [ ] Mapper 使用 @Mapper 注解
- [ ] Entity 使用 Lombok 注解

### URL 路由规范

| 用途 | 前缀 | 示例 | Facade |
|------|------|------|--------|
| 服务间调用 | `/rpc/` | `/rpc/imwe-project-service/api/action` | 需要创建 XxxFacade |
| 外部调用 | `/api/` | `/api/imwe-project-service/api/action` | 可选 |

**Facade 示例：**
```java
// API 模块中创建
@FeignClient(contextId = "captchaFacade", name = "imwe-message-center-service",
        url = "${feign.url.imwe-message-center-service:}", path = "/rpc/imwe-message-center-service")
public interface CaptchaFacade {

    @PostMapping("/msg/captcha/send")
    SendCaptchaResult sendCaptcha(@RequestBody CaptchaSendRequest request);
}
```

### 注意事项

1. **版本管理**：
   - 框架版本统一使用 `${imwe.framework.version}` 属性
   - 当前版本：`3.0.4-SNAPSHOT`
   - Java 版本：21

2. **依赖作用域**：
   - API 模块使用 `provided`（由使用者提供）
   - 其他模块默认作用域

3. **配置命名**：
   - 数据源分组使用 `imwe.datasource.groups`
   - Redis 分组使用 `imwe.data.redis.groups`
   - 业务配置使用自定义前缀

4. **Mapper XML 位置**：
   - 必须放在 `src/main/resources/mapper/` 目录下
   - XML 文件名与 Mapper 接口同名

5. **事务管理**：
   - Service 方法添加 `@Transactional(rollbackFor = Exception.class)`
   - 排除默认的 DataSourceTransactionManagerAutoConfiguration

### 常见问题

**Q: API 模块为什么要用 provided 作用域？**
A: API 模块只包含接口定义，由使用方（如 service 模块）提供实际依赖，使用 provided 避免重复打包。

**Q: 为什么要排除 DataSourceAutoConfiguration？**
A: IMWE 框架使用自定义的多数据源配置，需要排除 Spring Boot 默认的数据源自动配置。

**Q: Mapper 扫描配置在哪里？**
A: 在启动类使用 `@MapperScan("com.imwe.project.core.dao.mapper")` 注解。

**Q: 如何添加 Kafka 支持？**
A: 在 core 模块添加 `imwe-framework-starter-kafka` 依赖，并在 application.yml 中配置。

### 相关文档

- 参考：`imwe-message-center` 项目
- 目录：`.claude/skills/code-style/SKILL.md`
