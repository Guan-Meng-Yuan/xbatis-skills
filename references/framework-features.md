# Framework Features

适用于以下场景：

- 逻辑删除
- 多租户
- 乐观锁
- 动态值和默认值
- 结果映射和枚举展示
- 对象转条件 / 对象转排序

## 逻辑删除

真实类型以本地源码为准，常见类型：

- 注解：`@LogicDelete`、`@LogicDeleteTime`
- 开关：`cn.xbatis.core.logicDelete.LogicDeleteSwitch`
- 工具：`cn.xbatis.core.logicDelete.LogicDeleteUtil`

要点：

- 使用逻辑删除注解，而不是手工写删除标记条件
- 注意 `DeleteChain` 删除时逻辑删除不会生效
- 物理删除场景在 Service 中使用与 BasicMapper 匹配、显式绑定实体类型的 `DeleteChain` 工厂
- 可以通过全局拦截器补额外字段，例如删除人、删除时间
- 存在全局开关和局部开关

Agent 规则：

1. 如果项目使用逻辑删除，优先复用框架能力
2. 如果需求是“查已删除数据”，先检查是否应局部关闭逻辑删除
3. 不要在普通查询里反复手工拼 `deleted = 0`
4. 不要把 `DeleteChain` 当成逻辑删除入口；需要逻辑删除语义时优先使用框架删除方法
5. 实体字段已有 `@LogicDelete` / `@LogicDeleteTime` 等逻辑删除注解时，禁止通过 `@TableField` 配置 `defaultValue` / `updateDefaultValue` 等默认值相关属性

## 多租户

真实类型以本地源码为准，常见类型：

- 注解：`cn.xbatis.db.annotations.TenantId`
- 上下文：`cn.xbatis.core.tenant.TenantContext`

要点：

- 围绕实体类自动注入租户条件
- 常见于 `from`、`join`、`delete`、`update`
- Service 通过当前版本的 Chain 工厂绑定统一 BasicMapper 和实体类型，不增加 DAO 层
- 租户值通常通过 `TenantContext.registerTenantGetter(...)` 获取

Agent 规则：

1. 租户字段优先用 `@TenantId`
2. 查询、更新、删除优先走框架租户能力
3. 不要在业务代码里遍地补 `tenant_id`
4. 如果项目通过 `ThreadLocal` 提供租户值，保持现有上下文传递方式

## 动态数据源

xbatis 生态中可能通过外部能力支持动态数据源，例如 `cn.xbatis:xbatis-datasource-routing`。这类能力使用前必须确认项目依赖已接入。

常见能力：

- `@DS("master")`
- 主从、分组、连接池与配置解密
- 事务切换时需要注意传播行为

## 乐观锁

真实注解以本地源码为准，常见注解是 `cn.xbatis.db.annotations.Version`。

要点：

- `save` 时版本字段自动初始化
- `update(实体类)`、`delete(实体类)` 自动追加版本条件

Agent 规则：

1. 需要并发写保护时优先使用 `@Version`
2. 不要手工给每个 update 写版本判断逻辑

## 动态值 / 默认值

适合场景：

- 创建时间
- 修改时间
- 创建人
- 修改人
- 时间区间动态默认值

Agent 规则：

1. 优先用框架动态值和插入/更新事件
2. 不要在 Controller / Service 层重复填这些审计字段
3. 创建时间字段优先使用 `@TableField(defaultValue = "{NOW}", update = false)`
4. 修改时间字段优先使用 `@TableField(defaultValue = "{NOW}", updateDefaultValue = "{NOW}", updateDefaultValueFillAlways = true)`
5. 逻辑删除字段已有逻辑删除注解时，禁止补 `@TableField` 默认值配置
6. 普通字段无特殊语义时，不要为了声明列名机械补 `@TableField`
7. 项目已使用 Lombok 时可复用 `@FieldNameConstants`；未使用时不强制添加依赖
8. 实体类只保留数据库表字段，禁止混入非数据库表字段

## 结果映射 / 枚举展示

Agent 规则：

