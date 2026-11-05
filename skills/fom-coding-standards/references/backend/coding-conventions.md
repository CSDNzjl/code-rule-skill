# 编码规约 —— FOM3 融合版

> 本文件融合业界通用 Java 编码规约（编程规约/异常日志/单元测试/安全规约/设计规约，已去除"工程结构"与"MySQL 数据库"两栏）与 FOM3 项目约定，**规约全文收录于此**，无需查阅外部文档。
>
> - 规约用【强制】/【推荐】/【参考】标注，完整保留每条原文。
> - FOM3 项目特定约定放在每章末尾"FOM3 衔接"小节，与 `references/backend/layering-architecture.md` 的分层/包结构/命名对照阅读。
> - 建表/SQL/ORM 见 `references/database/mysql-conventions.md`。
>
> 写任何 Java 代码、异常处理、日志、接口、枚举、并发代码前，先看本文件。

## 目录

- [一、命名风格](#一命名风格)
- [二、常量定义](#二常量定义)
- [三、代码格式](#三代码格式)
- [四、OOP 规约](#四oop-规约)
- [五、日期时间](#五日期时间)
- [六、集合处理](#六集合处理)
- [七、并发处理](#七并发处理)
- [八、控制语句](#八控制语句)
- [九、注释规约](#九注释规约)
- [十、前后端规约](#十前后端规约)
- [十一、其他](#十一其他)
- [十二、异常处理](#十二异常处理)
- [十三、日志规约](#十三日志规约)
- [十四、单元测试](#十四单元测试)
- [十五、安全规约](#十五安全规约)
- [十六、设计规约](#十六设计规约)

---

## 一、命名风格

1. 【强制】代码中的命名不能以下划线或美元符号开始，也不能以下划线或美元符号结束。
2. 【强制】编程相关命名严禁使用拼音与英文混合的方式，也不允许直接使用中文。
3. 【强制】代码和注释中避免使用任何语言的种族歧视性词语。
4. 【强制】类名使用 `UpperCamelCase` 风格，`DO`、`BO`、`DTO`、`VO`、`AO`、`PO`、`UID` 等除外。
5. 【强制】方法名、参数名、成员变量和局部变量统一使用 `lowerCamelCase` 风格。
6. 【强制】常量命名全部大写，单词间用下划线隔开，并完整表达语义。
7. 【强制】抽象类以 `Abstract` 或 `Base` 开头；异常类以 `Exception` 结尾；测试类以被测试类名开头、以 `Test` 结尾。
8. 【强制】数组类型与中括号紧挨表示，例如 `int[] arrayDemo`，不要写成 `int arrayDemo[]`。
9. 【强制】POJO 中的布尔类型变量不要加 `is` 前缀，避免部分框架序列化解析错误。
10. 【强制】包名统一使用小写，点分隔符之间只有一个自然语义的英文单词；包名使用单数形式。
11. 【强制】避免父子类成员变量、同一方法不同代码块的局部变量使用完全相同的名称。
12. 【强制】杜绝完全不规范的缩写，避免出现无法望文知义的名称。
13. 【推荐】自定义编程元素命名时使用尽量完整的单词组合，实现代码自解释。
14. 【推荐】表示类型的名词放在变量名词尾，例如 `startTime`、`workQueue`、`nameList`。
15. 【推荐】使用设计模式时，在命名中体现具体模式，例如 `OrderFactory`、`LoginProxy`。
16. 【推荐】接口中的方法和属性不要增加 `public` 等修饰符，并添加有效的 Javadoc；尽量不要在接口中定义变量。
17. 接口和实现类命名：Service、DAO 接口的实现类使用 `Impl` 后缀；能力型接口优先使用形容词或 `-able` 形式。
18. 【参考】枚举类名可带 `Enum` 后缀，枚举成员全部大写并用下划线分隔。
19. 【参考】Service/DAO 方法命名：单个对象使用 `get`，多个对象使用 `list`，统计使用 `count`，新增使用 `save/insert`，删除使用 `remove/delete`，修改使用 `update`。数据对象使用 `DO`，数据传输对象使用 `DTO`，展示对象使用 `VO`，POJO 是这些对象的统称。

### FOM3 衔接

- FOM3 命名速查表见 layering §"命名约定速查"：`<Aggregate>Entity`/`<Aggregate>DTO`/`<Aggregate>VO`/`<Aggregate>PageReq`/`<Aggregate>EventReq`/`<Subject>StatusEnum`/`<Aggregate>Service`/`<Aggregate>Repository`/`<Aggregate>Mapper`/`<Aggregate>Convertor`/`<Aggregate>Controller`/`<Aggregate>Client`/`<Aggregate>Api`/`<Aggregate>ApiFallback`/`<Aggregate>Job`/`<Domain>Config`。
- `<Aggregate>` 以主表/聚合根命名（如 `PlanProductionOrder`），同聚合各类共享前缀，形成可检索家族。
- 包命名空间三层（layering §2）：`kocel.*`（本仓库，可改）/ `fom.*`（跨服务 Feign API，改动影响下游）/ `jnpf.*`（外部 jar，不可改）。
- 业务枚举带 `code`+`message`+`getInstance(code)` 反查（layering §3.4）；Feign 服务名等常量用接口承载（layering §3.5）。

## 二、常量定义

1. 【强制】禁止魔法值直接出现在代码中，应预先定义有语义的常量。
2. 【强制】`long` 或 `Long` 赋值时使用大写 `L`，例如 `2L`。
3. 【推荐】按常量功能拆分常量类，不要用一个大而全的常量类维护所有常量。
4. 【推荐】常量复用层次分为跨应用、应用内、子工程内、包内、类内五层，按使用范围放置。
5. 【推荐】变量值仅在固定范围内变化，且存在延伸属性时，使用 `enum` 定义。

```java
public enum SeasonEnum {
    SPRING(1), SUMMER(2), AUTUMN(3), WINTER(4);

    private int seq;

    SeasonEnum(int seq) {
        this.seq = seq;
    }

    public int getSeq() {
        return seq;
    }
}
```

### FOM3 衔接

- 跨模块共享常量（如 Feign 服务名）放 entity 层 `constants` 子包，用接口承载（layering §3.5）。
- 业务状态/动作用枚举（`enums` 子包），带 `code`+`message`+反查，避免散落魔法值。

## 三、代码格式

1. 【强制】空代码块写成 `{}`；非空代码块左大括号不换行，右大括号前换行。
2. 【强制】小括号内侧不加空格，左大括号前加空格。
3. 【强制】`if`、`for`、`while`、`switch`、`do` 等关键字与括号之间加空格。
4. 【强制】二目、三目运算符两侧各加一个空格。
5. 【强制】使用 4 个空格缩进，禁止使用 Tab 字符。
6. 【强制】双斜线注释与内容之间有且仅有一个空格。
7. 【强制】强制类型转换的右括号与被转换值之间不加空格，例如 `(int)first`。
8. 【强制】单行字符数不超过 120 个；换行时第二行缩进 4 个空格，运算符和方法调用点与下文一起换行，参数在逗号后换行。
9. 【强制】方法定义和调用中多个参数的逗号后必须有空格。
10. 【强制】IDE 文件编码使用 UTF-8，换行符使用 Unix 格式。
11. 【推荐】单个方法总行数不超过 80 行。
12. 【推荐】不要使用额外空格对齐多行赋值等号。
13. 【推荐】不同逻辑、语义或业务代码之间插入一个空行。

### FOM3 衔接

- 与项目现有写法保持一致；Controller/Service impl 类注解顺序见 layering §5.1/§6.1（`@Slf4j @Service @RequiredArgsConstructor` / `@Slf4j @RestController @RequestMapping @Tag`）。

## 四、OOP 规约

1. 【强制】访问静态变量或方法直接使用类名，不要通过对象引用访问。
2. 【强制】所有覆写方法必须添加 `@Override`。
3. 【强制】只有参数类型和业务含义相同，才使用 Java 可变参数；可变参数必须放在最后。
4. 【强制】外部调用或二方库依赖的接口不得修改方法签名；过时接口添加 `@Deprecated` 并说明替代方案。
5. 【强制】不得使用过时的类或方法。
6. 【强制】使用常量或确定非空对象调用 `equals`，例如 `"test".equals(object)`；也可使用 `Objects.equals`。
7. 【强制】整型包装类对象之间的值比较使用 `equals`。
8. 【强制】货币金额使用最小货币单位，以整型类型存储。
9. 【强制】浮点数不要直接使用 `==` 或包装类型的 `equals` 判断相等，应使用误差范围或 `BigDecimal`。
10. 【强制】`BigDecimal` 等值比较使用 `compareTo()`，不要使用 `equals()`。
11. 【强制】DO 属性类型必须与数据库字段类型匹配，例如 `bigint` 对应 `Long`。
12. 【强制】禁止使用 `new BigDecimal(double)`，优先使用字符串构造方法或 `BigDecimal.valueOf`。
13. 【强制】POJO 属性和 RPC 参数、返回值使用包装类型；局部变量推荐使用基本类型。
14. 【强制】DO、DTO、VO 等 POJO 不设定属性默认值。
15. 【强制】序列化类新增属性不要修改 `serialVersionUID`；完全不兼容升级时才修改。
16. 【强制】构造方法中禁止加入业务逻辑，初始化逻辑放在 `init` 方法中。
17. 【强制】POJO 必须实现 `toString()`；继承 POJO 时注意包含 `super.toString()`。
18. 【强制】同一 POJO 属性不能同时存在 `isXxx()` 和 `getXxx()`。
19. 【推荐】使用 `String.split` 得到数组并按索引访问时，检查末尾分隔符后的内容。
20. 【推荐】多个构造方法或同名方法应连续放置。
21. 【推荐】类内方法顺序为公有/保护方法、私有方法、getter/setter 方法。
22. 【推荐】setter 参数与成员变量同名并使用 `this` 赋值；getter/setter 不增加业务逻辑。
23. 【推荐】循环内字符串连接使用 `StringBuilder.append`。
24. 【推荐】合理使用 `final` 修饰类、成员变量、方法和局部变量。
25. 【推荐】慎用 `clone`，默认是浅拷贝，深拷贝需自行实现。
26. 【推荐】类成员与方法访问控制从严，能使用 `private` 就不要使用更宽泛的访问级别。

### FOM3 衔接

- Entity 用 `@Data` + `@TableName` + 显式 `@TableField`；DTO/VO 纯 `@Data` 无 MyBatis 注解；Request 用 `jakarta.validation` 校验（中文消息）（layering §3）。
- POJO 属性用包装类型（`Long`/`Integer`/`String`），不设默认值，与通用规约一致。
- 跨服务 Feign 出入参只用 entity 层 DTO/VO，不用枚举或含枚举的 POJO（呼应二方库规约）。
- 对象拷贝用 Spring `BeanUtils.copyProperties`（禁 Apache BeanUtils），统一放 `convertor/` 静态方法（layering §5.4）。

## 五、日期时间

1. 【强制】日期格式年份统一使用小写 `y`，例如 `yyyy-MM-dd HH:mm:ss`，不要使用 `YYYY`。
2. 【强制】区分 `M/m`：`M` 表示月份、`m` 表示分钟；区分 `H/h`：`H` 表示 24 小时制、`h` 表示 12 小时制。
3. 【强制】获取当前毫秒数使用 `System.currentTimeMillis()`；纳秒级计时使用 `System.nanoTime()`。
4. 【强制】不使用 `java.sql.Date`、`java.sql.Time`、`java.sql.Timestamp`。
5. 【强制】不要硬编码一年为 365 天，使用日期 API 处理闰年。
6. 【推荐】避免公历闰年 2 月 29 日的一年后日期问题。
7. 【推荐】使用枚举值指代月份；使用数字时注意 `Date`、`Calendar` 的月份范围为 0 到 11。

```java
int daysOfThisYear = LocalDate.now().lengthOfYear();
int daysOfSpecifiedYear = LocalDate.of(2011, 1, 1).lengthOfYear();
```

### FOM3 衔接

- Entity 审计字段统一 `LocalDateTime`（layering §3.1/§8）；更新时间用 `LocalDateTime.now()` 显式 set，不依赖数据库默认值。
- 前后端时间格式统一 `yyyy-MM-dd HH:mm:ss`。

## 六、集合处理

1. 【强制】覆写 `equals` 必须同时覆写 `hashCode`；作为 `Set` 元素或 `Map` 键的自定义对象必须覆写二者。
2. 【强制】判断集合为空使用 `isEmpty()`，不要使用 `size() == 0`。
3. 【强制】`Collectors.toMap()` 必须提供重复 key 的合并函数。
4. 【强制】`Collectors.toMap()` 需要注意 value 为 `null` 会抛出 NPE。
5. 【强制】`ArrayList.subList()` 返回视图，不可强转成 `ArrayList`。
6. 【强制】`Map.keySet()`、`values()`、`entrySet()` 返回的集合不能添加元素。
7. 【强制】`Collections.emptyList()`、`singletonList()` 等返回不可变集合，不能增删元素。
8. 【强制】父集合在 `subList` 创建后增加或删除元素，可能导致子列表操作抛出 `ConcurrentModificationException`。
9. 【强制】集合转数组使用类型一致、长度为 0 的数组：`list.toArray(new String[0])`。
10. 【强制】调用 `Collection.addAll()` 前检查输入集合是否为 `null`。
11. 【强制】`Arrays.asList()` 返回的集合不能调用 `add/remove/clear`；其元素与原数组共享。
12. 【强制】遵循 PECS 原则：生产者使用 `<? extends T>`，消费者使用 `<? super T>`。
13. 【强制】无泛型集合赋值给泛型集合时，使用元素前进行 `instanceof` 检查。
14. 【强制】foreach 中不要进行 `remove/add`；删除使用 `Iterator.remove()`，并发时加锁。
15. 【强制】Comparator 必须满足反对称、传递和等价一致性，否则排序可能抛异常。
16. 【推荐】JDK7+ 使用 diamond 语法或省略泛型。
17. 【推荐】集合初始化时指定合理的初始容量，无法确定时可使用 16。
18. 【推荐】遍历 Map 的 key/value 使用 `entrySet` 或 JDK8 的 `forEach`。
19. 【推荐】注意不同 Map 对 `null` key/value 的支持情况。

| 集合类 | Key | Value | 线程安全 | 说明 |
|---|---|---|---|---|
| `Hashtable` | 不允许 `null` | 不允许 `null` | 是 | `Dictionary` |
| `ConcurrentHashMap` | 不允许 `null` | 不允许 `null` | 是 | 锁分段技术，JDK8 使用 CAS |
| `TreeMap` | 不允许 `null` | 允许 `null` | 否 | `AbstractMap` |
| `HashMap` | 允许 `null` | 允许 `null` | 否 | `AbstractMap` |

20. 【参考】合理利用集合的有序性、稳定性，避免无序和不稳定带来的影响。
21. 【参考】利用 `Set` 元素唯一的特性进行去重，避免使用 `List.contains()` 遍历去重。

### FOM3 衔接

- Convertor 列表转换返回 `Collections.emptyList()` 防 NPE（layering §5.4 示例）。
- Repository 查询用 `LambdaQueryWrapper`/`LambdaUpdateWrapper`，返回集合判空用 `isEmpty()`。

## 七、并发处理

1. 【强制】单例对象及其方法必须保证线程安全。
2. 【强制】创建线程或线程池时指定有意义的线程名称。
3. 【强制】线程资源必须通过线程池提供，不得显式创建线程。
4. 【强制】线程池不得使用 `Executors` 创建，应使用 `ThreadPoolExecutor` 明确线程池规则。
5. 【强制】`SimpleDateFormat` 非线程安全；使用锁、`ThreadLocal` 或 JDK8 的日期时间 API。
6. 【强制】自定义 `ThreadLocal` 必须回收，在线程池场景中使用 `try-finally` 调用 `remove()`。
7. 【强制】高并发同步调用应评估锁损耗，能无锁则无锁，能锁代码块则不锁整个方法，能对象锁则不使用类锁。
8. 【强制】多个资源同时加锁时保持一致的加锁顺序，避免死锁。
9. 【强制】阻塞等待获取锁时，先 `lock()`，再进入 `try`，并在 `finally` 中 `unlock()`。
10. 【强制】使用 `tryLock()` 时先判断是否持有锁，再进入业务代码并释放锁。
11. 【强制】并发修改同一记录时加锁或使用乐观锁；乐观锁以 `version` 为依据，重试次数不得少于 3 次。
12. 【强制】多线程定时任务不要使用可能因一个任务异常而终止全部任务的 `Timer`，应使用 `ScheduledExecutorService`。
13. 【推荐】资金相关金融敏感信息使用悲观锁策略。
14. 【推荐】使用 `CountDownLatch` 时确保每个线程退出前调用 `countDown()`，并捕获子线程异常。
15. 【推荐】避免多个线程共享 `Random`，JDK7+ 使用 `ThreadLocalRandom`。
16. 【推荐】双重检查锁中的目标属性声明为 `volatile`。
17. 【参考】`volatile` 解决可见性，但不能解决多写场景的复合操作安全问题；计数可使用 `AtomicInteger` 或 `LongAdder`。
18. 【参考】高并发下避免 `HashMap` 扩容导致风险。
19. 【参考】`ThreadLocal` 通常使用 `static`，但它不能解决共享对象的更新问题。

### FOM3 衔接

- Spring 单例 Service 默认无状态，避免在字段放可变共享状态；定时任务走 `job/` HTTP 入口，由外部调度触发（layering §6.3），批处理用游标分页。

## 八、控制语句

1. 【强制】`switch` 的每个 `case` 必须通过 `continue/break/return` 终止，或注释说明继续执行路径；必须有放在最后的 `default`。
2. 【强制】外部参数作为 `String switch` 表达式时，先进行 `null` 判断。
3. 【强制】`if/else/for/while/do` 必须使用大括号。
4. 【强制】三目运算符注意类型对齐引发的自动拆箱 NPE。
5. 【强制】高并发场景避免使用"等于"作为中断条件，使用大于或小于的区间判断。
6. 【推荐】方法超过 10 行时，`return/throw` 等中断逻辑的右大括号后增加空行。
7. 【推荐】异常分支少用 `if-else`，优先使用卫语句、策略模式或状态模式，嵌套不超过 3 层。
8. 【推荐】复杂条件判断先赋值给有意义的布尔变量。
9. 【推荐】不要在其它表达式中插入赋值语句。
10. 【推荐】循环体中对象创建、变量定义、数据库连接和不必要的 `try-catch` 尽量移至循环外。
11. 【推荐】避免取反逻辑，优先写正向条件。
12. 【推荐】公开接口必须进行入参保护，尤其是批量操作接口。
13. 【参考】低频、高耗时、高稳定性要求、开放接口和敏感权限入口应进行参数校验。
14. 【参考】高频循环调用、底层高频调用以及已由调用方校验的 private 方法，可视情况省略重复校验。

### FOM3 衔接

- Service 业务编排四步（layering §5.1）：**校验入参（Assert）→ 状态/规则判定（枚举 + switch）→ 调 infrastructure/跨服务 API → convertor 转 VO/DTO**。用卫语句 + `Assert` 早返回，避免深嵌套。

## 九、注释规约

1. 【强制】类、类属性和类方法使用 Javadoc，不使用普通 `//` 代替。
2. 【强制】所有抽象方法和接口方法必须使用 Javadoc，说明功能、参数、返回值、异常和调用注意事项。
3. 【强制】所有类添加创建者和创建日期。
4. 【强制】方法内部单行注释写在被注释语句上方，多行注释使用 `/* */` 并与代码对齐。
5. 【强制】枚举字段必须注释说明用途。
6. 【推荐】优先用中文准确说明问题，专有名词和关键字保留英文。
7. 【推荐】代码修改时同步修改注释。
8. 【推荐】删除未使用字段、方法、内部类、参数和局部变量。
9. 【参考】谨慎注释掉代码；无用代码直接删除，暂时保留时说明原因。
10. 【参考】注释应准确反映设计思想、代码逻辑和业务含义。
11. 【参考】好的命名和代码结构可以自解释，注释应精简准确。
12. 【参考】`TODO`、`FIXME` 等特殊标记注明标记人和时间，并及时清理。

### FOM3 衔接

- Entity 每个 `@TableField` 上写中文 Javadoc（layering §3.1）；跨域只读表/Repository 类 Javadoc 注明归属与直查原因。
- Controller 用 Knife4j `@Tag(name=..., description="<domain>")`/`@Operation(summary=...)`（layering §6.1），按领域分组。
- `application/` 下 Service 接口的每个方法必须加中文 Javadoc，统一多行格式（`/**` 单起一行，描述、`@param`、`@return` 各占一行，`*/` 收尾），一句话说明"做什么"，描述与对应 Controller 的 `@Operation(summary)` 保持一致（如"生产订单分页查询"、"生产订单审核-发起事件"）；无对应 Controller 端点的方法（如仅供 Feign 服务端委托调用的查询）按同样风格补写。
- Service impl 方法在关键位置加行注释 `//`（写在被注释语句上方）：状态/规则判定、跨服务调用、委托 extension、分支边界（如早返回）、持久化等，解释"为什么这样做"或"这一步在做什么"，不注释显而易见的代码（layering §5.1）。
- `application/extension/` 下扩展处理类（Handler/Task）的类头必须加多行 Javadoc 说明职责（如"通用审批流程处理类"），每个公开方法加多行 Javadoc 说明"做什么"并用 `@param` 标注关键参数（layering §5.1.1）。
- `controller/job/` 下定时任务入口（Job）的每个端点方法必须加多行 Javadoc 说明接口用途（做什么、由外部调度系统 HTTP 触发），并补 `@return`（layering §6.3）。

## 十、前后端规约

1. 【强制】API 明确协议、域名、路径、请求方法、请求内容、状态码和响应体。生产环境使用 HTTPS；路径使用小写名词和下划线；请求体设置 `Content-Type`。
2. 【强制】列表接口无数据时返回空数组 `[]` 或空集合 `{}`。
3. 【强制】服务端错误响应包含 HTTP 状态码、`errorCode`、`errorMessage` 和用户提示信息。
4. 【强制】JSON key 使用以小写字母开始的 `lowerCamelCase`，例如 `errorCode`、`menuList`。
5. 【强制】`errorMessage` 用于前后端错误追踪，不应包含敏感信息。
6. 【推荐】雪花 ID/超大整数可直接返回 `Long`：本项目框架（Jackson 全局配置）已自动序列化防精度丢失，无需 VO 改用 `String`。
7. 【强制】URL 传参不超过 2048 字节。
8. 【强制】body 传参必须控制长度，超过服务器限制时应调整服务端配置并评估风险。
9. 【强制】分页参数小于 1 时返回第一页；超过总页数时返回最后一页。
10. 【强制】服务器内部重定向使用 `forward`；外部重定向地址使用统一 URL 代理模块生成。
11. 【推荐】服务端响应明确是否可缓存。
12. 【推荐】服务端优先使用 JSON 而不是 XML。
13. 【推荐】前后端时间格式统一为 `yyyy-MM-dd HH:mm:ss`，统一使用 GMT。
14. 【参考】接口路径不加入版本号，版本控制在 HTTP 头信息中体现。

### FOM3 衔接

- Controller 返回统一用 `jnpf.base.ActionResult` 包装：分页 `ActionResult.page(list, pagination)`，普通 `ActionResult.success(data)`，无返回 `ActionResult.success()`（layering §6.1）。
- `@Tag(description="<domain>")` 标注所属领域，Knife4j 文档按此分组；接口文档 `http://localhost:<port>/doc.html`。
- 雪花 ID/超大整数直接返回 `Long` 即可，框架自动处理前端精度丢失，VO 无需用 `String`。

## 十一、其他

1. 【强制】正则表达式使用预编译功能，不要在方法体内重复 `Pattern.compile()`。
2. 【强制】避免使用 Apache BeanUtils 进行属性 copy，可使用 Spring BeanUtils 或 Cglib BeanCopier，注意均为浅拷贝。
3. 【强制】Velocity 调用 POJO 属性时直接使用属性名，由模板引擎按规范调用 getter。
4. 【强制】后台输送给页面的变量使用 `$!{var}`，避免 null 直接显示。
5. 【强制】`Math.random()` 返回 `0 <= x < 1` 的 double；获取整数随机数使用 `Random.nextInt/nextLong`。
6. 【推荐】视图模板中不要加入复杂业务逻辑。
7. 【推荐】数据结构初始化时指定大小，避免无限增长耗尽内存。
8. 【推荐】及时清理不再使用的代码段和配置信息。

### FOM3 衔接

- 属性 copy 统一走 `convertor/`（layering §5.4），不在 Service 散写。

## 十二、异常处理

1. 【强制】可通过预检查规避的 RuntimeException 不应通过 catch 处理，例如 NPE、IndexOutOfBoundsException。
2. 【强制】异常捕获不能用于流程控制或条件控制。
3. 【强制】区分稳定代码和非稳定代码，尽量按异常类型分别捕获处理。
4. 【强制】捕获异常必须处理；不处理就继续抛给调用者，最外层必须转化为用户可理解的内容。
5. 【强制】事务中 catch 异常后需要回滚时，必须手动回滚事务。
6. 【强制】`finally` 关闭资源和流，并处理关闭异常；JDK7+ 推荐 try-with-resources。
7. 【强制】禁止在 `finally` 中使用 `return`。
8. 【强制】捕获异常类型必须完全匹配抛出异常，或捕获其父类。
9. 【强制】调用 RPC、二方包或动态生成类的相关方法时，捕捉异常使用 `Throwable` 拦截。
10. 【推荐】返回值允许为 null 时，必须注释说明返回 null 的条件。
11. 【推荐】防止自动拆箱、数据库查询、集合元素、远程返回对象、Session 数据及级联调用引起的 NPE。
12. 【推荐】区分 unchecked/checked 异常，使用有业务含义的自定义异常，不直接抛出 `Exception` 或 `Throwable`。
13. 【参考】对外 HTTP/API 使用 errorCode；应用内部优先抛异常；跨应用 RPC 优先使用包含 `isSuccess()`、`errorCode`、`errorMessage` 的 Result。

### FOM3 衔接

- FOM3 业务异常统一用 `BizException`，校验用 `Assert.notNull/isTrue/isFalse/notEmpty`（layering §5.5），**禁 `throw new RuntimeException`**。
- 跨服务 Feign 调用 `try-catch` 包成 `BizException(msg, e)` 保留原因链；事务边界在 application 层，事务内异常需手动回滚（呼应通用规约）。
- 对外 HTTP/API 用 `ActionResult`（含错误码/消息），契合"对外用 errorCode"原则。

## 十三、日志规约

1. 【强制】依赖 SLF4J、JCL 等日志门面，不直接使用 Log4j、Logback API。
2. 【强制】普通日志至少保存 15 天；安全事件和个人敏感信息操作日志不少于六个月并进行多机备份。
3. 【强制】扩展日志命名为 `appName_logType_logName.log`。
4. 【强制】日志字符串拼接使用占位符，例如 `logger.debug("id: {}", id)`。
5. 【强制】trace/debug/info 日志输出前进行日志级别开关判断。
6. 【强制】配置 `additivity=false`，避免重复打印日志。
7. 【强制】生产环境禁止使用 `System.out`、`System.err` 和 `e.printStackTrace()` 输出日志。
8. 【强制】异常日志包含案发现场信息和异常堆栈信息。
9. 【强制】日志打印禁止直接使用 JSON 工具将对象转为字符串。
10. 【推荐】谨慎记录日志，生产环境禁止 debug，合理使用 info 和 warn，避免日志量过大。
11. 【推荐】用户输入参数错误使用 warn，系统逻辑错误、异常和重要错误使用 error。
12. 【推荐】日志错误信息尽量使用英文，英文无法清晰表达时使用中文。

### FOM3 衔接

- Service impl 用 `@Slf4j`（layering §5.1），关键业务操作打 info（含业务主键），跨服务调用失败打 error（含堆栈）。
- 定时任务 job 用游标分页，单页失败 try-catch 记 error 继续，不中断整批（layering §6.3）。

## 十四、单元测试

1. 【强制】单元测试遵守 AIR 原则：Automatic（自动化）、Independent（独立性）、Repeatable（可重复）。
2. 【强制】单元测试全自动、非交互式，使用 assert 验证，不使用 `System.out` 人工检查。
3. 【强制】测试用例相互独立，不依赖执行顺序。
4. 【强制】测试不受网络、服务、中间件等外部环境影响，依赖应注入并使用本地或 Mock 实现。
5. 【强制】测试粒度最多到类级别，一般为方法级别。
6. 【强制】核心业务、核心应用和核心模块的增量代码确保单元测试通过。
7. 【强制】测试代码放在 `src/test/java`，不得放在业务代码目录。
8. 【推荐】基本语句覆盖率达到 70%，核心模块语句覆盖率和分支覆盖率达到 100%。
9. 【推荐】测试遵守 BCDE 原则：Border（边界）、Correct（正确输入）、Design（结合设计）、Error（错误输入）。
10. 【推荐】数据库测试使用程序或导入方式准备数据，不假设数据存在。
11. 【推荐】数据库测试自动回滚，或为测试数据增加明确前后缀。
12. 【推荐】及时重构不可测代码，使代码可测试。
13. 【推荐】设计评审阶段由开发和测试共同确定单元测试范围。
14. 【推荐】项目提测前完成单元测试，不要发布后补充。
15. 【参考】业务代码避免构造方法过重、全局变量和静态方法过多、外部依赖过多、条件语句过多。
16. 【参考】不要误解单元测试是测试人员的专属工作，也不要认为测试代码无需维护或与线上故障无关。

### FOM3 衔接

- AGENTS.md 注明：Java 模块当前无单元测试；新增核心业务逻辑时应补测试，遵循 AIR + BCDE；前端 Vitest 用例与被测代码同目录（`src/views/ai/`、`src/components/Bitable/` 等改动时补用例）。

## 十五、安全规约

1. 【强制】用户个人页面和功能必须进行权限控制校验，防止越权访问。
2. 【强制】用户敏感数据禁止直接展示，必须脱敏。
3. 【强制】SQL 参数使用参数绑定或元数据限定，禁止字符串拼接 SQL。
4. 【强制】用户请求的所有参数必须做有效性验证，防范溢出、慢查询、缓存击穿、SSRF、重定向、注入和 ReDoS。
5. 【强制】禁止向 HTML 输出未经安全过滤或转义的用户数据。
6. 【强制】表单和 AJAX 提交必须执行 CSRF 安全验证。
7. 【强制】外部重定向目标地址必须执行白名单过滤。
8. 【强制】短信、邮件、电话、下单、支付等平台资源必须实现防重放机制。
9. 【推荐】发帖、评论和即时消息等用户生成内容实现防刷和违禁词过滤。

### FOM3 衔接

- MyBatis XML 统一 `#{}` 禁 `${}`（见 mysql-conventions §四）；跨域数据走 Feign，不直接查别域表（共用数据源只读例外）。
- `jnpf.*` 框架提供用户/权限上下文（`UserProvider`），业务侧复用，不自建鉴权。

## 十六、设计规约

1. 【强制】存储方案和底层数据结构设计必须评审通过并沉淀为文档，生产执行后进行 double check。
2. 【强制】交互 User 超过一类且相关 User Case 超过 5 个时使用用例图。
3. 【强制】业务对象状态超过 3 个时使用状态图并明确状态转换触发条件。
4. 【强制】功能调用链涉及对象超过 3 个时使用时序图并明确各环节输入输出。
5. 【强制】模型类超过 5 个且依赖复杂时使用类图。
6. 【强制】超过 2 个对象存在协作且流程复杂时使用活动图。
7. 【推荐】架构设计明确系统边界、模块关系、演化原则和非功能性需求。
8. 【推荐】需求分析和系统设计同时评估异常流程与业务边界。
9. 【推荐】类的设计与实现遵守单一职责原则。
10. 【推荐】谨慎使用继承，优先使用聚合或组合；使用继承时遵守里氏替换原则。
11. 【推荐】根据依赖倒置原则，尽量依赖抽象类和接口。
12. 【推荐】对扩展开放，对修改闭合。
13. 【推荐】抽取公共模块、配置、类和方法，遵守 DRY 原则。
14. 【推荐】敏捷开发不是"讲故事 + 编码 + 发布"，关键设计和文档仍需沉淀。
15. 【参考】设计文档用于明确需求、理顺逻辑和后期维护，也可指导编码。
16. 【参考】可扩展性的本质是找到并隔离系统变化点。
17. 【参考】设计的本质是识别和表达系统难点。
18. 【参考】代码不是完整文档，调用关系、业务逻辑和非功能需求仍需文档表达。
19. 【参考】无障碍设计需支持 Tab 聚焦、合理焦点顺序、验证码替代方式和明确的自定义控件交互。

### FOM3 衔接

- FOM3 分层架构（entity/api/biz/controller/server）即设计落地的骨架（layering 全文），新增领域/聚合前先对照分层与依赖红线（layering §10）。
- 业务流程用 LiteFlow 编排，链路名见 `kocel.common.ChainNameConstants`，节点逻辑在 `kocel.component.*`（AGENTS.md）；状态流转用枚举 + 状态图思路建模。
