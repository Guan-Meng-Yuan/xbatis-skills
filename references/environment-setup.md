# Environment Setup

适用于以下场景：

- 新建 xbatis 项目骨架
- 给现有 Spring Boot / Solon 项目接入 xbatis
- 生成 starter 依赖、数据源配置、启动类、Mapper 扫描配置
- 判断单 Mapper / 多 Mapper 在不同容器环境下的落地方式

## 总原则

生成环境接入代码时，先遵循这些规则：

1. 先识别当前运行环境：Spring Boot 2 / 3 / 4，还是 Solon
2. 已有项目优先跟随现有环境、已有 parent、已有 starter、已有数据源方案
3. 不要在同一模块里同时生成 Spring Boot 和 Solon 两套接入代码
4. 优先保持项目已解析的依赖版本；按各组件对应版本的 source JAR、tag / commit 核验 API，不用默认分支最新源码替代当前依赖
5. 先把容器、数据源、Mapper 扫描接起来，再生成实体、统一 BasicMapper、Service
6. 核实当前 starter 支持后，按项目要求在开发环境配置 POJO 检查；测试和生产环境不默认开启
7. Lombok 为可选依赖，已有则复用；不为遵循技能默认值强制添加

IDEA / JetBrains MCP 可用时，先按 [IDEA 工作流](idea-workflow.md) 获取目标业务项目实际加载的依赖，并与构建文件核对，再确认当前版本的 Mapper、注解及插件 API。工具或 IDE 项目不可用时使用 Maven / Gradle 解析结果与匹配版本源码，不阻塞接入工作。

## 模块总览

分析本地 xbatis 源码时，常见模块职责如下：

- `xbatis-annotation`：注解、分页字段、监听注解
- `xbatis-sql-api`：SQL 抽象接口与调用解析
- `xbatis-sql-api-impl`：数据库函数实现，核心入口是 `db.sql.api.impl.cmd.Methods`
- `xbatis-core`：Mapper、链式 DSL、分页、租户、逻辑删除、MVC 支撑
- `xbatis-bom` / `xbatis-parent`：依赖管理与父 POM
- `xbatis-ddl-auto`：DDL SQL 生成与结构增量执行；与 core 分别版本管理，配置/API/模式详见 [DDL 指南](ddl.md)

## 全局配置

真实类名以本地 xbatis 源码为准，常见入口是 `cn.xbatis.core.XbatisGlobalConfig`。
一般不建议主动调用这些配置。

配置点：

- `setTableUnderline(...)` / `setColumnUnderline(...)`：下划线策略
- `setDatabaseCaseRule(...)`：数据库大小写规则
- `setDynamicValue(...)`：注册动态默认值
- `setLogicDeleteInterceptor(...)` / `setLogicDeleteSwitch(...)`：逻辑删除行为
- `setGlobalOnInsertListener(...)` / `setGlobalOnUpdateListener(...)`：全局写入监听
- `setPagingProcessor(...)`：按数据库定制分页
- `addMapperMethodInterceptor(...)`：扩展 Mapper 方法拦截

内置动态值以源码为准，常见值包括：

- `{BLANK}`
- `{EMPTY}`
- `{NOW}`
- `{TODAY}`

## 版本选择

已有项目保持当前兼容版本，仅在明确接入或升级任务中按对应环境选择依赖；示例不能作为升级理由：

- Spring Boot 2：`xbatis-spring-boot-parent` 的普通版本线
- Spring Boot 3：`xbatis-spring-boot-parent` 的 `-spring-boot3` 版本线
- Spring Boot 4：`xbatis-spring-boot-parent` 的 `-spring-boot4` 版本线
- Solon：`xbatis-solon-plugin`

不要把 skill 示例版本或本地任意 checkout 的 README 当成当前依赖的依据。先识别构建文件和解析后的实际版本，再查看对应版本资料；无现有依赖的新项目才按容器版本评估兼容版本：

- 本地 `xbatis-source/README.md`
- 本地 `xbatis-source/README.zh-CN.md`
- 官方站对应章节：springboot2 / springboot3 / springboot4 / solon

实用接入约定：

- 搜索接口优先显式声明 `mybatis.configuration.databaseId`，减少数据库自动识别开销
- AI 生成新项目时，优先同时生成 Mapper、实体、VO、条件 DTO 和开发环境安全检查配置，避免只生成半套结构

## Spring Boot 接入

### Maven 依赖

Spring Boot 项目优先使用：

