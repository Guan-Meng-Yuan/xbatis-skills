---
name: xbatis-skills
description: 在已使用 xbatis，或用户明确要求接入 xbatis 的 Java 项目中实现、改写、审查或优化持久化代码时使用。支持 Spring Boot 与 Solon；本分支默认使用 BasicMapper 单 Mapper + Service，不生成 DAO。涵盖 Chain、分页、映射及框架特性，不用于直接连接数据库查询数据或自动迁移其他 ORM 项目。
---

# Xbatis Skills

使用 xbatis 原生能力实现持久化代码。本分支默认采用统一 `XbatisMapper extends BasicMapper`，Service 直接注入 Mapper 并组织查询和业务事务，不生成 DAO、BaseDao 或 DAO 接口。

## 任务边界与项目约定

- 用户当前要求和项目 `AGENTS.md`、依赖、配置约定优先于本技能的默认建议。
- 先确认项目已使用 xbatis，或用户明确要求接入；不要仅因项目使用 Java、Solon 或 MyBatis 就增加 xbatis 依赖。
- 新项目或用户未指定 Mapper 模式时，使用 `BasicMapper` 单 Mapper + Service。已有多 Mapper 项目保持原模式，除非用户明确要求迁移。
- 不新增 DAO 层；已有 DAO 的局部任务保留现有代码，不自动删除或重构。
- Lombok 是可选工具。已有则沿用；未使用时不要仅为字段常量或本技能偏好新增依赖。
- 不自动升级依赖，不修改无关配置，不把本技能示例类名当作项目真实类型。

## API 与版本核验

1. 先检查项目构建文件及解析后的依赖，确认 xbatis、starter、容器、Java 和数据库版本。
2. IDEA / JetBrains MCP 可用且已打开目标业务项目时，优先获取项目加载的依赖，再搜索并读取依赖源码或反编译 class；具体流程见 [IDEA 工作流](references/idea-workflow.md)。IDEA 不可用时使用构建工具解析结果、source JAR 或缓存的对应版本源码；本地目录存在不代表版本匹配。
3. 仅在现有资料不能确认所需 API 时，按各组件自己的依赖版本从官方仓库获取对应 tag / commit，并记录版本与来源。
4. 源码放在临时目录或工具缓存中，不写入业务仓库。不使用默认分支的最新源码替代项目依赖版本。
5. 无法确定版本映射、类名、包名、签名或泛型时明确说明待核实，不猜测调用，也不为迁就示例升级依赖。

官方源码：

