# Entity And Mapper

适用于以下场景：

- 新增或审查实体注解
- 判断 `@TableField` 是否必要
- 选择 `MybatisMapper<T>` / `BasicMapper` 的默认 CRUD 能力
- 新增或修改用于写入的 Model / POJO
- 单条数据先查后改，需要精准局部修改
- 处理主键或唯一键冲突写入

## 实体注解

实体类注解真实包名优先按本地 xbatis 源码确认；常见注解在 `cn.xbatis.db.annotations` 下。

- `@Table`
  - 定义表名、`schema`
  - 可设置列名规则和数据库大小写规则（推荐默认即可，不需要配置）
- `@TableId`
  - 支持自增、生成器、SQL 生成等主键策略
  - 可按数据库类型差异化配置
- `@TableField`
  - 控制字段是否参与 `select` / `insert` / `update`
  - 可配置 `neverUpdate`、`exists = false`
  - 可配置 `jdbcType`、`typeHandler`
  - 可配置 `defaultValue` / `updateDefaultValue`
- `@LogicDelete`
  - 定义删除前后值
  - `afterValue` 支持动态值
  - 字段已有逻辑删除注解时，禁止通过 `@TableField` 配置 `defaultValue` / `updateDefaultValue` 等默认值相关属性
- `@LogicDeleteTime`
  - 记录逻辑删除时间
  - 字段已有逻辑删除注解时，禁止通过 `@TableField` 配置 `defaultValue` / `updateDefaultValue` 等默认值相关属性
- `@TenantId`
  - 标记租户字段
- `@Version`
  - 标记乐观锁字段

生命周期注解不在 `cn.xbatis.db.annotations` 下，真实包名是：

- `cn.xbatis.listener.annotations.OnInsert`
- `cn.xbatis.listener.annotations.OnUpdate`

## 其他常用注解

以下注解同样常见于 `cn.xbatis.db.annotations`，但不要都当成实体字段注解使用。生成或审查代码时先按用途区分实体、Model、QO、排序对象和 VO。

### Model 与忽略字段

- `@ModelEntityField`
  - 用在 Model 字段上，用于 Model 字段名和实体属性名不一致的场景
  - 可配置 `forceUpdate` 强制参与更新，或 `ignoreDefaultValue` 不继承实体默认值规则
  - Model 字段和实体字段同名时不需要机械补充
- `@Ignore`
  - 用在 VO、QO、DTO、Model 等 POJO 字段上，表示该字段不参与 xbatis 处理
  - 实体类应只保留数据库表字段，不要靠 `@Ignore` 长期容纳展示字段或业务临时字段
- `@Ignores`
  - 用在 VO、QO、DTO、Model 等 POJO 类上，通过字段名数组声明需要忽略的字段
  - 适合集中声明多个忽略字段；单个字段优先直接在字段上使用 `@Ignore`

### 对象条件与排序

- `@ConditionTarget`
  - 用在查询条件对象类上，声明条件默认作用的实体类型
  - 可配置 `storey` 区分同实体多层映射，可配置 `logic` 控制默认 AND / OR 组合
- `@Condition`
  - 用在条件对象字段上，声明 EQ、NE、IN、LIKE、BETWEEN、NULL 等条件类型
  - 字段名不一致时配置 `property`；字段引用复用项目已有常量，已使用 Lombok 时可用 `@FieldNameConstants`
  - LIKE 场景按需配置 `likeMode`，日期结束时间按需配置 `toEndDayTime`
  - 可配置 `defaultValue`、`cast`，但不要用它替代业务层必须显式校验的入参规则
- `@Conditions`
  - `@Condition` 的重复注解容器，用于一个入参字段映射多个条件
  - 典型场景是关键词同时匹配多个列
- `@ConditionGroup` / `@ConditionGroups`
  - 用在条件对象类上，将多个字段组织成一组条件并指定组内逻辑
  - 适合需要括号语义的 OR / AND 组合，不要在 Service 层散落手工拼 where
- `@OrderByTarget`
  - 用在排序对象类上，声明排序默认作用的实体类型
  - `strict = true` 时只匹配带排序注解的字段，适合对外排序参数需要白名单控制的接口
- `@OrderBy`
  - 用在排序对象字段上，将该字段映射到实体属性排序
  - 字段顺序影响排序优先级，生成排序对象时要保持声明顺序有业务含义
- `@OrderByColumn`
  - 用在排序对象字段上，直接指定列名排序
  - 只有实体属性映射不够表达时才使用，不要优先写硬编码列名
- `@OrderByAsField`
  - 用于 `.as(Entity::getXxx)` 这类 select 别名字段的排序映射
  - VO / 别名排序优先使用字段引用，不要默认退回字符串别名

### VO 结果映射

- `@ResultEntity`
  - 用在 VO 类上，声明 VO 与实体的结果映射关系
  - VO 和实体字段同名时优先靠它自动 select / 自动映射
  - 多表或同实体多层映射时按需配置 `storey`
