---
name: fom-coding-standards
description: FOM3 铸造全流程运营管控平台的前后端编码规范，覆盖 fom-* Spring Cloud 微服务和 ifom3-web Vue3 前端。凡是在 fom-* 中新增或修改 Java、Controller、Service、Mapper、Entity、DTO、VO、Feign、MyBatis XML、SQL，或在 ifom3-web 中新增/修改 Vue、TypeScript、低代码表单/页面配置、高代码页面、BasicForm、BasicTable、API、组件、状态和路由时，务必先加载本 skill，遵循对应的分层、命名、Schema 和高低代码选型约定。
---

# FOM3 编码规范

本规范从 `fom-example` 示例骨架提炼，并融合业界通用 Java 编码规约与 MySQL 数据库规约。规约全文收录在 `references/` 下，skill 自包含，无需查阅外部文档。

规范分多块，**渐进式披露**：本文件只放最高频的核心约定，详细骨架模板与规约全文在 `references/` 下按需查阅。

| 规范块 | 状态 | 何时查阅 |
|--------|------|----------|
| 后端 - 分层架构 | ✅ 已就绪 | 写任何 Java 代码前 |
| 后端 - 编码规约（命名/OOP/异常/日志/并发/前后端/安全/设计） | ✅ 已就绪 | 写 Java 代码、异常处理、日志、接口、枚举、并发时 |
| 数据库（MySQL 建表/索引/SQL/ORM） | ✅ 已就绪 | 建表、写 Mapper XML、Repository 查询、Entity 字段时 |
| 前端（ifom3-web） | ✅ 已就绪 | 写 Vue/TypeScript、API、组件、低代码表单/页面时 |

> 各块深入阅读入口：
> - 后端分层架构完整骨架/命名速查/依赖红线 → `references/backend/layering-architecture.md`
> - 编码规约全文 + FOM3 衔接 → `references/backend/coding-conventions.md`
> - 数据库规约全文 + FOM3 衔接 → `references/database/mysql-conventions.md`
> - 前端共同基线、选型和验证 → 本文件“前端规约”章节
> - 低代码、动态表单脚本、模型契约与高低结合 → `references/frontend/low-code-conventions.md`
> - Vue 高代码、Basic 组件、API、状态、路由与高低结合页面 → `references/frontend/high-code-conventions.md`
>
> 下文是分层架构核心精要，绝大多数日常改动看完这里即可动手；编码规约与数据库规约的核心精要在其后。

## 后端分层架构核心

### 模块依赖链（不可逾越）

每个领域 `fom-<domain>/` 是 5 个 Maven 子模块，依赖方向**只能自上而下**：

```
fom-<domain>-server        启动 + 打包（spring-boot-maven-plugin）
    └── fom-<domain>-controller   REST 控制器 / Feign 服务端 / 定时任务入口
            └── fom-<domain>-biz       业务核心：application + infrastructure + convertor + exception + config
                    └── fom-<domain>-api     跨服务 Feign 接口 + fallback（对外契约）
                            └── fom-<domain>-entity  数据库实体 / DTO / VO / Request / 枚举 / 常量
```

要点：
- **依赖链是纯线性的**：server → controller → biz → api → entity。biz 的 pom 只声明依赖自家 `api`，entity 经 api 传递引入（biz 代码可直接 import entity 类）。
- **biz 是依赖汇聚点**：它依赖自家 `api`，并按需引入各兄弟领域的 `-api` jar（如 `fom-process-api`）。跨服务只依赖 `-api`，**绝不依赖别域的 `-biz`/`-controller`/`-entity`**。
- **controller 只依赖 biz**，不直接依赖 entity/api/server。控制器拿到的是 Service 接口和 VO/DTO。
- **entity 是最底层**，只依赖 `kocel-common-database`（提供 MyBatis-Plus 注解），不反向依赖任何业务层。
- **api 依赖 entity**：对外 Feign 契约里出现的出入参只能是 entity 层的 DTO/VO，这样消费方只引一个 `-api` jar 就够。

### 包命名空间三层

这三层前缀必须分清，混用会导致扫描不到 Bean 或误改框架代码：

