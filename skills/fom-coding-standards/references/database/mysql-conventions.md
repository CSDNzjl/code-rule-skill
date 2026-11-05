# 数据库规约（MySQL）—— FOM3 融合版

> 本文件融合业界通用 MySQL 数据库规约与 FOM3 项目建表/ORM 约定。
>
> - 规约用【强制】/【推荐】/【参考】标注，**MySQL 数据库规约全文收录于此**，无需查阅外部文档。
> - FOM3 项目特定约定放在每章末尾"FOM3 衔接"小节，与 `references/backend/layering-architecture.md` 的 §3.1（实体）与 §8（字段约定）、§9（Mapper XML）对照阅读。
>
> 写任何建表 SQL、Mapper XML、Repository 查询、Entity 字段前，先看本文件。

## 目录

- [一、建表规约](#一建表规约)
- [二、索引规约](#二索引规约)
- [三、SQL 语句](#三sql-语句)
- [四、ORM 映射](#四orm-映射)

## 一、建表规约

1. 【强制】是/否字段使用 `is_xxx` 命名，类型为 `unsigned tinyint`，1 表示是，0 表示否。
2. 【强制】库名、表名、字段名只使用小写字母或数字，不以数字开头，不使用两个下划线包围数字。
3. 【强制】表名不使用复数名词。
4. 【强制】禁用 MySQL 保留字。
5. 【强制】主键索引名为 `pk_字段名`，唯一索引名为 `uk_字段名`，普通索引名为 `idx_字段名`。
6. 【强制】小数使用 `decimal`，禁止使用 `float` 和 `double`。
7. 【强制】字符串长度几乎相等时使用 `char`。
8. 【强制】`varchar` 长度不超过 5000；超出时使用 `text` 并独立成表。
9. 【强制】表必备 `id`、`create_time`、`update_time` 三个字段；`id` 为 `bigint unsigned` 主键。
10. 【推荐】表名遵循"业务名称_表的作用"，例如 `trade_config`。
11. 【推荐】库名尽量与应用名称一致。
12. 【推荐】修改字段含义或状态时及时更新字段注释。
13. 【推荐】允许适当冗余提高查询性能，但冗余字段不应频繁修改、不能是唯一索引或超长文本。
14. 【推荐】单表超过 500 万行或 2GB 后再考虑分库分表。
15. 【参考】合理选择字段长度，使用 unsigned 避免误存负数并扩大范围。

| 对象 | 年龄区间 | 类型 | 字节 | 表示范围 |
|---|---|---|---:|---|
| 人 | 150 岁以内 | `tinyint unsigned` | 1 | 0 到 255 |
| 龟 | 数百岁 | `smallint unsigned` | 2 | 0 到 65535 |
| 恐龙化石 | 数千万年 | `int unsigned` | 4 | 0 到约 43 亿 |
| 太阳 | 约 50 亿年 | `bigint unsigned` | 8 | 0 到约 10 的 19 次方 |

### FOM3 衔接

- **主键**：统一 `bigint unsigned` + 雪花 ID，Java 侧 `Long id` + `@TableId(type = IdType.ASSIGN_ID)`（见 layering §3.1/§8）。
- **审计字段**：FOM3 在通用"id/create_time/update_time 三件套"基础上，约定五件套——`id` / `create_time` / `update_time` / `create_user` / `update_user`，均由业务侧维护，Entity 显式 `@TableField`。
- **逻辑删除**：FOM3 不使用常见的 `is_deleted`，固定用 `f_delete_mark`（`@TableLogic`，null=未删除、1=已删除），配套 `f_delete_time`、`f_delete_user_id`，统一 `f_` 前缀。
- **流程/租户字段**：`f_flow_id` / `f_flow_task_id` / `f_flow_node_code` / `f_tenant_id`，仅当表接入审批流程或多租户时才加，不强制。
- **命名**：表名 snake_case 对应 Entity `@TableName`；列名 snake_case，Java 字段驼峰，**每个字段显式 `@TableField(列名)`**，不依赖驼峰自动映射。
- **是/否字段**：除逻辑删除外，其它布尔字段可沿用 `is_xxx` + `unsigned tinyint`；POJO 侧布尔属性不加 `is` 前缀，在 `@TableField`/`resultMap` 完成映射。
- **建表脚本**：放在 `<domain>/sql/`，命名 `YYYYMMDD_描述.sql`（中文描述可），按日期顺序手工执行；已执行脚本不修改（见 AGENTS.md）。

## 二、索引规约

1. 【强制】业务唯一字段必须建立唯一索引，即使是组合字段。
2. 【强制】超过三个表禁止 JOIN；关联字段类型必须一致且有索引。
3. 【强制】`varchar` 索引必须指定索引长度，按区分度决定。
4. 【强制】页面搜索禁止左模糊或全模糊，需要时使用搜索引擎。
5. 【推荐】`ORDER BY` 场景利用索引有序性，组合索引中排序字段放在最后。
6. 【推荐】利用覆盖索引避免回表。
7. 【推荐】超大分页使用延迟关联或子查询优化。
8. 【推荐】SQL 性能目标至少达到 range，最好达到 ref 或 const。
9. 【推荐】组合索引优先放区分度高的字段；等号条件字段优先于范围条件字段。
10. 【推荐】避免字段类型不同导致隐式转换，使索引失效。
11. 【参考】避免"索引宁滥勿缺""吝啬创建索引""抵制唯一索引"等极端误解。

### FOM3 衔接

- **索引命名**：遵循 `pk_`/`uk_`/`idx_` 前缀，与通用规约一致。
- **跨域只读表**：FOM3 允许共用数据源的跨域只读表直查（Repository 类 Javadoc 注明归属与直查原因），关联字段同样要求有索引，且不得反向写入。
- **分页**：FOM3 用 MyBatis-Plus `Page` + Mapper XML；超大分页优先用游标分页（`gt(id, lastId) + ORDER BY id ASC + LIMIT size`），见 layering §5.2/§6.3。

## 三、SQL 语句

1. 【强制】使用 `count(*)` 统计行数，不使用 `count(列名)` 或 `count(常量)` 替代。
2. 【强制】注意 `count(distinct col)` 对 NULL 的处理，以及多列 distinct 的特殊行为。
3. 【强制】`sum` 全为 NULL 时返回 NULL，必要时使用 `IFNULL(SUM(column), 0)`。
4. 【强制】使用 `IS NULL` 或 `IS NOT NULL` 判断 NULL。
5. 【强制】分页查询 count 为 0 时直接返回，不再执行后续分页 SQL。
6. 【强制】禁止使用外键与级联，在应用层解决关联关系。
7. 【强制】禁止使用存储过程。
8. 【强制】数据订正前先 `SELECT` 确认范围，再执行更新或删除。
9. 【强制】多表操作的列名前必须加表别名或表名限定。
10. 【推荐】表别名使用 `as t1`、`as t2`、`as t3` 的顺序命名。
11. 【推荐】避免 `IN`；无法避免时将集合控制在 1000 个以内。
12. 【参考】统一使用 utf8 字符集；需要存储表情时使用 utf8mb4，字符计数区分 `LENGTH` 和 `CHARACTER_LENGTH`。
13. 【参考】`TRUNCATE TABLE` 速度快但无事务且不触发 trigger，不建议在开发代码中使用。

### FOM3 衔接

- **逻辑删除条件**：MyBatis-Plus 对 `@TableLogic` 字段仅在单表 API（`ServiceImpl` 继承方法）自动追加 `f_delete_mark IS NULL`；**自定义 Mapper XML 不会自动追加**，须手写 `WHERE f_delete_mark IS NULL`（见 layering §9）。
- **跨域数据**：跨域取数走 Feign `-api`，本域只操作自家表；共用数据源的跨域只读表例外，Repository 须注释说明，且不写跨域更新 SQL。
- **分页实现**：FOM3 分页用 `Page<Entity>` + Mapper `selectXxxPage(page, query)`，count 为 0 时 MyBatis-Plus 自动跳过 records 查询，契合通用规约。
- **数据订正**：生产数据订正先 `SELECT` 确认范围再操作；结构变更走 `<domain>/sql/` 脚本，不直接改已执行脚本。

## 四、ORM 映射

1. 【强制】查询字段必须明确列出，禁止使用 `*`。
2. 【强制】POJO 布尔属性不加 `is`，数据库字段使用 `is_`，在 resultMap 中完成映射。
3. 【强制】不要使用 `resultClass`，每张表都定义对应的 `<resultMap>`。
4. 【强制】SQL XML 参数使用 `#{}`，禁止使用易引发 SQL 注入的 `${}`。
5. 【强制】不推荐使用 iBATIS `queryForList` 的内存分页方式。
6. 【强制】禁止直接使用 HashMap 或 Hashtable 作为查询结果集输出。
7. 【强制】更新表记录时同步更新 `update_time` 为当前时间。
8. 【推荐】更新接口只更新实际发生变化的字段，不要写大而全的更新 SQL。
9. 【参考】谨慎使用 `@Transactional`，评估数据库、缓存、搜索、消息和统计等回滚方案。
10. 【参考】正确理解 `<isEqual>`、`<isNotEmpty>` 和 `<isNotNull>` 的条件含义。

### FOM3 衔接

- **ORM 框架**：FOM3 用 MyBatis-Plus，不使用 iBATIS/`resultClass`；复杂结果映射用 `<resultMap>`，简单单表用 `BaseMapper`/`ServiceImpl` 自带方法，不重复定义。
- **参数占位符**：统一 `#{}`，禁止 `${}`（防注入）；动态条件用 `<if>`，取值 `#{query.xxx}` 对应 Mapper 方法 `@Param("query")`（见 layering §9）。
- **查询字段**：通用规约强制禁止 `SELECT *`；FOM3 自定义 XML 同样显式列出列名；`BaseMapper` 自带方法（selectById 等）由框架处理，不在此列。
- **布尔映射**：POJO 布尔属性不加 `is`，数据库 `is_xxx`，在 `@TableField`/`resultMap` 完成映射；逻辑删除字段 `f_delete_mark` 用 `Integer`（null/1）而非 `boolean`。
- **更新时间**：Repository 用 `LambdaUpdateWrapper` 更新时显式 `.set(UpdateTime, LocalDateTime.now())`，不依赖数据库默认值（见 layering §5.2 示例）。
- **事务**：`@Transactional` 谨慎使用，事务边界放在 **application 层（Service impl）**，infrastructure 层不加事务；跨服务调用异常 `try-catch` 包成 `BizException`，事务内 catch 异常需手动回滚（呼应异常处理章）。
- **Mapper XML 位置**：`fom-<domain>-biz/src/main/resources/mapper/<Aggregate>Mapper.xml`，`namespace` 指向 `kocel.infrastructure.mapper.<Aggregate>Mapper`。