```xml
<dependencyManagement>
    <dependencies>
        <dependency>
            <groupId>cn.xbatis</groupId>
            <artifactId>xbatis-spring-boot-parent</artifactId>
            <version>${xbatis.version}</version>
            <type>pom</type>
            <scope>import</scope>
        </dependency>
    </dependencies>
</dependencyManagement>

<dependencies>
    <dependency>
        <groupId>cn.xbatis</groupId>
        <artifactId>xbatis-spring-boot-starter</artifactId>
    </dependency>
</dependencies>
```

补充规则：

- `xbatis.version` 按当前 Spring Boot 主版本选择对应版本线
- Spring Boot 2 通常使用 `xbatis-spring-boot-starter`
- Spring Boot 3 通常使用 `xbatis-spring-boot3-starter`
- Spring Boot 4 通常使用 `xbatis-spring-boot4-starter`
- JDBC 驱动必须补，例如 `mysql-connector-j`
- 连接池优先复用 Spring Boot 默认或项目现有方案；不要无故再引入另一套数据源实现
- 仅项目已有或用户要求使用 Lombok 时增加依赖；版本跟随项目依赖管理

项目选择使用 Lombok 时的可选依赖示例：

```xml
<dependency>
    <groupId>org.projectlombok</groupId>
    <artifactId>lombok</artifactId>
    <optional>true</optional>
</dependency>
```

### 数据源配置

优先在 `application.yml` / `application.yaml` 中生成：

```yaml
spring:
  datasource:
    url: ${DB_URL}
    username: ${DB_USERNAME}
    password: ${DB_PASSWORD}
```

只有项目已有显式 `DataSource` Bean 习惯，或需要特殊数据源能力时，才再生成 `@Bean DataSource`。

### 启动类与 Mapper 扫描

基础写法：

```java
@SpringBootApplication
@MapperScan("com.example.mapper")
public class XbatisApplication {

    public static void main(String[] args) {
        SpringApplication.run(XbatisApplication.class, args);
    }
}
```

单 Mapper 模式补充规则：

- 如果项目统一定义 `XbatisMapper extends BasicMapper`，优先扫描统一 Mapper 所在包
- 如果项目已使用 `markerInterface = BasicMapper.class`，延续现有配置
- 单 Mapper 模式下不需要主动生成或调用 `XbatisGlobalConfig.setSingleMapperClass(...)`

多 Mapper 模式补充规则：

- `@MapperScan` 优先扫描实体 Mapper 包
- 不要为多 Mapper 项目强行切成单 Mapper 风格

### POJO 安全检查

当前 starter 支持且项目启用时，在开发环境配置 POJO 检查，检查 VO、Model、条件对象、排序对象的映射和注解缺口；不因缺少该可选能力升级依赖。

Spring / Spring Boot 项目优先使用当前 starter 支持的 `@XbatisPojoCheckScan`：

`basePackages` 方案：

```java
@Profile("dev")
@Configuration
@XbatisPojoCheckScan(
    basePackages = "com.example"
)
public class XbatisSafeCheckConfig {
}
```

细分包路径方案：

```java
@Profile("dev")
@Configuration
@XbatisPojoCheckScan(
    modelPackages = "com.example.model",
    resultEntityPackages = "com.example.vo",
    conditionTargetPackages = "com.example.qo",
    orderByTargetPackages = "com.example.qo"
)
public class XbatisSafeCheckConfig {
}
```

注意：

- 旧 README / starter 文档记录的注解包名是 `org.mybatis.spring.boot.autoconfigure.XbatisPojoCheckScan`
- 当前 xbatis core 源码里没有该类型；生成 import 前必须从当前项目依赖或本地 starter 源码确认真实包名
- 新项目默认只生成 dev profile 下的安全检查配置；测试和生产环境不要求默认开启
- `basePackages` 和 `modelPackages` / `resultEntityPackages` / `conditionTargetPackages` / `orderByTargetPackages` 是二选一的扫描方案，不要同时生成
- 扫描范围必须覆盖 VO、Model、QO、排序对象所在包，不要只扫描实体包

## Solon 接入

### Maven 依赖

Solon 项目优先使用：

```xml
<!-- 注意顺序 -->
<dependency>
    <groupId>cn.xbatis</groupId>
    <artifactId>xbatis-solon-plugin</artifactId>
    <version>${xbatis.solon.version}</version>
</dependency>

<dependency>
    <groupId>org.noear</groupId>
    <artifactId>mybatis-solon-plugin</artifactId>
    <version>${mybatis.solon.version}</version>
</dependency>
```

规则：