| 前缀 | 归属 | 能否修改 |
|------|------|----------|
| `kocel.*` | 本仓库代码 | ✅ 可改 |
| `fom.*` | 跨服务 Feign API（多领域共享） | ✅ 可改，但改动影响下游领域 |
| `jnpf.*` | 基础平台框架，来自外部 jar | ❌ 不要修改 |

启动类的扫描范围因此是三者全包：
```java
@ComponentScan(basePackages = {"kocel", "jnpf", "fom"})
@MapperScan(basePackages = "kocel.infrastructure.mapper")
@EnableFeignClients(basePackages = {"kocel", "jnpf", "fom"})
```

### 各模块的包结构

注意 biz/controller/server 顶层用 `kocel.*`（不带域名），entity/api 用 `kocel.<domain>.*`：

```
fom-<domain>-entity     kocel.<domain>
                          ├── model.entity      数据库实体（@TableName + @TableField）
                          ├── model.dto        跨服务传输对象（纯 POJO）
                          ├── model.vo         返回前端的视图对象（纯 POJO）
                          ├── model.request    请求对象（XxxPageReq / XxxEventReq，带 jakarta.validation）
                          ├── enums            业务枚举（code + message + getInstance 反查）
                          └── constants        常量（如 Feign 服务名）

fom-<domain>-api        kocel.<domain>           @FeignClient 接口（对外契约）
                          └── fallback           降级实现（@Component，返回降级提示）

fom-<domain>-biz        kocel                     ← 注意：顶层 kocel，不带域名
                          ├── application        Service 接口（编排领域逻辑）
                          │   ├── impl           Service 实现（@Slf4j @Service @RequiredArgsConstructor）
                          │   └── extension      从 impl 拆出的独立处理逻辑（@Component，类名 …Handler / …Task）
                          ├── infrastructure     Repository（extends ServiceImpl<Mapper, Entity>）
                          │   └── mapper         Mapper 接口（extends BaseMapper<Entity>）
                          ├── convertor          静态转换器（Entity → VO/DTO，BeanUtils.copyProperties）
                          ├── exception          Assert 断言 + BizException 业务异常
                          ├── config             @RefreshScope 配置类（@Value 读 Nacos）
                          └── utils              静态工具类

fom-<domain>-controller kocel
                          ├── controller         REST 控制器（@RestController + @Tag + @Operation）
                          ├── feign              Feign 服务端实现（implements XxxApi，路径 /feign/<x>）
                          └── job                定时任务入口（@RestController，路径 /job/<x>，外部调度触发）

fom-<domain>-server     kocel                     启动类 Fom<Domain>Application
```

### 各层一句话职责

- **entity**：纯数据结构，无业务逻辑。Entity 贴表、DTO/VO 纯字段、Request 带校验、枚举带反查。
- **api**：跨服务调用的契约，只放 Feign 接口 + 降级，不写业务。
- **biz.application**：领域编排。校验入参 → 状态/规则判定（用枚举 + Assert）→ 调 infrastructure 或跨服务 API 拿数据 → convertor 转 VO/DTO 返回。事务边界在这一层。
- **biz.application.extension**：从 impl 拆出的、可独立成块的处理逻辑（如发起审批、推送外部系统）。当某个 Service 方法逻辑过重时，把可独立、可复用的处理步骤抽到这里，类名以 `Handler`（同步处理一段流程）或 `Task`（执行一个独立任务）结尾，标 `@Component`，由 impl 构造注入并委托调用——目的是让 impl 每个方法读起来仍是"校验 → 判定 → 委托/持久化 → 返回"的主干，细节处理交给 extension，同时便于复用和单独单测。事务边界仍归 application 层，extension 不另起事务、不做跨域持久化。
- **biz.infrastructure**：数据访问。Repository 封装 MyBatis-Plus 查询（LambdaQueryWrapper），Mapper 只放接口签名，复杂 SQL 走 XML。
- **biz.convertor**：Entity ↔ VO/DTO 的静态转换方法，避免在 Service 里手写拷贝。
- **biz.exception**：`Assert.notNull/isTrue/isFalse` 校验失败抛 `BizException`，别用 `throw new RuntimeException`。
- **controller**：只做协议转换。HTTP → 调 Service → `ActionResult` 包装返回。**控制器里不写业务逻辑**。
- **feign**：本服务对外暴露的 Feign 接口服务端实现，委托给 Service。
- **job**：定时任务 HTTP 入口（由外部调度系统触发），批处理用游标分页。
- **server**：启动类 + Nacos 配置引导（bootstrap.yml）+ 本地端口/日志（application.yml）。

