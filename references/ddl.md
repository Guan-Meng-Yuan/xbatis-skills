# DDL 使用指南

用于接入 `xbatis-ddl-auto`、从实体生成建表 SQL、按数据库现状计算增量，以及配置 Solon 启动 DDL。DDL 不依赖 DAO；业务仍采用统一 BasicMapper + Service。

内容索引：版本与来源 → 依赖 → 实体与索引 → SQL 生成/预览/执行 → 模式 → Solon 配置 → 多库/分表/序列 → 故障与验证。

## 核验基线与来源

本指南于 2026-10-19，通过业务项目 IDEA / JetBrains MCP 加载的 Maven source JAR 核验：

| 组件 | 核验版本 | 重点源码 |
| --- | --- | --- |
| `cn.xbatis:xbatis-solon-plugin` | `1.10.13` | `cn.xbatis.solon.integration.XbatisAdapterDefault`、`XbatisDDLAutoItem`、`PropsUtil` |
| `cn.xbatis:xbatis-ddl-auto` | `1.10.2` | `cn.xbatis.ddl.auto.DDLAuto`、`Mode`、`DefaultDDLAutoExecutor`、`DefaultDDLBuilder` |
| `cn.xbatis:xbatis-core` / `xbatis-annotation` | `1.10.13` | `ColumnDefinition`、`TableDefinition`、`Index`、`IndexField`、`TableId` |
| `org.noear:mybatis-solon-plugin` / Solon | `4.1.0` | 数据源与 Mapper 集成；实际 MyBatis 为 `3.5.17` |

在 IDEA 外部库中定位这些类型；源码通常位于 Maven 本地仓库 `cn/xbatis/<artifact>/<version>/<artifact>-<version>-sources.jar`。换版本后重新核验，不能假设 DDL 与 core 同版本。