- `xbatis-solon-plugin` 放在 `mybatis-solon-plugin` 前面
- 保持项目兼容版本，按对应版本源码或官方 Solon 章节确认，不直接复制最新分支版本
- MyBatis Solon 插件在 xbatis Solon 插件中是 optional，需显式声明；使用 Solon parent / BOM 时其版本优先沿用项目管理。需要 DDL 时还要显式加入 `xbatis-ddl-auto`，不能认为已传递安装，见 [DDL 指南](ddl.md)。
- 仅项目已有或用户要求使用 Lombok 时增加依赖，版本沿用项目依赖管理

### app.yml 与环境配置

Solon 默认加载 `app.yml`，按 `solon.env` 加载 `app-{env}.yml`。复用项目现有配置及嵌套 YAML 格式，不新建未配置加载的 `solon.yml`。

`app.yml` 中的数据源及 Mapper 示例：

```yaml
ds:
  jdbcUrl: ${DB_URL}
  driverClassName: com.mysql.cj.jdbc.Driver
  username: ${DB_USERNAME}
  password: ${DB_PASSWORD}

mybatis:
  master:
    mappers:
      - "com.example.mapper"
```

示例使用 MySQL 与下文 HikariDataSource Bean；仅在项目选择该方案时补齐对应 JDBC 驱动、连接池依赖及环境变量。项目已有数据源则沿用，不新增第二套。存在 XML 时再按需添加 XML 扫描路径。

只有当前 starter 版本支持 POJO 检查时，再在开发环境配置 `app-dev.yml` 中声明，例如：

```yaml
mybatis:
  master:
    pojoCheck:
      basePackages: com.example
```

- 开发运行时按项目约定选择 `dev` 环境，例如 `--env=dev`，不要让生产配置默认加载开发检查。
- 扫描范围二选一：`basePackages`，或当前版本支持的 `modelPackages` / `resultEntityPackages` / `conditionTargetPackages` / `orderByTargetPackages`；不叠加两套方案。
- `mybatis.<数据源 bean 名称>` 与数据源 Bean 名称对应，多数据源逐个核验。
- 生产 / 测试环境是否检查遵循项目要求，不因示例开启全环境扫描。

### DataSource Bean

Solon 项目优先保持和官方示例一致：

```java
@Configuration
public class MybatisConfig {

    @Bean(name = "master", typed = true)
    public DataSource dataSource(@Inject("${ds}") HikariDataSource ds) {
        return ds;
    }
}
```

### 使用方式

Solon 中如果项目已把 Mapper 托管到容器：

- 优先沿用项目现有注入方式
- `@Db`、`@Inject`、直接注入 Mapper 哪种是现有规范，就跟随哪种

不要在 Solon 项目里生成 Spring 的 `@Autowired`、`@SpringBootApplication`、`application.yml` 风格代码。

## 代码生成时的默认落地顺序

生成接入代码时，优先按这个顺序：

1. 识别环境：Spring Boot 2 / 3 / 4 或 Solon
2. 生成或对齐 xbatis 依赖
3. 生成或对齐 JDBC 驱动 / 数据源配置
4. 生成启动类、容器配置类、`@MapperScan` 或 Solon Bean
5. 当前 starter 支持且项目启用时，生成 dev 环境的 xbatis POJO 检查配置
6. 再生成统一 BasicMapper、Service、实体及按需使用的 QO、VO、Model，不生成 DAO / BaseDao

## 环境接入后的生成检查

提交前检查：

1. 依赖是否和当前环境主版本匹配
2. Spring Boot 2 / 3 / 4 是否选对了对应版本线
3. Solon 插件顺序是否正确
4. 数据源配置是否和容器风格一致
5. `@MapperScan` 是否和单 Mapper / 多 Mapper 模式一致
6. 是否没有同时混入另一套容器代码
7. Service 是否直接注入可用的统一 BasicMapper，是否避免新增 DAO / BaseDao
8. 已启用的 dev POJO 检查是否覆盖实际使用的 POJO 包，test/prod 是否按项目要求配置

## 不推荐行为

1. 不识别当前环境就直接生成依赖
2. 在 Spring Boot 项目里生成 Solon 配置
3. 在 Solon 项目里生成 Spring Boot 启动类
4. 不核对版本线就硬编码旧版 `xbatis.version`
5. 单 Mapper / 多 Mapper 还没判断，就先生成 `@MapperScan`
6. 项目已要求启用 POJO 检查，却漏掉对应 dev 配置
7. 在项目未要求时，为 test/prod 默认开启 POJO 安全检查
