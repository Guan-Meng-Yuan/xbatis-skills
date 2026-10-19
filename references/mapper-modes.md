# Mapper Modes

本分支采用 **BasicMapper 单 Mapper + Service**。适用于选择 Mapper 入口、注入、创建 Chain，以及确定业务事务边界。

## 模式选择

- 新项目或用户未指定模式时，定义统一 `XbatisMapper extends BasicMapper`。
- 不为每个实体创建 `MybatisMapper<T>`，不生成 DAO、BaseDao 或 DAO 接口。
- 已有多 Mapper 项目延续原方式，除非用户明确要求迁移；局部任务不重构已有 DAO。
- 用户当前要求、项目 `AGENTS.md` 和现有依赖优先于本技能默认值。

## 单 Mapper + Service

统一 Mapper 示意：

```java
public interface XbatisMapper extends BasicMapper {
}
```

实际 `BasicMapper` 的 import、扫描和注入根据当前依赖及容器确认。

- Spring 项目通过已有 `@MapperScan` 规则扫描统一 Mapper；沿用项目注入方式。
- Solon 项目通过当前 MyBatis 插件配置注册 Mapper，并使用项目已有 `@Db` / `@Inject` 方式注入；先确认该方式能获取 Mapper。
- Service 直接持有统一 Mapper，组织查询和业务写入；Controller 调用 Service。
- 不调用 `XbatisGlobalConfig.setSingleMapperClass(...)`，不添加 DAO 基类或 setter 转发层。

查询示意：

```java
QueryChain.of(xbatisMapper, SysUser.class)
        .eq(SysUser::getId, id)
        .list();
```

上例假设 `id` 已通过业务校验。实际实体、getter 和 API 重载必须与当前项目版本一致。

简单 CRUD 优先统一 Mapper 的内置方法：

- 查实体：`xbatisMapper.getById(Entity.class, id)`
- 查部分列：`xbatisMapper.getById(Entity.class, id, Entity::getName, ...)`
- 查单列：`xbatisMapper.getValueById(Entity.class, id, Entity::getName)`
- 删除：按当前版本确认 `deleteById(Entity.class, id)` 的行为以及逻辑删除配置
- 新增、更新、批量写入、`partialUpdate` 和 Model 重载按当前依赖源码确认

`BasicMapper` 不绑定单一实体。写入 Chain 也必须明确实体类型，使用当前版本支持的工厂，最终通过 `execute()` 执行。不要照搬多 Mapper 的工厂重载。

## 事务边界

事务由业务原子性决定，不能仅从 Mapper 或 DAO 的层级判断。

- Service 中一个业务方法涉及多表或多次写入，需要共同成功或失败时，在最外层业务操作建立事务。
- 例如创建订单、扣减余额、写入流水需要原子性时，不能只给每个独立持久化方法各开事务。
- 核验容器的事务依赖、注解、数据源接入、代理调用、传播行为及异常回滚规则；不要向 Solon 引入 Spring 注解。
- 使用同一个 Mapper 并不会自动把多个方法调用合并成事务。
- 已有 DAO 项目中，DAO 自身的原子操作可以有局部事务，但不能替代需要跨多个调用的外层事务。
- 不因事务写在 Service 就报告风险，也不把事务下移到 DAO 当作例行重构。
- 行为验证应覆盖后续写入失败时先前写入能否回滚，以及需要的并发冲突语义。

## 已有多 Mapper 项目

只在项目已使用或用户明确选择该模式时沿用：

```java
public interface SysUserMapper extends MybatisMapper<SysUser> {
}
```

- Service 直接注入对应实体 Mapper。
- 查询通常使用 `QueryChain.of(sysUserMapper)`；不要混入 BasicMapper 的实体类型重载。
- 自定义 Mapper / XML 的 namespace 跟随具体实体 Mapper。
- 不仅为遵循本分支默认值就迁移整个项目。

## XML 与自定义语句

单 Mapper 模式下，自定义 XML 的 namespace 和调用方式应围绕统一 `BasicMapper` 子接口设计。`withSqlSession(...)`、statement id 约定及相关参数按当前版本和项目既有代码确认；不要为调用 XML 临时增加一套实体 Mapper。

## 审查重点

1. 新代码是否符合 BasicMapper + Service，或保持了用户明确选择的已有模式。
2. 是否意外新增 DAO、BaseDao、DAO 接口或每实体 Mapper。
3. Service 是否注入了可用的 Mapper，BasicMapper 的 Chain 是否明确实体类型。
4. 内置 CRUD 和 Chain API 是否已按当前依赖版本确认。
5. 业务事务是否覆盖需要共同回滚的所有写入。
6. XML 的 namespace、扫描配置和 Mapper 入口是否一致。
7. 是否因更新或删除条件缺失扩大操作范围。