### 命名约定速查

| 元素 | 命名 | 示例 |
|------|------|------|
| 启动类 | `Fom<Domain>Application` | `FomExampleApplication` |
| 实体 | `<Aggregate>Entity` | `PlanProductionOrderEntity` |
| DTO/VO | `<Aggregate>DTO` / `<Aggregate>VO` | `PlanProductionOrderDTO` |
| 分页请求 | `<Aggregate>PageReq` | `PlanProductionOrderPageReq` |
| 事件请求 | `<Aggregate>EventReq` | `PlanOrderEndEventReq` |
| 枚举 | `<Subject>StatusEnum` / `<Subject>ActionEnum` | `ProdOrderStatusEnum` |
| Service | `<Aggregate>Service` / `…ServiceImpl` | `PlanProductionOrderService` |
| 扩展处理类 | `<Subject>Handler` / `<Subject>Task` | `CommonFlowHandler` |
| Repository | `<Aggregate>Repository` | `PlanProductionOrderRepository` |
| Mapper | `<Aggregate>Mapper` | `PlanProductionOrderMapper` |
| 转换器 | `<Aggregate>Convertor` | `PlanProductionOrderConvertor` |
| Controller | `<Aggregate>Controller` | `PlanProductionOrderController` |
| Feign 服务端 | `<Aggregate>Client` | `PlanProductionOrderClient` |
| Feign 契约 | `<Aggregate>Api` | `PlanProductOrderApi` |
| 降级 | `<Aggregate>ApiFallback` | `PlanProductOrderApiFallback` |
| 定时任务 | `<Aggregate>Job` | `PlanProductionOrderJob` |
| 配置类 | `<Domain>Config` | `ExampleConfig` |

`<Aggregate>` 通常以主表/主聚合根命名（如 `PlanProductionOrder`），同一聚合下的各类共享这一前缀，形成可检索的家族。

**域前缀规则**：各域 `fom-<domain>` 的业务类，其 `<Aggregate>`/`<Subject>` 必须以域前缀开头——`fom-base` 用 `Base`（如 `BaseCustomerService`、`BaseCustomerEntity`、`BaseCustomerApi`），`fom-plan` 用 `Plan`（如 `PlanProcessReportService`、`PlanOrderEndEventReq`）。范围：biz 根（Service/ServiceImpl/Convertor/Repository/Mapper）、controller（Controller/Client）、entity 的 model 层（DTO/Entity/PageReq/VO/EventReq）。跨服务引用时类名一眼可识别来源域，避免不同域同名类混淆；新增聚合类时先确定域前缀再命名，存量类缺前缀的按此规则补齐。例外（不加前缀）：`application/extension/` 下的扩展处理类（Handler/Strategy）、`enums` 下的枚举、`Assert`/`BizException`（exception）、`Toolkit`（utils）、`FeignName`（constants）、启动类 `Fom<Domain>Application`、配置类 `<Domain>Config`。

## 编码规约核心

以下是与 FOM3 日常编码最高频的条目（通用规约 + FOM3 衔接），完整版见 `references/backend/coding-conventions.md`。