- `@ResultEntityField`
  - 用在 VO 字段或 getter 上，解决 VO 字段与实体属性命名不一致或字段冲突
  - 推荐优先于 `@NestedResultEntityField` 使用
- `@ResultField`
  - 用在 VO 字段或 getter 上，表示该字段来自非表真实列，例如聚合列、计算列或自定义 select 别名
  - 可按需配置 `jdbcType`、`typeHandler`
- `@ResultCalcField`
  - 用在 VO 字段或 getter 上，声明 count、sum、max、min 等计算列
  - 聚合查询优先用它表达结果列，不要为了一个聚合字段直接退回 XML
- `@NestedResultEntity`
  - 用在 VO 嵌套对象字段或 getter 上，声明嵌套对象对应的实体映射
  - 多层嵌套结果再使用，不要把简单扁平 VO 复杂化
- `@NestedResultEntityField`
  - 用在嵌套 VO 字段或 getter 上，解决嵌套对象字段名不一致
  - 源码注释推荐优先使用更强的 `@ResultEntityField`
- `@Fetch`
  - 用在 VO 字段或 getter 上，用于自动级联或补充字段
  - 支持源字段 / 列、目标实体、目标属性、中间表、排序、limit、内存 limit、缓存、逻辑删除策略等能力
  - 适合“查询 A 顺带带出 B”的场景，先考虑它再考虑 join
- `@PutEnumValue`
  - 用在 VO 字段或 getter 上，根据源实体枚举 code 填充枚举展示值
  - 可配置 `code`、`value`、`required`、`defaultValue`
- `@PutValue`
  - 用在 VO 字段或 getter 上，通过指定 factory 静态方法补充值，并带 session 级缓存语义
  - 只在确实需要外部工厂换算或补值时使用，不要替代普通字段映射
- `@CreatedEvent`
  - 用在 VO 类上，在 VO 创建后调用指定类的 `onCreatedEvent` 静态方法
  - 适合结果对象创建后的集中补充处理，不要把数据库查询条件或写入逻辑放进去
- `@TypeHandler`
  - 用在 VO 类或字段上，为 VO 结果字段指定自定义 MyBatis TypeHandler
  - 实体字段类型处理优先使用 `@TableField(typeHandler = ...)` 或项目既有类型处理规范

### 分表与分页辅助

- `@SplitTable`
  - 用在分表实体类上，指定 `TableSplitter` 实现
  - `strict` 控制跨分区行为，生成前必须确认项目分表策略和分表键
- `@SplitTableKey`
  - 用在分表实体字段上，标记分表键
  - 查询、更新、删除分表实体时必须确保条件中带分表键
- `@Paging`
  - 用在 XML Mapper 方法上，要求参数和返回值符合 `Pager` 约定
  - 只在项目确实使用 XML 分页方法时考虑；常规分页优先使用 `paging(Pager)`

### DDL 与兼容注解

完整依赖、注解、索引、SQL 预览/执行、Solon 配置及模式限制见 [DDL 使用指南](ddl.md)，不要只补列注解而遗漏容器接入和执行边界。

- `@ColumnDefinition`
  - 用于列 DDL 元数据，例如长度、唯一性、nullable、precision、scale、definition、comment
  - 只有项目使用 xbatis 自动建表 / DDL 能力时才补，不要为了普通 ORM 映射机械添加
- `@ForeignKey`
  - 源码中已标记 `@Deprecated`
  - 不要在新代码中主动生成；遇到旧代码时按兼容存量处理

## 枚举约定

新增、生成或改写持久化枚举时，枚举类必须完整遵循 xbatis 枚举契约，并实现/继承（Java 代码使用 `implements`）真实接口 `cn.xbatis.core.mybatis.typeHandler.EnumSupport<T>`：

```java
public enum Status implements EnumSupport<Integer> {

    ENABLED(1),
    DISABLED(0);

    private final Integer code;

    Status(Integer code) {
        this.code = code;
    }

    @Override
    public Integer getCode() {
        return code;
    }

    public static Status of(Integer code) {
        for (Status item : values()) {
            if (java.util.Objects.equals(item.getCode(), code)) {
                return item;
            }
        }
        return null;
    }
}
```

规则：

- 枚举通过 `getCode()` 返回数据库存储值
- 枚举生成时必须提供 `of(T code)` 静态方法，找不到匹配项返回 `null`
- `T` 按数据库字段实际类型选择，例如 `String`、`Integer`、`Long`
- 不要默认依赖 `ordinal()`、`name()` 或手写 TypeHandler
- 需要返回枚举名称或展示值时，优先配合 `@PutEnumValue`

## 字段注解规则

