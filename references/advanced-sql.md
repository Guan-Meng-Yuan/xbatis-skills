# Advanced SQL Decision

适用于以下场景：

- QueryChain 表达困难
- 报表 SQL
- 数据库方言差异明显
- 复杂函数运算

## 优先顺序

1. 先确认是否真的不能用 `QueryChain`
2. 再考虑 SQL 模板
3. 再考虑动态 query / 动态 where
4. 最后才退到 XML / 原生 SQL

## SQL 模板适用场景

适合：

- 局部函数包装
- 特殊表达式
- 仍希望和框架字段、列对象结合

模板类型：

- 命令模板优先使用 `Methods.tpl(...)`
- 函数模板优先使用 `Methods.fTpl(...)`
- 条件模板优先使用 `Methods.cTpl(...)`

Agent 要点：

1. 当只是少量表达式复杂时，不要整条 SQL 都回退原生写法
2. 优先在原有 QueryChain 结构中用模板解决局部复杂度
3. 创建 SQL 模板时，优先通过 `Methods.tpl`、`Methods.fTpl`、`Methods.cTpl` 这些官方入口创建，不要自己手写模板对象

常用函数和模板入口优先静态导入：

```java
import static db.sql.api.impl.cmd.Methods.*;
```

常见能力：

- 聚合和数学函数：`count`、`sum`、`avg`、`round`、`pow`
- 字符串函数：`concat`、`upper`、`lower`、`substring`
- 日期函数：`currentDate`、`dateDiff`、`dateAdd`、`year`、`month`、`day`、`hour`、`minute`、`second`
- `tpl(...)`：普通 SQL 片段模板
- `fTpl(...)`：函数表达式模板，可继续 `.as(...)`、`.gt(...)` 等
- `cTpl(...)`：条件模板，用于 `where` / `having`

模板占位符使用 `{0}`、`{1}` 传入字段和值，不要把用户输入拼进模板 SQL 字符串。

单列表达式直接用普通 getter；多列表达式再使用 `GetterFields.of(...)`。生成代码前必须确认实体、VO、Mapper、字段 getter 和返回字段真实存在，示例类名不能直接照抄。

### 模板示例

通用 import：

```java
import cn.xbatis.core.sql.executor.chain.QueryChain;
import db.sql.api.cmd.GetterFields;
import db.sql.api.impl.cmd.basic.OrderByDirection;

import static db.sql.api.impl.cmd.Methods.*;
```

单列表达式：

```java
QueryChain.of(xbatisMapper, SysUser.class)
        .select(SysUser::getId,
                c -> fTpl("count({0})", c).as(UserStatVo::getUserCount))
        .where(SysUser::getUserName,
                c -> cTpl("{0} like {1}", c, "%" + keyword + "%"))
        .returnType(UserStatVo.class)
        .list();
```

多列表达式：

```java
QueryChain.of(xbatisMapper, SysUser.class)
        .select(GetterFields.of(SysUser::getScore, SysUser::getExtraScore),
                cs -> fTpl(
                        "coalesce({0}, 0) + coalesce({1}, 0)",
                        cs[0],
                        cs[1]
                ).as(UserStatVo::getTotalScore))
        .returnType(UserStatVo.class)
        .list();
```

`groupBy` / `orderBy` / `having` 中也可以使用模板：

```java
QueryChain.of(xbatisMapper, SysUser.class)
        .groupBy(SysUser::getCreatedAt, c -> fTpl("date({0})", c))
        .orderBy(OrderByDirection.DESC,
                SysUser::getScore,
                c -> fTpl("coalesce({0}, 0)", c))
        .having(SysUser::getScore,
                c -> fTpl("sum({0})", c).gt(minScore))
        .list();
```

使用要点：

- 动态普通条件优先用 `.eq(when, getter, value)`、`.like(when, getter, value)` 等内置方法
- 只有跨字段表达式或框架函数未覆盖时才用 `cTpl(...)`
- 示例默认不写 `boolean when`；只有需要动态开关时，才使用 `select(boolean, ...)`、`where(boolean, ...)`、`groupBy(boolean, ...)` 等重载
- `having` 默认推荐用 `.having(...)`；只有需要显式切换连接关系时，才使用 `.havingAnd(...)` 或 `.havingOr(...)`
- 部分函数有数据库差异时，结合 `dbAdapt(...)` 处理

## 特殊字符注意

模板底层涉及 `MessageFormat` 风格占位，单引号要注意转义问题。

Agent 在生成模板时：

1. 尽量把固定值作为参数传入
2. 避免直接在模板里硬编码复杂字面量

## 原生 SQL / XML 何时合理

合理：

- 报表 SQL 结构极重
- 方言差异大
- 历史包袱明确要求保留
- xbatis 原生写法可读性明显更差

不合理：

- 简单 CRUD
- 标准列表页
- 常规联表
- 只因为开发者熟悉 XML

## 多库差异化

`QueryChain` / `InsertChain` / `UpdateChain` / `DeleteChain` 以及 Mapper 都可按源码确认是否支持 `dbAdapt(...)`。

适用场景：

- 同一逻辑在 MySQL、PostgreSQL、H2、SQL Server 上要生成不同 SQL
- 冲突处理、分页、函数调用需要按库区分

默认先用统一 DSL；只有数据库方言差异真实存在时，再包一层 `dbAdapt(...)`。