- [xbatis](https://github.com/xbatis/xbatis) / [Gitee](https://gitee.com/xbatis/xbatis)
- [Spring Boot starter](https://github.com/xbatis/xbatis-spring-boot-parent) / [Gitee](https://gitee.com/xbatis/xbatis-spring-boot-parent)
- [Solon plugin](https://github.com/xbatis/xbatis-solon-plugin) / [Gitee](https://gitee.com/xbatis/xbatis-solon-plugin)

## 按任务读取参考

只读取当前任务相关文件；参考中的 API 仍需匹配项目版本。

| 任务 | 参考 |
| --- | --- |
| 利用 IDEA 核验依赖/API、检查和编译 Java 改动、核对表结构 | [idea-workflow](references/idea-workflow.md) |
| Spring Boot / Solon 依赖、数据源、扫描与 POJO 检查 | [environment-setup](references/environment-setup.md) |
| BasicMapper、Service 注入、事务边界及已有多 Mapper 项目 | [mapper-modes](references/mapper-modes.md) |
| 实体、Model、内置 Mapper、局部更新、冲突写入 | [entity-and-mapper](references/entity-and-mapper.md) |
| Chain、对象条件与结果映射注解 | [query-dsl](references/query-dsl.md) |
| 分页、搜索、联表和 VO | [query-strategy](references/query-strategy.md) |
| 租户、逻辑删除、乐观锁、动态值、分表 | [framework-features](references/framework-features.md) |
| SQL 模板、动态 SQL 与数据库方言 | [advanced-sql](references/advanced-sql.md) |
| XML / 原生 SQL 的使用边界 | [xml-boundaries](references/xml-boundaries.md) |
| 实现审查与交付检查 | [review-checklist](references/review-checklist.md) |

## BasicMapper + Service 工作方式

- 定义统一的 `XbatisMapper extends BasicMapper`，不为每个实体新建 `MybatisMapper<T>`。
- Mapper 扫描与注入方式跟随容器。Spring 和 Solon 分别使用自己的注解，不混用。
- Service 直接注入统一 Mapper，优先调用内置 CRUD；Chain 在 Service 中通过当前版本支持的工厂创建。
- `BasicMapper` 不绑定单一实体，查询、更新和删除均按真实方法签名明确实体类型，不照搬多 Mapper 示例。
- 查询示意：`QueryChain.of(xbatisMapper, SysUser.class)`；实体和字段必须在项目中真实存在。
- 按 id 查实体或部分列优先 `getById(Entity.class, id, ...)`，查单列优先 `getValueById(Entity.class, id, ...)`；生成前核验具体重载。
- 不主动调用 `XbatisGlobalConfig.setSingleMapperClass(...)`，也不为了使用 Chain 增加 DAO 包装层。

## 事务边界

- 事务覆盖完整业务操作；Service 编排的多个数据库写入若需要共同成功或回滚，应置于同一个外层业务事务。
- 不因使用同一个 BasicMapper 就假设多次调用自动处于同一事务；核验事务集成、代理调用、传播、隔离与回滚行为。
- 事务注解跟随当前容器和版本；不要向 Solon 引入 Spring 事务依赖或注解。
- 涉及跨表写入时验证后续步骤失败能否回滚先前写入。不要以注解所在层级代替行为验证。

## 实现决策

### 查询与结果

- 优先内置 Mapper、Chain、对象转条件和映射能力，复杂表达再评估 SQL 模板；XML / 原生 SQL 是补充方案。
- 搜索条件对象可实现 `cn.xbatis.core.mvc.QO`，通过 `WhereUtil.where(this)` 收敛条件；按当前版本的入口传给 Chain。
- `forSearch(true)` 仅用于允许忽略空值并 trim 的搜索场景；详情、唯一性检查、内部业务约束不默认启用。
- 可选条件可使用 predicate 或 boolean `when` 重载；必填 id、租户或权限条件缺失时应失败，不得静默去掉条件。
- 局部 OR 使用 `.nested(...)`；调用 `.or()` 后按需用 `.and()` 切回，避免改变后续条件含义。
- 分页优先 `paging(Pager)`。一对多联表验证分页结果、count 和 join 优化是否正确。
- 顺带补充关联信息可评估 `@Fetch`；真正的 join 明确主表与 ON 条件。VO 优先 `returnType`、`select(VO.class)` 与结果映射。
- Controller 接收和输出项目约定的 DTO / QO / VO；不要为接口展示字段污染持久化实体。

### 写入与实体

- 实体只承载表字段，使用真实 `@Table`、`@TableId` 等注解；普通字段不机械补 `@TableField`。
- Model 或 DTO 仅包含当前业务允许写入的字段；复用项目已有映射方式，不扩大客户端可修改范围。
- 单条先查后改部分字段可使用 `partialUpdate(...)`；确认当前版本的 BasicMapper 重载，不照搬省略实体类型的调用。
- 写入 Chain 使用与 BasicMapper 匹配的工厂，并以 `execute()` 执行；更新、删除条件不能为空或因忽略空值而扩大范围。
- `DeleteChain` 不作为逻辑删除入口，需要逻辑删除时使用当前版本支持的框架删除能力。
- 逻辑删除字段不叠加冲突的默认值配置；多租户、乐观锁和动态值优先使用框架能力并验证实际效果。
- Lombok 已启用时可复用 `@FieldNameConstants`；未启用时使用现有字段常量等方案，不强制添加依赖。

## 环境与验证

- Spring Boot 和 Solon 分别按已有依赖、配置文件和容器注解接入，禁止混入另一套框架。
- Solon 默认配置使用 `app.yml`，环境配置使用 `app-{env}.yml`；复用嵌套 YAML 等项目格式。
- POJO 检查仅在已核实当前 starter 支持后配置；开发环境按项目约定开启，不能把校验开关当作运行时授权或租户隔离。
- 不在示例中硬编码真实凭证；使用环境变量或项目约定的占位符。
- Java 代码修改后，IDEA 可用时先检查修改文件，再编译相关模块；仍须运行项目要求的 Maven / Gradle 构建和必要行为验证，IDEA 编译不能替代打包。涉及事务、分页、逻辑删除、租户隔离时验证对应行为。纯技能文档改动检查格式与引用，不触发业务项目编译。
- 审查先报告数据正确性、事务、安全及编译问题，避免把本技能的默认偏好当作既有代码的缺陷。