- 创建时间字段优先使用 `@TableField(defaultValue = "{NOW}", update = false)`
- 修改时间字段优先使用 `@TableField(defaultValue = "{NOW}", updateDefaultValue = "{NOW}", updateDefaultValueFillAlways = true)`
- 普通字段无特殊语义时，不要为了声明列名机械补 `@TableField`
- 只有需要控制默认值、更新行为、类型处理、查询参与、非表字段等特殊语义时才补 `@TableField`
- 实体类只保留数据库表字段；VO、QO、DTO、Model 中需要忽略的字段优先使用 `@Ignore` 或 `@Ignores`
- 实体注解只能写在实体类上，禁止写在 VO、DTO、QO、Model 等非实体类上

## Model 修改约定

新增或修改用于新增、更新的 POJO 时，优先判断是否应作为 xbatis Model 使用。

规则：

- 修改操作优先使用 Model 类作为传参载体，收敛本次业务允许写入的字段
- Model 可以像实体类一样直接参与 `save(Model)`、`update(Model)`
- 能使用 Model 直接写入时，不要再编写 DTO / Model / 实体之间的手工转换代码
- 只有接口契约隔离、跨上下文复用、聚合字段拆装等真实需求明确存在时，才保留额外 DTO 到 Model / 实体的转换层
- Model 中凡是不参与数据库操作、需要忽略的字段，优先使用 `@Ignore` 或 `@Ignores`

## Mapper 默认能力

本分支默认单 Mapper；下述多 Mapper 写法仅供已有多 Mapper 项目沿用，一个实体对应一个 `MybatisMapper<T>`：

```java
public interface SysUserMapper extends MybatisMapper<SysUser> {
}
```

特点：

- `QueryChain.of(sysUserMapper)` 可以自动推断实体类型
- 对当前实体的简单查询，通常可以省略 `select` / `from` / `returnType`
- 更适合中小项目和边界清晰的业务模块

`MybatisMapper<T>` 默认覆盖：

- 查询：`get`、`list`、`exists`、`count`、`paging`、`cursor`、`mapWithKey`
- 新增：`save`、`saveBatch`、`saveOrUpdate`、`saveModel`
- 更新：`partialUpdate`、`update`、`updateBatch`
- 删除：`deleteById`、`deleteByIds`、`delete(where)`
- 原生 SQL：`execute`、`select`、`selectList`
- 多库差异化：`dbAdapt(...)`

按 id 查询优先复用内置方法，不默认写 `QueryChain`：

- 返回完整实体：`getById(id)`
- 返回单表部分列：`getById(id, Entity::getName, Entity::getStatus)`
- 返回单列值：`getValueById(id, Entity::getName)`

单 Mapper 模式下，统一 Mapper 继承 `BasicMapper`，链式 API 里显式传入实体类型：

```java
QueryChain.of(xbatisMapper, SysUser.class)
        .eq(SysUser::getId, 1)
        .list();
```

单 Mapper 模式不要主动生成或调用 `XbatisGlobalConfig.setSingleMapperClass(...)`。

## 精准局部修改

单条数据先查后改少量字段时，优先评估 `partialUpdate(...)`，缩小本次写入字段范围，避免全量 `update(entity)` 覆盖无关字段。

- 本分支通过 Service 注入的统一 BasicMapper 调用，不新增 DAO。
- 先用 `xbatisMapper.getById(Entity.class, id)` 获取实体并检查存在性与访问权限。
- 按当前依赖源码核验 BasicMapper 的 `partialUpdate` 重载及回调签名；不要直接复制多 Mapper 省略实体类型的调用。
- 需要把字段置为 `null` 时，验证所用更新 API 确实表达了置空，不能将“未提交字段”误作“清空字段”。
- 该能力控制更新字段范围，不替代业务事务、租户隔离或并发控制。

## Service 注入约定

Service 直接注入统一的 `XbatisMapper extends BasicMapper`，不生成 DAO、BaseDao 或 DAO 接口。Spring 沿用当前项目的注入方式；Solon 按当前插件能力与项目既有规范选择 `@Db` / `@Inject`，不引入 Spring 注解。

## 写入冲突策略

`save` / `saveBatch` / `InsertChain` 可按项目实际依赖确认是否支持 `onConflict`：

- 冲突忽略：`doNothing()`
- 冲突更新：`doUpdate(...)`

这是跨数据库封装能力，适合主键或唯一键冲突场景。生成代码前必须确认当前 xbatis 版本和数据库方言是否支持。

## 生成代码时的默认选择

- 新项目默认统一 `XbatisMapper extends BasicMapper`，Service 直接注入；已有多 Mapper 项目保持现状，除非用户明确要求迁移
- 实体字段优先用注解表达映射规则，不要把映射逻辑分散到业务层
- 默认使用 Lambda 属性引用，不写硬编码列名
- 不知道类路径、真实类名、方法名、注解包名或泛型签名时，先用本地 xbatis 源码确认，不要猜测