1. 返回 VO 时，优先让框架自动映射，而不是手工 copy
2. 需要返回枚举名称时，优先使用 `@PutEnumValue`
3. 新增、生成或改写持久化枚举类必须实现/继承（Java 代码使用 `implements`）`cn.xbatis.core.mybatis.typeHandler.EnumSupport<T>`，通过 `getCode()` 提供数据库存储值
4. 新增、生成或改写持久化枚举时必须提供 `of(T code)` 静态方法，按 `code` 匹配，找不到时返回 `null`
5. 枚举字段类型 `T` 按数据库实际存储类型选择，不要默认使用 `ordinal()` 或 `name()`
6. 不要为了展示字段把实体类硬改成接口响应模型
7. 实体类注解只能写在实体类上，禁止写在 VO、DTO、QO、Model 等其他类上
8. VO 优先使用真实存在的结果映射注解，例如 `@ResultEntity`、`@NestedResultEntity`、`@NestedResultEntityField`、`@ResultCalcField`
9. 注解涉及字段依赖时，复用项目字段常量；已使用 Lombok 时可用 `@FieldNameConstants` 生成的 `Fields`，例如 `SysUser.Fields.id`
10. VO 中凡是不是数据库操作字段、需要忽略的字段，优先使用 `@Ignore` 或 `@Ignores`

## 修改入参 / 更新字段范围

Agent 规则：

1. 新增或修改用于新增、更新的 POJO 时，优先判断是否应作为 xbatis Model 使用
2. 修改操作优先使用 Model 类作为传参载体
3. Model 可以像实体类一样直接参与 `save`、`update`
4. 能通过 Model 直接写入时，不要先把 Model / DTO 手工 copy 成实体类再调用 `save` / `update`
5. 根据接口语义缩小允许修改的字段范围
6. 更新时只更新本次业务真正允许变化的字段
7. 不要把查询实体、响应 VO 或全量实体直接作为修改入参使用
8. 不要因为前端传了字段就默认全部参与 update
9. Model 中凡是不是数据库操作字段、需要忽略的字段，优先使用 `@Ignore` 或 `@Ignores`
10. 单条数据如果是“先查询出来，再改部分字段”的场景，优先使用 `partialUpdate(...)` 精准修改

## 对象转条件 / 对象转排序

适合场景：

- 搜索表单
- 后台筛选
- 动态排序
- 复杂条件对象

Agent 规则：

1. 查询对象优先实现 `cn.xbatis.core.mvc.QO`
2. `QO.where()` 负责集中构建 `Where`
3. 条件对象优先声明 `@ConditionTarget`
4. 字段条件优先使用 `@Condition`、`@Conditions`、`@ConditionGroup`
5. `QO.where()` 中优先使用 `WhereUtil.where(this)`
6. Service 接收查询对象，将 `qo.where()` 传入当前版本支持的 BasicMapper Chain 条件入口
7. 搜索对象优先映射成条件对象
8. 多列关键字搜索优先使用条件注解体系
9. 多字段排序优先使用对象转排序
10. 非搜索查询需要按值忽略条件时，优先使用 predicate 重载，例如 `.eq(SysUser::getId, id, Objects::nonNull)`
11. 无法使用 predicate 重载时，再使用 boolean `when` 重载，例如 `.eq(Objects.nonNull(id), SysUser::getId, id)`
12. 需要括号包裹一组条件，尤其组内存在 `or` 条件时，使用 `.nested(chain -> ...)`
13. 条件拼接默认是 `and`；调用 `.or()` 后会持续使用 `or`，需要切回时调用 `.and()`
14. 局部 `or` 条件优先放进 `.nested(...)`，避免 `.or()` 状态影响后续外层条件
15. 不要为动态筛选写一长串 `if`
16. 可复用 where 条件集中在 QO 或 Service 的查询构建方法中，不增加 DAO 转发层
17. 排序通过框架安全字段映射在 Service 中构建，不拼接客户端原始 `order by`
18. 使用 `Methods` 风格时，优先静态导入所需方法
19. 方法嵌套优先写成 `eq(gt(...))` 这类形式，避免混用多种调用风格
20. QO 中凡是不是数据库操作字段、需要忽略的字段，优先使用 `@Ignore` 或 `@Ignores`

## 分表

常见类型以本地源码为准：

- `@SplitTable`
- `@SplitTableKey`
- `TableSplitter`

Agent 规则：

1. 分表实体必须确认分表键字段和分表策略真实存在
2. 条件中必须带上分表键，否则无法定位真实表
3. `TableSplitter#support(Class<?>)` 用于声明支持的分片键类型
4. `TableSplitter#split(String, Object)` 用于计算真实表名
5. 链式查询和更新与普通实体基本一致，但不能遗漏分表键条件