- **命名**：类 `UpperCamelCase`（`DO`/`DTO`/`VO`/`BO` 等后缀除外），方法/变量 `lowerCamelCase`，常量全大写下划线；POJO 布尔不加 `is`；包名小写单数。FOM3 命名速查见上节"命名约定速查"。
- **POJO**：属性用包装类型，不设默认值，实现 `toString`；DO 属性类型匹配 DB 字段（`bigint→Long`）；属性 copy 用 Spring `BeanUtils`（禁 Apache BeanUtils），统一放 `convertor/`。
- **异常**：业务异常用 `BizException` + `Assert.notNull/isTrue/isFalse`，**禁 `throw new RuntimeException`**；跨服务 Feign 调用 `try-catch` 包成 `BizException(msg, e)` 保留原因链；事务边界在 application 层，事务内 catch 需手动回滚。
- **日志**：SLF4J 占位符 `log.info("x: {}", id)`，异常带堆栈，生产禁 `System.out`/`e.printStackTrace()`；`@Slf4j` 关键业务打 info（含业务主键），调用失败打 error。
- **注释**：Service 接口每个方法、`application/extension/` 下类的类头和公开方法、`controller/job/` 下定时任务入口的每个端点方法必须加中文 Javadoc——统一多行格式（`/**` 单起一行，描述、`@param`、`@return` 各占一行，`*/` 收尾）；Service 方法描述与 Controller `@Operation(summary)` 一致，extension 类头说明职责、公开方法用 `@param` 标注关键参数，Job 端点说明接口用途（做什么、由外部调度触发）并补 `@return`。Service impl 方法在关键位置（状态/规则判定、跨服务调用、委托 extension、分支边界、持久化等）加行注释 `//`，写在被注释语句上方，解释"为什么这样做"或"这一步在做什么"，不注释显而易见的代码。
- **集合**：判空用 `isEmpty()`；`Collectors.toMap` 提供重复 key 合并函数并防 value 为 null 的 NPE；foreach 中删除用 `Iterator.remove()`。
- **并发**：线程池用 `ThreadPoolExecutor`（禁 `Executors`）；`ThreadLocal` 用 `try-finally` 回收；定时任务走 `job/` HTTP 入口，批处理游标分页。
- **前后端**：Controller 只做协议转换，返回 `ActionResult`（分页 `ActionResult.page`，普通 `ActionResult.success`）；JSON key `lowerCamelCase`；雪花 ID/超大整数可直接返回 `Long`，框架自动序列化防精度丢失，VO 无需改用 `String`。
- **安全**：MyBatis XML 统一 `#{}` 禁 `${}`；跨域数据走 Feign `-api`，不直查别域表（共用数据源只读例外须注释）。
- **设计**：分层架构即设计落地，新增领域/聚合先对照依赖红线；业务流程用 LiteFlow 编排，状态流转用枚举建模。

## 数据库规约核心

以下是与 FOM3 建表/SQL/ORM 最高频的条目（通用规约 + FOM3 衔接），完整版见 `references/database/mysql-conventions.md`。

- **建表**：`id` 为 `bigint unsigned` 主键（FOM3 用雪花 ID `IdType.ASSIGN_ID`）；必备 `id`/`create_time`/`update_time`（FOM3 再加 `create_user`/`update_user`）；小数用 `decimal` 禁 `float`/`double`；`varchar`≤5000 超出用 `text` 独立成表；表名小写非复数禁保留字。
- **FOM3 字段约定**：逻辑删除固定 `f_delete_mark`（`@TableLogic`，null=未删除/1=已删除）+ `f_delete_time`/`f_delete_user_id`，**不用** `is_deleted`；流程/租户字段 `f_` 前缀按需添加；列名 snake_case，Java 字段驼峰，**每个字段显式 `@TableField(列名)`** 不依赖自动映射。
- **索引**：`pk_`/`uk_`/`idx_` 前缀；超 3 表禁 JOIN，关联字段类型一致且有索引；`varchar` 索引指定长度；禁左模糊/全模糊。
- **SQL**：统计用 `count(*)`；禁外键级联与存储过程；数据订正先 `SELECT` 确认范围；逻辑删除表的自定义 XML 须手写 `WHERE f_delete_mark IS NULL`（`@TableLogic` 仅对单表 API 自动追加，自定义 XML 不自动）；分页用 MyBatis-Plus `Page`，超大分页用游标 `gt(id,lastId)+LIMIT`。
- **ORM**：禁 `SELECT *`；参数 `#{}` 禁 `${}`；更新同步 `update_time`（`LambdaUpdateWrapper` 显式 `.set(updateTime, now)`）；`@Transactional` 谨慎，事务放 application 层；Mapper XML 在 `fom-<domain>-biz/src/main/resources/mapper/<Aggregate>Mapper.xml`，`namespace` 指向 `kocel.infrastructure.mapper.<Aggregate>Mapper`。

## 前端规约

适用于 `ifom3-web` 的改动。前端规约按“共同基线 + 实现方式专题”组织，先判断边界，再阅读对应专题，避免把低代码运行时脚本和高代码 Vue 页面混用。