对应发布源码：[Solon 插件 source JAR](https://repo.maven.apache.org/maven2/cn/xbatis/xbatis-solon-plugin/1.10.13/xbatis-solon-plugin-1.10.13-sources.jar)、[DDL source JAR](https://repo.maven.apache.org/maven2/cn/xbatis/xbatis-ddl-auto/1.10.2/xbatis-ddl-auto-1.10.2-sources.jar)、[注解 source JAR](https://repo.maven.apache.org/maven2/cn/xbatis/xbatis-annotation/1.10.13/xbatis-annotation-1.10.13-sources.jar)。[官网 DDL 入口](https://xbatis.cn/zh-CN/function/core/ddlAuto.html)用于导航；其旧版示例和部分注释与当前执行器存在差异，行为以匹配版本实现为准。

## 1. 安装依赖

已使用 Solon `4.1.0` parent / BOM 的项目，依赖示例：

```xml
<properties>
    <xbatis.version>1.10.13</xbatis.version>
    <xbatis-ddl.version>1.10.2</xbatis-ddl.version>
</properties>

<dependencies>
    <dependency>
        <groupId>cn.xbatis</groupId>
        <artifactId>xbatis-solon-plugin</artifactId>
        <version>${xbatis.version}</version>
    </dependency>
    <dependency>
        <groupId>org.noear</groupId>
        <artifactId>mybatis-solon-plugin</artifactId>
    </dependency>
    <dependency>
        <groupId>cn.xbatis</groupId>
        <artifactId>xbatis-ddl-auto</artifactId>
        <version>${xbatis-ddl.version}</version>
    </dependency>
</dependencies>
```

- MyBatis Solon 插件和 DDL 在 xbatis Solon 插件中是 optional，使用时需要显式声明。
- 保持 xbatis Solon 插件在 MyBatis Solon 插件前，核验启动使用的是 `XbatisAdapterDefault`。
- 该组合解析后的 core / annotation / SQL API 都是 `1.10.13`。DDL 自身 POM 引用 core `1.10.8`，项目解析结果必须检查，避免错误混搭；已有复杂依赖管理时按项目需要统一版本。
- 连接数据库还需要 JDBC 驱动和可用的 `DataSource`；沿用项目已有连接池与数据源方案，见 [环境接入](environment-setup.md)。仅安装这三个模块不会自动创建数据库、数据源或业务表。
- Spring 项目沿用自己的 xbatis starter，再增加 DDL 模块；不要引入 Solon 插件。容器自动扫描注解需从所用 starter 对应源码核验，本指南的 Java 门面 API 不依赖容器。

在后端模块中核对：

```bash
mvn dependency:tree '-Dincludes=cn.xbatis:*,org.noear:mybatis-solon-plugin,org.mybatis:mybatis'
mvn dependency:resolve-sources -DincludeGroupIds=cn.xbatis,org.noear
```

在 IDEA 同步 Maven 后再次核对外部库版本；不要用同步前的索引判断 API。

## 2. 实体、列和索引怎么写

下面是独立的 DDL 示例，不代表业务已经有这些表。DDL 读取字段元数据，不要求为了示例安装 Lombok；用于 Getter DSL / CRUD 时按项目风格补访问器。

```java
package com.example.entity;

import cn.xbatis.db.annotations.ColumnDefinition;
import cn.xbatis.db.annotations.Index;
import cn.xbatis.db.annotations.IndexField;
import cn.xbatis.db.annotations.Table;
import cn.xbatis.db.annotations.TableDefinition;
import cn.xbatis.db.annotations.TableField;
import cn.xbatis.db.annotations.TableId;
import java.math.BigDecimal;
import java.time.LocalDateTime;

@Table("ddl_demo_device")
@TableDefinition(comment = "DDL 示例设备")
@Index(name = "uk_ddl_demo_device_code", unique = true,
       fields = {@IndexField(name = "deviceCode")})
public class DdlDemoDevice {
    @TableId
    private Long id;

    @ColumnDefinition(length = 64, nullable = false, comment = "设备编号")
    private String deviceCode;

    @ColumnDefinition(precision = 12, scale = 2, nullable = false, defaultValue = "0")
    private BigDecimal price;

    @ColumnDefinition(defaultValue = "CURRENT_TIMESTAMP")
    private LocalDateTime createdAt;

    @TableField(exists = false)
    private String displayLabel;
}
```

### 映射与 DDL 元数据

| 写法 | 作用 / 注意 |
| --- | --- |
| `@Table(value, schema)` | 表名和 schema；数据库/catalog/schema 不能靠示例猜测，连接目标须事先存在 |
| `@Table(table = false)` | 不参与 DDL，适用于视图等；执行器也会跳过识别为视图的已有对象 |
| `@TableId` | 默认 `IdAutoType.AUTO`，按方言生成数据库自增；不要把业务分配 ID 误写成数据库自增 |
| `@TableId(IdAutoType.NONE)` | 自行提供 ID；需导入 `cn.xbatis.db.IdAutoType` |
| `@TableField("column_name")` | 显式列映射；普通字段沿用项目命名策略，不机械添加 |
| `@TableField(exists = false)` | 不生成该列，不能靠忽略字段来保留 SYNC 下的数据库独有列 |
| `@ColumnDefinition(index = 1)` | 建表字段排列顺序，**不是数据库索引** |
| `length / precision / scale` | 字符长度、十进制精度/小数位；Java 类型与方言共同决定最终类型 |
| `nullable` | 默认 true；主键强制非空；已有列的 nullable 差异不是自动同步目标 |
| `unique` | 单列唯一约束；复合唯一索引用类级 `@Index(unique = true, ...)` |
| `defaultValue` | SQL 表达式，字符串字面量自行写 SQL 引号，例如 `"'READY'"`，数字 `"0"` 不加引号 |
| `definition` | 自定义列类型片段，例如 `"DECIMAL(12,0)"`；构建器还会追加自增、唯一、非空、默认值等，不要重复写这些属性 |
| `comment` | 按数据库能力生成注释，不能保证所有方言效果一致 |
| `javaType` | 覆盖 DDL 类型推断，不是 MyBatis TypeHandler 或运行时字段转换 |
| `@TableDefinition` | 表注释和建表语句尾部 `definition`；例如 MySQL ENGINE 选项，只在确定该方言时使用 |

`definition`、`defaultValue` 和表尾片段直接参与 SQL，应为受控源码常量，不接收请求参数。

`@ColumnDefinition` / `@TableDefinition` 可重复声明并用 `dbType` 区分方言。当前选择规则是：只有一个注解时直接使用；多个时先匹配数据库名，再选 UNKNOWN 通用项，仍无匹配时回退第一个。多方言项目应提供通用定义，不能把 dbType 当作“不匹配就忽略这个注解”的开关。

常见默认类型：String → VARCHAR(255)，UUID → VARCHAR(36)，Long → BIGINT，Integer → INTEGER，BigDecimal → DECIMAL(19,2)，LocalDate → DATE；LocalDateTime 在 MySQL 是 DATETIME，其他常见方言是 TIMESTAMP。Boolean、二进制、时区时间、Oracle/SQL Server 字符串等有方言差异，先输出实际 SQL。未知 Java 类型默认可能退回字符串，不能据此认定 JSON / 自定义类型可正确落库。

当前推断对 BigDecimal 的 `scale = 0` 会回退到默认 2；需要零小数时显式 `definition = "DECIMAL(12,0)"`，并核对目标数据库 SQL。

`ColumnDefinitionValue.of` 的默认值顺序为：显式 DDL defaultValue → 逻辑删除初值 → `@TableField.defaultValue` → 乐观锁初值 1。部分动态日期/布尔值会转换成数据库表达式，不能假设所有 ORM 动态值/Java 初始化值都会变成数据库 DEFAULT。`defaultValue = ""` 本身不表示 SQL 空字符串；空字符串默认值用 `"''"`。

### 索引

- 导入 `cn.xbatis.db.annotations.Index` / `IndexField`，不要误用 JPA 注解。
- `@IndexField(name = "deviceCode")` 推荐填实体字段名；当前实现也能回退按真实列名查找，需保证映射存在。
- 多列按 `fields` 的顺序组成联合索引；方向用 `direction = IndexDirection.ASC / DESC`（`cn.xbatis.db.IndexDirection`）。
- 未指定 name 时框架生成名称。单表显式命名便于稳定维护；多物理分表禁止显式索引名，见下文。
- UPDATE 按索引名称补缺失索引，不比较同名索引定义；修改同名索引字段、顺序或唯一性，不能只改实体后期待 UPDATE 自动处理。
- 不同时用列 unique 和类级 unique Index 重复声明同一唯一约束。
- 新代码不生成已废弃的 `@ForeignKey`；外键、数据回填、主键调整等使用经核验的迁移 SQL。

## 3. 区分离线生成、数据库差异和真正执行

### 3.1 只生成完整建表 SQL，不连接数据库

```java
import cn.xbatis.ddl.auto.DDLAuto;
import db.sql.api.DbType;
import java.util.List;

List<String> statements = DDLAuto.of(DbType.MYSQL)
        .add(DdlDemoDevice.class)
        .sqlList();
statements.forEach(System.out::println);
```

结果可能包含多条：序列、建表、注释、索引，不能只取第一条。`sqlList()` 无参数时 CREATE / UPDATE / SYNC 都输出完整创建 SQL，并不计算已有数据库差异；NONE 返回空列表。

### 3.2 按已有库结构生成待执行 SQL

```java
import cn.xbatis.ddl.auto.Mode;

List<String> pending = DDLAuto.of(DbType.MYSQL)
        .mode(Mode.UPDATE)
        .add(DdlDemoDevice.class)
        .sqlList(dataSource); // javax.sql.DataSource，使用已有连接时传 Connection
pending.forEach(System.out::println);
```

CREATE / UPDATE 在这里读取元数据并返回业务 DDL，不执行返回的语句。元数据可见范围必须正确，否则已有表可能被误判为不存在。

**SYNC 差异预览有副作用。** 当前 `sqlList(DataSource / Connection)` 同样调用类型探测；需要比较已有列类型时，可能在连接的默认 schema 创建并删除 `xbatis_ddl_auto_type_probe`，且先删除该名称的已有表。它是固定名称的普通探测表，不是保证会话隔离的临时表。不能把该重载一概视为严格只读，也不能在业务库把这个名字用于普通表。需要严格只读时用离线 `sqlList()`，或自行用元数据/外部迁移工具比较；SYNC 差异计算应在隔离副本中进行，避免并发探测。

### 3.3 执行

```java
DDLAuto.of(DbType.MYSQL)
        .mode(Mode.UPDATE)
        .add(DdlDemoDevice.class)
        .execute(dataSource);
```

`execute` 会重新读取库状态、重新计算并执行；它不是“应用上次 pending 列表”。需要保证审阅内容与执行内容完全一致时，导出 SQL 后按项目发布流程执行该文件。

DataSource 重载自行获取并关闭连接；Connection 重载由调用者管理连接。执行器逐条 JDBC Statement 执行，失败会抛异常，不能承诺所有数据库 DDL 事务回滚或整批原子性。不能用 Service 的 `@Transaction` 代替迁移恢复方案。

可用 `.executionListener(new DDLExecutionListener() { ... })` 记录实际执行：`beforeExecute(String, List<String>)`、`afterExecute(String, List<String>)`、`onExecuteError(String, SQLException, List<String>)`。默认执行器还提供 `getExecutedSqlList()`；门面没有该 getter，可通过监听器收集。探测表语句直接执行，不经过上述监听路径；监听结果不等于全部数据库副作用。

## 4. 模式的实际行为

以下指 `execute` 或带连接的 `sqlList`，不是离线无参 `sqlList()`：

| 模式 | 表不存在 | 表已存在 |
| --- | --- | --- |
| NONE | 不处理 | 不处理 |
| CREATE（默认） | 创建表及相应索引/注释/序列 | 保留现状，不补已有表的缺列/缺索引 |
| UPDATE | 创建 | 补缺列、缺失名称的索引及需要的序列；不删除多余对象，不比较同名列/索引定义 |
| SYNC | 创建 | 补缺并同步列/索引，可能删除多余列/索引、重建变化索引，修改类型/长度/精度/默认值；支持时还处理列注释和自增差异 |

注意 `Mode.UPDATE` 的枚举注释只强调新增字段，但当前 `createSqlList` 实现还调用 `createAddIndexSqlList`。这是源码核验确认的行为。

- SYNC 不删除业务表，不识别字段重命名，也不以 nullable 差异自动生成修改；不能称为完整迁移系统。
- SYNC 会保护读取到的主键列及部分主键/唯一约束索引，**不能据此推断数据库中所有人工索引或独有列均被保留**。实体必须完整反映纳入同步的表。
- 重命名字段可能被视为删旧列加新列，数据不会自动搬迁。类型缩小、NOT NULL 新列、唯一索引都可能因已有数据而失败。
- 默认使用 NONE（不启用）或开发环境 UPDATE；只有明确需要并已核对 SQL 时才选择 SYNC，不能因“同步”这个名称默认开启。

## 5. Solon 自动 DDL 配置

### 数据源和 BasicMapper

先按 [环境接入](environment-setup.md) 注册可用的命名数据源 `master`、安装选定数据库驱动和连接池。Mapper 包里保持统一 Mapper：

```java
package com.example.mapper;

import cn.xbatis.core.mybatis.mapper.BasicMapper;

public interface XbatisMapper extends BasicMapper {
}
```

Service 使用 Solon `@Component` + `org.apache.ibatis.solon.annotation.Db("master")` 注入该 Mapper。DDL 扫描实体，Mapper 扫描接口，两者分别配置；不为建表生成 DAO 或每实体 Mapper。

`app.yml` 保留 Mapper 配置，默认关闭 DDL：

```yaml
mybatis:
  master:
    mappers:
      - "com.example.mapper"
    ddlAuto:
      - entityPackages: "com.example.entity.**"
        mode: NONE
```

只在 `app-dev.yml` 配置开发启用项（按项目约定 `--env=dev` 启动）：

```yaml
mybatis:
  master:
    ddlAuto:
      - entityPackages: "com.example.entity.**"
        mode: UPDATE
        dbType: MYSQL
```

仅在项目已经选择 MySQL 时填写 MYSQL，其他数据库使用当前 `db.sql.api.DbType` / `DbTypes` 支持的真实名称；例如 PostgreSQL 是 `PGSQL`，不要猜写 POSTGRESQL。Solon 插件没有独立的 `ddlAuto.enabled`、`dryRun` 或回滚开关；未配置 ddlAuto 或 mode NONE 即不执行。`ddlAuto` 是列表，不能直接把 entityPackages 写到它下面当单个对象。

### 每个列表项的字段

| 键 | 类型/默认 | 说明 |
| --- | --- | --- |
| entityPackages | String，必填 | Solon 扫描表达式；多个表达式写一个逗号分隔字符串，不是 YAML 数组；避免逗号后加空格 |
| mode | NONE / CREATE / UPDATE / SYNC；省略为 CREATE | 即使 NONE 也应填写非空 entityPackages，因为插件先检查它 |
| dbType | 可选 String | 省略时从目标 DataSource 推断，自动推断需要连接数据库 |
| dataSource | 可选 String | 目标 DataSource 的 Bean 名；省略时使用当前 mybatis.master 对应数据源；不是 JDBC URL |
| makerInterface | 可选 Class 的完整名称 | **真实拼写是 makerInterface**，筛选可赋值给该接口/父类的实体；不要写 markerInterface |

扫描后只有带 `cn.xbatis.db.annotations.Table` 的类被加入。不要扫整个 classpath，更不能假设 marker 过滤代替表清单核验。

自动执行发生在 `XbatisAdapterDefault.initConfiguration` 内：初始化 xbatis configuration → POJO 检查 → DDL。完成后发布 `cn.xbatis.solon.integration.XbatisDDLAutoCompleteEvent`；不宜再在任意 `@Init` 中重复执行一遍 DDL。该事件不能证明有实体被扫描或有 SQL 被执行；应同时检查扫描数量和库元数据。没有对应数据源适配器时，单独写 YAML 不会建表。

当前插件按 dataSource 名查 DataSource Bean；其路由数据源反射回退分支有无法正常命中的逻辑，不能承诺配置某个路由内部 key 就能选择物理库。多库优先显式命名实际 DataSource Bean，或用 Java API 指定已经解析到的物理 DataSource。

## 6. 多库、分表和序列

### 多数据源

把配置放在各自的 `mybatis.<Bean名称>.ddlAuto` 下，实体包分开；同包多库可用 marker 接口过滤，例如：

```yaml
mybatis:
  archive:
    ddlAuto:
      - entityPackages: "com.example.archive.entity.**"
        mode: UPDATE
        dbType: MYSQL
        dataSource: archive
        makerInterface: com.example.archive.ArchiveEntity
```

实体实现这个真实存在的接口。分别核对连接、catalog/schema 和扫描名单；不把主库实体误扫进归档库。BasicMapper 按命名数据源注入，不增加 DAO。

### 物理分表

```java
import cn.xbatis.ddl.auto.DDLTableNameResolverUtil;
import java.util.Arrays;

DDLTableNameResolverUtil.set(DdlDemoDevice.class,
        base -> Arrays.asList(base + "_202610", base + "_202611"));
```

注册必须早于 DDL。返回物理表名，不含 `schema.`；schema 使用 `@Table.schema`。空列表跳过该实体，null、空白表名非法。这个 resolver 只控制 DDL 建哪些物理表，不等于配置运行时 CRUD 分表路由。

多物理表时**移除示例中的显式 `@Index.name`**，由框架按物理表生成索引名，否则会抛异常。该工具是进程内静态配置，短期工具/测试用完可 `remove(Entity.class)`；不要在自动 DDL 执行前过早清理。

### 序列主键

当前构建器会从 `@TableId(IdAutoType.SQL, sql = ...)` 解析序列引用，例如 PostgreSQL：

```java
@TableId(value = IdAutoType.SQL, dbType = DbType.Name.PGSQL,
         sql = "SELECT nextval('ddl_demo_device_seq')")
private Long id;
```

还识别 Oracle `sequence.NEXTVAL` 和 SQL Server / DB2 `NEXT VALUE FOR sequence`。生成列表包含需要的 CREATE SEQUENCE，但不负责自定义序列初值/步长、业务最大 ID 回填或通用序列迁移。已有数据库的序列权限、schema、初值和增长策略须单独核验。

## 7. 方言、发布与故障定位

- ORM 支持某数据库不代表 DDL 的每个操作都支持。`DDLDialect` 对 MySQL/MariaDB、PostgreSQL 模型、Oracle/DM、SQL Server、H2、DB2、SQLite 等有不同分支；目标数据库版本和 JDBC 驱动需实际验证。
- SQLite、ClickHouse 的修改列能力受限；ClickHouse 不支持本模块的普通 CREATE INDEX / UNIQUE。不要宣称 SYNC 跨库等价。
- 生产默认不启用启动改表。按项目发布流程保存审阅后的 SQL、在目标库副本验证、安排执行和备份恢复；框架没有迁移版本历史、分布式迁移锁或自动数据回填，不假设多实例同时启动安全。
- 已存在表的新增非空列先考虑历史数据回填；字段重命名、缩窄、外键/主键变动用明确迁移步骤。DDL 报错可能已经部分成功，先核对实际结构及成功/失败 SQL，再决定后续，不盲目重跑。

| 现象 | 优先检查 |
| --- | --- |
| 找不到 DDL 类 / NoSuchMethodError | 是否显式加入 DDL 与 MyBatis Solon 插件；core 版本仲裁；IDEA 是否同步 |
| 未建表 / 扫描为 0 | 当前环境、命名数据源、ddlAuto 列表、entityPackages 表达式、@Table、makerInterface 实现关系 |
| 写了 markerInterface 没过滤 | Solon 实际属性是 makerInterface，改后核验扫描名单 |
| 不补缺列 / 同名索引不变 | CREATE 不改已有表；UPDATE 不比较已有列与同名索引定义 |
| SQL 反复创建已有对象 | catalog/schema、大小写/quoted 名称、JDBC 元数据可见权限；不要靠切换 SYNC 掩盖 |
| 预览要求 CREATE / DROP 权限 | 是否使用了 SYNC 的带连接 sqlList，类型探测会有副作用 |
| 执行中断 | 查看失败 SQL、已成功语句、约束/数据冲突、驱动方言；不得把失败描述成全部回滚 |

## 8. 验证和交付要求

1. 依赖、配置、实体样例先编译；IDEA 核对真实声明，运行项目要求的 Maven / Gradle 构建。
2. 无真实数据库时，可验证离线 SQL；行为验证在隔离库中进行。核对 CREATE 重复运行无变更、UPDATE 补列/索引但保留历史列、SYNC 删除/修改范围、失败后实际状态。
3. Solon 配置验证扫描数量、选中的 DataSource / 环境、BasicMapper 注入，以及默认环境没有启动 DDL。SQL 模块能运行不等于容器接入正确。
4. 报告实际验证的数据库及版本。H2 结果不能当 MySQL、PostgreSQL 或生产测试通过；未连接目标数据库时明确写出未验证的部分。

本基线已在临时 H2 `2.4.240` 内存库验证离线 SQL、零小数精度、序列 SQL、分表索引限制、NONE/CREATE/UPDATE/SYNC、重复执行、探测表副作用与监听器；在 Solon `4.1.0` 验证默认关闭/dev 开启、YAML 列表、实体扫描、makerInterface、多数据源选择和 BasicMapper 注入/查询。MySQL/PostgreSQL 实际执行、真实业务数据迁移及生产恢复未验证。
