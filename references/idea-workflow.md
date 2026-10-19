# IDEA 工作流

当目标业务项目已在 IntelliJ IDEA 中打开，且当前会话提供 JetBrains MCP 时，优先利用它核验依赖、API 和生成代码。该能力是可选增强，不是技能的安装前提；本分支仍使用 BasicMapper + Service，不新增 DAO。

## 先选对项目

- 所有支持 `projectPath` 的调用都显式传入目标业务项目的绝对路径。不要把技能仓库路径、当前 shell 目录或另一个已打开项目当作目标。
- 在多模块项目中先识别实际使用 xbatis 的后端模块，读取对应构建文件与现有代码。
- 工具可用不代表项目已导入或依赖已同步。项目不存在、索引未完成、结果不完整时说明状态；只做有依据的定向重试，不反复轮询或为此修改 IDE 全局设置。
- 不自动安装插件、打开其他项目或引入 xbatis；IDEA 不可用时直接走下文回退流程。

## 1. 确认加载的依赖

调用 `get_project_dependencies(projectPath)`，关注 xbatis、starter、MyBatis、容器、JDBC 驱动及连接池。

- 返回条目可能只有 JAR 名称而没有独立 `version` 字段，不能把缺失版本当成未安装。
- 将 IDE 加载的依赖与目标模块的 POM / Gradle 配置核对。继承、BOM 或传递依赖的实际版本以解析结果为依据，不只看声明文本。
- 若 IDE 与构建工具结果不一致，优先核对目标模块和依赖同步状态，再使用 Maven / Gradle 的实际解析结果；不要以某一侧的缓存直接断言版本兼容。
- 项目未引入 xbatis 时，不为查 API 自动添加依赖；只有用户明确要求接入时才处理依赖变更。

## 2. 核验 BasicMapper、Chain 与注解

按需搜索 `BasicMapper`、`QueryChain`、`UpdateChain`、`partialUpdate`、`@Db` 或实际事务注解。

1. 使用 `search_symbol(q, projectPath)` 查找项目中的统一 Mapper、Service 或已有调用。
2. 未找到相关框架声明时，再用 `include_external=true` 搜索依赖与 SDK。不能因默认搜索没有结果就断言 API 不存在。
3. 使用 `read_file(file_path, projectPath, offset, limit)` 读取返回的真实路径；支持 source JAR、`jar!/…` 条目和 class 反编译，无需先下载整个框架仓库。
4. 已有项目代码引用可用 `get_symbol_info(filePath, line, column, projectPath)` 查看解析到的声明、泛型和重载。文件路径相对项目根目录，行列均从 1 开始。

检查重点：

- 符号确实来自当前模块所用的依赖版本，避免把同名项目类或其他库类型当成 xbatis API。
- BasicMapper 是否需要显式实体类型，Chain 工厂及终止方法的重载、返回类型是否匹配。
- 容器注解的包名、Mapper 注入能力及事务集成是否符合 Spring / Solon 当前版本。
- 遵循返回的 `more`、`partialResultReason` 或超时信息；部分结果不能当成完整搜索结论。
- 反编译可帮助核验二进制声明和实现，但不等同于原始源码、注释或完整设计文档。遇到依赖语义、源码级注解或缺失信息时，再查匹配版本的 source JAR 或官方 tag / commit。

需要下载源码时仍遵守主文件的版本核验规则，不用默认分支最新代码替代项目依赖。

## 3. 检查与编译 Java 改动

- 对修改的 Java 文件调用 `get_file_problems(filePath, projectPath)`，处理与本次改动有关的错误及有意义的警告。默认保留警告，不顺带清理无关历史问题。
- 检查 import、方法重载、泛型、类型转换以及 IDEA 能识别的注解问题；无告警不能证明容器注入或事务运行正常。
- 调用 `build_project(projectPath, filesToRebuild)` 编译有关文件；影响依赖、模块配置或跨模块 API 时按实际范围编译，不把全量重建作为默认步骤。
- 只有未超时且返回 `isSuccess=true` 才报告 IDEA 编译成功。检查或构建结果超时、部分返回时，如实说明验证不完整。
- 项目要求的 Maven / Gradle 打包、测试仍须执行；IDEA 编译不替代 CI 构建，也不证明事务回滚、分页或租户隔离正确。
- 只有 Markdown / 技能元数据变更时做技能校验、YAML 与引用检查，不编译无关业务项目，也不把文档检查表述为 Java 验证。

## 4. 按需核对数据库映射

仅在实体生成、字段映射或 SQL 排查需要表结构时，利用用户已配置的数据源：

1. `list_database_connections(projectPath)` 定位相关连接；多个候选无法确定时先确认，不随意选择生产或其他项目的库。
2. `list_database_schemas(connectionId, projectPath)` 取得真实 database / schema 标识，不自行拼造。
3. 对选定 schema 使用 `list_schema_objects(...)` 定位所需表，再用 `get_database_object_description(...)` 查看列、类型、主键和索引。
4. 若返回 `isIntrospected=false` 或元数据过时，按当前可用的 schema introspection 能力加载必要范围，再核对结构；无法读取时使用项目 DDL / 迁移文件。

映射核验优先读取元数据，不自动预览业务行数据或执行 SQL，不创建或修改连接，不把连接凭证复制到代码、文档或日志。用户另有明确的数据查询或修改请求时按该任务范围处理；本技能本身不授予数据操作权限。

## IDEA 不可用时

- 依赖版本：读取构建文件，并使用项目已有 Maven / Gradle wrapper 或对应工具获取解析结果。
- API 声明：查看依赖对应的 source JAR、缓存源码或必要的二进制签名；仍不能确认时才获取官方对应版本源码。
- 代码验证：执行项目要求的编译、打包和相关测试；不伪造 IDEA 检查结果。
- 表结构：使用项目 DDL、迁移文件或用户已有的授权连接能力；不因缺少 IDEA 数据源就自动配置数据库。

交付时区分已确认的版本与声明、已完成的静态检查、编译结果及实际行为验证，不将其中一项替代其他项。