### 阅读路径

1. **判断实现方式**：普通 CRUD、筛选、详情、审批、门户、报表和布局优先使用低代码/设计器；复杂交互、特殊布局、非标准性能或平台无法表达时才新增高代码 Vue。
2. **阅读共同基线**：项目使用 Vue 3、TypeScript、Vite、Vue Router 4、Pinia、VueUse、Ant Design Vue 4、Less/Windi CSS；业务页面优先复用 Ant Design Vue 和自研 `Basic*` 组件，不引入 Element Plus、DaisyUI 等第二套基础 UI。
3. **阅读实现专题**：
   - 低代码、动态表单脚本、列表设计、字段联动、虚拟列表、模型契约 → `references/frontend/low-code-conventions.md`
   - Vue 页面、组件、Basic 组件、API、状态、路由、样式和测试 → `references/frontend/high-code-conventions.md`
   - `CustomModel`、`customClick`、父子弹窗刷新等高低结合场景 → 两份都读，以低代码运行时契约和高代码边界为准。
4. **先读现有实现**：修改前检查目标目录的组件导出、API 定义、类型和运行时实现，不按其他项目模板臆造接口。设计器预览通过或页面能打开，都不能替代真实接口、权限、异常和刷新链路验证。

### 共同基线

- 新 Vue 单文件组件使用 `<script setup lang="ts">`，区块顺序为 `template`、`script`、`style`。
- 纯函数用 `function` 声明；优先 `interface`，边界动态数据避免 `any`。
- 表单和列表优先使用 Schema/columns 描述，异步配置使用 `updateSchema`，自定义渲染优先具名 slot。
- 页面不直接调用 axios；API 集中在 `src/api/<domain>/`，统一使用项目 `defHttp` 封装。
- 处理 loading、空数据、异常、重复提交；大整数 ID 由后端框架自动序列化防精度丢失，前端无需自行字符串化；token 不写死、不输出日志。
- 前端隐藏不能替代后端鉴权；动态脚本禁止 `eval`、不可信 HTML、动态脚本注入和未经审核的第三方接口。

### 交付验证

前端命令统一在前端工程目录、且已 `pnpm bootstrap` 安装依赖后执行。验证以**本次改动范围**为准即可，全仓库扫描不作为卡点（仓库存在大量存量类型/lint 错误，全量跑只会淹没新增问题）：

1. 类型检查：`npx vue-tsc --noEmit --skipLibCheck`，把输出过滤到本次改动文件所在目录（如 `src/views/<module>`），改动文件零错误即通过；存量模块的既有错误不属于本次交付范围。
2. Lint：只对本次改动文件执行 `npx eslint <文件>`、`npx stylelint <文件>`（命中 ignore 清单的文件跳过）。
3. 行为逻辑改动在被测代码附近补充 `*.spec.ts`，用 `npx vitest run <spec>` 只跑相关用例。
4. 仅当改动涉及全局配置、依赖或构建链路时，才执行 `pnpm build`（联邦模块场景用 `pnpm build:federation`）。
5. 低代码改动验证真实发布模型、权限、接口数据、空值、旧配置和移动端路径；高代码改动验证桌面/窄屏、亮色/暗色、加载/空数据/错误、重复提交和弹窗关闭刷新。

## 深入阅读

- 写新模块、拿不准包放哪、要看完整代码骨架时，读 `references/backend/layering-architecture.md`——里面有每一层的可复制模板、字段约定（审计字段/逻辑删除/雪花主键）、Mapper XML 规范、依赖红线清单。
- 写 Java 代码/异常/日志/接口/枚举/并发，或需要编码规约全文时，读 `references/backend/coding-conventions.md`（含编码规约全文 + FOM3 衔接）。
- 建表/写 Mapper XML/Repository 查询/Entity 字段，或需要 MySQL 规约全文时，读 `references/database/mysql-conventions.md`（含 MySQL 数据库规约全文 + FOM3 衔接）。
- 修改 `ifom3-web` 时，先阅读本文件的“前端规约”选型路径，再按实现方式阅读 `references/frontend/low-code-conventions.md` 或 `references/frontend/high-code-conventions.md`；高低代码结合场景两份都读。
