# Review Checklist

用于审查 xbatis 实现的编译、数据正确性、事务、映射和项目适配。先报告行为缺陷，不把分层偏好当成风险。

## 版本与范围

- 项目已使用 xbatis，或用户明确要求接入。
- 已确认解析后的 xbatis、starter、容器、Java 与数据库版本。
- API 依据当前依赖、source JAR 或匹配 tag / commit 的源码核验；不直接用默认分支最新源码。
- 仅在无法确认所需 API 时获取缺少的对应版本源码；缓存按版本检查，不将源码加入业务仓库。
- 保留已有依赖、配置与用户改动，不顺带升级或迁移。

## BasicMapper + Service

- 本分支默认统一 `XbatisMapper extends BasicMapper`，Service 直接注入。
- 不新增 DAO、BaseDao、DAO 接口或每实体 `MybatisMapper<T>`。
- 已有多 Mapper 或 DAO 项目保留现状，除非用户明确要求迁移。
- BasicMapper 查询和写入使用匹配的实体类型与 API 重载。
- Mapper 注册、扫描和注入方式适配当前容器。
- 不主动调用 `XbatisGlobalConfig.setSingleMapperClass(...)`。

## 事务与写入

- Service 的完整业务操作涵盖需要原子提交的所有写入，不把独立调用的局部事务当作全局业务事务。
- 已核验事务依赖、代理调用、传播、隔离、异常处理和回滚行为。
- 后续写入失败时，先前需要回滚的写入确实回滚。
- Model / DTO 仅允许修改当前业务字段，不能因客户端传入就全部更新。
- 先查后改部分字段时评估 `partialUpdate(...)`，按 BasicMapper 的当前重载调用。
- 更新、删除有有效条件；必填 id、租户或权限条件缺失时失败，不通过忽略空值扩大范围。
- 写入 Chain 最终调用 `execute()`；逻辑删除场景不误用物理删除 `DeleteChain`。
- 乐观锁在实际使用的更新路径上生效，验证并发更新冲突。

## 查询与分页

- 简单 CRUD 优先内置 Mapper，按 id 查询不必生成额外 Chain。
- Chain 由 Service 创建，条件可复用 QO 与 `WhereUtil.where(this)`；不添加 DAO 转发层。
- `forSearch(true)` 仅在允许忽略空值、空字符串并 trim 的搜索场景启用。
- 可选筛选与必填业务约束分开；详情查询、唯一性校验不能因空参数丢掉关键条件。
- 局部 OR 用 `.nested(...)`，`.or()` 后需要时用 `.and()` 恢复连接关系。
- 分页使用当前项目的 `Pager`；验证一对多 join 的结果行、总数和优化行为。
- 关联数据按语义选择 `@Fetch` 或 join，不只为符合风格强行替换。
- 主表、ON 条件、VO 映射与别名正确，不因省略字段引入重复、漏列或越权数据。
- 租户上下文在实际执行路径中有效，不把可选前端筛选作为权限约束。

## 实体与映射

- 实体只承载数据库表字段，VO / DTO / QO / Model 不滥用实体注解。
- 普通字段不机械补 `@TableField`；特殊默认值、更新规则和类型处理与项目版本一致。
- 逻辑删除字段不叠加冲突的默认值配置。
- 持久化枚举的 `EnumSupport`、存储类型和反向映射按当前版本核验。
- VO 使用当前版本支持的结果映射；只返回业务需要的字段。
- Model / QO / VO 中非数据库操作字段按实际映射语义忽略。
- Lombok 已启用时可复用 `@FieldNameConstants`；未启用不要求增加依赖。

## 容器与配置

- Spring Boot 与 Solon 依赖和注解不混用。
- Solon 默认配置是 `app.yml`，环境配置是 `app-{env}.yml`，保持项目已有 YAML 格式。
- Mapper 注册和数据源 Bean 名称、配置键及插件版本一致。
- POJO 检查配置已核实当前 starter 支持，开发与生产环境的扫描按项目要求配置。
- 示例凭证使用占位符，未提交真实密钥或个人数据库配置。

## SQL 与交付

- 优先可读且合适的 Mapper / DSL / 模板；已有成熟 XML 或复杂 SQL 不因个人偏好重写。
- SQL 模板使用真实 `Methods.tpl` / `fTpl` / `cTpl` API，值参数化，不拼接不受信任输入。
- XML namespace 与单 Mapper / 既有多 Mapper 模式匹配。
- 运行项目要求的构建和必要行为验证；报告实际覆盖，不将构建通过说成业务验证通过。
- 审查结论按影响排序并引用位置；不以缺少 DAO、缺少 Lombok、事务位于 Service 为缺陷依据。
