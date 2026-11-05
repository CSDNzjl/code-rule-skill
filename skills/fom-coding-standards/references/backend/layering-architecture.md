# 后端代码分层架构规范

本规范从 `fom-example` 骨架提炼，适用于所有 `fom-<domain>` 后端微服务。文中 `<domain>` 指领域名（如 `example`、`plan`、`smelt`），`<Domain>` 指其首字母大写形式，`<Aggregate>` 指聚合根/主表名（如 `PlanProductionOrder`）。同一聚合的各类共享 `<Aggregate>` 前缀。

## 目录

- [1. 模块结构与依赖链](#1-模块结构与依赖链)
- [2. 包命名空间约定](#2-包命名空间约定)
- [3. entity 层](#3-entity-层)
- [4. api 层](#4-api-层)
- [5. biz 层](#5-biz-层)
- [5.1 application（应用服务）](#51-application应用服务)
  - [5.1.1 extension（扩展处理类）](#511-extension扩展处理类)
- [5.2 infrastructure（仓储）](#52-infrastructure仓储)
  - [5.3 mapper（数据访问接口）](#53-mapper数据访问接口)
  - [5.4 convertor（对象转换器）](#54-convertor对象转换器)
  - [5.5 exception（断言与业务异常）](#55-exception断言与业务异常)
  - [5.6 config（配置类）](#56-config配置类)
  - [5.7 utils（工具类）](#57-utils工具类)
- [6. controller 层](#6-controller-层)
  - [6.1 controller（REST 控制器）](#61-controllerrest-控制器)
  - [6.2 feign（Feign 服务端实现）](#62-feignfeign-服务端实现)
  - [6.3 job（定时任务入口）](#63-job定时任务入口)
- [7. server 层](#7-server-层)
- [8. 字段约定](#8-字段约定)
- [9. Mapper XML 规范](#9-mapper-xml-规范)
- [10. 依赖红线](#10-依赖红线)

---

## 1. 模块结构与依赖链

每个 `fom-<domain>/` 是 5 个 Maven 子模块，`pom.xml`（packaging=pom）聚合，依赖方向只能自上而下：

```
fom-<domain>-server ─► fom-<domain>-controller ─► fom-<domain>-biz ─► fom-<domain>-api ─► fom-<domain>-entity
```

各模块 pom 依赖约定（以 fom-base / fom-plan 实际 pom 为准）：

| 模块 | 依赖 | 说明 |
|------|------|------|
| `fom-<domain>-entity` | `kocel-common-database` | 仅提供 MyBatis-Plus 注解，无内部依赖 |
| `fom-<domain>-api` | `fom-<domain>-entity` + `kocel-common-feign` | 对外契约，出入参用 entity 层 DTO/VO |
| `fom-<domain>-biz` | `fom-<domain>-api` + `kocel-cloud-base` + 各兄弟领域 `-api` + `liteflow-editor-*` | 业务核心，依赖汇聚点；entity 经 api 传递引入，pom 不直连 |
| `fom-<domain>-controller` | `fom-<domain>-biz` + `kocel-common-springaop` | 只依赖 biz，不直连 entity/api |
| `fom-<domain>-server` | `fom-<domain>-controller` + `liteflow-editor-starter` | spring-boot-maven-plugin 打包 |

- 本地各域模块自身 version 为 `1.0.0`（父 pom `fom-cloud` 为 `5.0.0-RELEASE`）；跨域依赖兄弟领域 `-api` 时坐标版本写 `5.0.0-RELEASE`（私服发布版本）。
- `server` 的 `spring-boot-maven-plugin` 指定 `<mainClass>kocel.Fom<Domain>Application</mainClass>` + `<layout>ZIP</layout>`。
- 跨领域共享接口（`-api`/`-entity`）改动后，须 `mvn install` 该模块，下游领域才能解析到新 jar，否则静默用旧构件。

## 2. 包命名空间约定

| 前缀 | 归属 | 可改 |
|------|------|------|
| `kocel.*` | 本仓库代码 | ✅ |
| `fom.*` | 跨服务 Feign API（多领域共享） | ✅（影响下游） |
| `jnpf.*` | 基础平台框架，来自外部 jar | ❌ |

各模块包根：

```
fom-<domain>-entity      kocel.<domain>
fom-<domain>-api        kocel.<domain>          (+ kocel.<domain>.fallback)
fom-<domain>-biz        kocel                   ← 顶层 kocel，不带域名
  ├── kocel.application  (+ kocel.application.impl + kocel.application.extension)
  ├── kocel.infrastructure  (+ kocel.infrastructure.mapper)
  ├── kocel.convertor
  ├── kocel.exception
  ├── kocel.config
  └── kocel.utils
fom-<domain>-controller kocel
  ├── kocel.controller
  ├── kocel.feign
  └── kocel.job
fom-<domain>-server     kocel                   (启动类 kocel.Fom<Domain>Application)
```

> biz/controller/server 用顶层 `kocel.*`（不带域名），因为启动类 `@ComponentScan({"kocel","jnpf","fom"})` 扫描 `kocel` 全包。entity/api 带 `<domain>` 子包是为了与兄弟领域同名类区分（各领域都有 `model.entity`、`enums` 等）。

---

## 3. entity 层

包 `kocel.<domain>`，子包：`model.entity` / `model.dto` / `model.vo` / `model.request` / `enums` / `constants`。

### 3.1 数据库实体（model.entity）

纯数据结构，`@Data` + `@TableName`，每个字段显式 `@TableField` + 中文 Javadoc。

```java
package kocel.<domain>.model.entity;

import com.baomidou.mybatisplus.annotation.*;
import lombok.Data;
import java.time.LocalDateTime;

/**
 * 生产订单表
 * plan_production_order
 */
@Data
@TableName("plan_production_order")
public class PlanProductionOrderEntity {

    /** 主键 */
    @TableId(type = IdType.ASSIGN_ID)
    private Long id;

    /** 销售订单号 */
    @TableField("sale_order_no")
    private String saleOrderNo;

    /** 订单状态 */
    @TableField("order_status")
    private String orderStatus;

    // …业务字段…

    /** 创建时间 */
    @TableField("create_time")
    private LocalDateTime createTime;
    /** 更新时间 */
    @TableField("update_time")
    private LocalDateTime updateTime;
    /** 创建人id */
    @TableField("create_user")
    private Long createUser;
    /** 更新人id */
    @TableField("update_user")
    private Long updateUser;

    /** 删除标志 null:未删除 1:已删除 */
    @TableLogic
    @TableField("f_delete_mark")
    private Integer fDeleteMark;
    /** 删除时间 */
    @TableField("f_delete_time")
    private LocalDateTime fDeleteTime;
    /** 删除用户 */
    @TableField("f_delete_user_id")
    private Long fDeleteUserId;

    // —— 以下流程/租户字段按需添加，非每张表强制（仅当表接入审批流程或多租户时）——
    /** 租户id */
    @TableField("f_tenant_id")
    private String fTenantId;
    /** 流程id */
    @TableField("f_flow_id")
    private String fFlowId;
    /** 流程任务主键 */
    @TableField("f_flow_task_id")
    private String fFlowTaskId;
    /** 流程节点编码 */
    @TableField("f_flow_node_code")
    private String fFlowNodeCode;
}
```

要点：
- 主键统一 `@TableId(type = IdType.ASSIGN_ID)`（雪花 ID）。
- 每个 `@TableField` 显式写数据库列名（snake_case），不依赖驼峰自动映射。
- 逻辑删除字段固定 `f_delete_mark`，标注 `@TableLogic`。审计字段、逻辑删除字段、流程/租户字段见 [§8](#8-字段约定)。
- 跨域只读表（归属别域、共用数据源本地直查）在类 Javadoc 注明归属与直查原因。

### 3.2 DTO / VO（model.dto / model.vo）

纯 POJO，`@Data`，**无 MyBatis 注解**。DTO 用于跨服务传输，VO 用于返回前端。字段按需裁剪，不必照搬 Entity 全部字段。

```java
package kocel.<domain>.model.dto;

import lombok.Data;
import java.time.LocalDateTime;

@Data
public class PlanProductionOrderDTO {
    /** 主键 */
    private Long id;
    /** 生产订单号 */
    private String productionOrderNo;
    // …对外传输所需字段…
}
```

### 3.3 请求对象（model.request）

- 分页查询：`<Aggregate>PageReq`，只放查询条件字段。
- 事件/动作请求：`<Aggregate>EventReq`，用 `jakarta.validation` 校验，校验消息中文。

```java
package kocel.<domain>.model.request;

import jakarta.validation.constraints.NotBlank;
import jakarta.validation.constraints.NotNull;
import lombok.Data;

@Data
public class OrderEndEventReq {
    /** 流程ID */
    @NotBlank(message = "流程ID不能为空")
    private String flowId;
    /** 订单ID */
    @NotNull(message = "订单ID不能为空")
    private Long orderId;
}
```

### 3.4 枚举（enums）

`@Getter`，`code` + `message` 双字段，提供 `getInstance(code)` 静态反查。同一业务对象的两套编码（如入参动作编码 ↔ 审批流程模板编码）一一对应时，**合并进一个枚举**，多加编码字段并提供按各编码反查的静态方法，不拆成两个枚举互相引用。

```java
package kocel.<domain>.enums;

import lombok.Getter;
import java.util.Arrays;
import java.util.Objects;

@Getter
public enum OrderAuditActionEnum {
    FREEZEN("FREEZEN", "冻结", "PlanOrderFreezenApproval", "计划生产订单冻结审批"),
    TERMINATED("TERMINATED", "终止", "PlanOrderTerminateApproval", "计划生产订单终止审批"),
    UNFREEZE("UNFREEZE", "解冻", "PlanOrderUnFreezeApproval", "计划生产订单解冻审批"),
    ;

    /** 变更类型（动作）编码，即入参 changeType 的取值 */
    private final String code;
    private final String message;
    /** 审批流程模板编码（enCode） */
    private final String flowCode;
    /** 审批流程模板名称 */
    private final String flowMessage;

    OrderAuditActionEnum(String code, String message, String flowCode, String flowMessage) {
        this.code = code;
        this.message = message;
        this.flowCode = flowCode;
        this.flowMessage = flowMessage;
    }

    /** 按变更类型编码反查 */
    public static OrderAuditActionEnum getInstance(String code) {
        return Arrays.stream(values()).filter(x -> Objects.equals(x.code, code)).findFirst().orElse(null);
    }

    /** 按审批流程模板编码反查 */
    public static OrderAuditActionEnum getByFlowCode(String flowCode) {
        return Arrays.stream(values()).filter(x -> Objects.equals(x.flowCode, flowCode)).findFirst().orElse(null);
    }
}
```

### 3.5 常量（constants）

如 Feign 服务名等跨模块共享常量用接口承载：

```java
package kocel.<domain>.constants;

public interface FeignName {
    String PLAN_SERVER_NAME = "fom-<domain>";
}
```

---

## 4. api 层

包 `kocel.<domain>`（+ `kocel.<domain>.fallback`）。对外跨服务调用的契约，只放 Feign 接口 + 降级，**不写业务**。

### 4.1 Feign 接口

```java
package kocel.<domain>;

import jnpf.base.ActionResult;
import kocel.<domain>.constants.PlanFeignName;
import kocel.<domain>.fallback.PlanProductOrderApiFallback;
import kocel.<domain>.model.dto.PlanProductionOrderDTO;
import org.springframework.cloud.openfeign.FeignClient;
import org.springframework.web.bind.annotation.GetMapping;
import org.springframework.web.bind.annotation.RequestParam;

@FeignClient(
    name = PlanFeignName.PLAN_SERVER_NAME,
    fallback = PlanProductOrderApiFallback.class,
    path = "/feign/productionOrder"
)
public interface PlanProductOrderApi {
    @GetMapping("/getByOrderNo")
    ActionResult<PlanProductionOrderDTO> getByOrderNo(@RequestParam("productionOrderNo") String productionOrderNo);
}
```

### 4.2 降级实现

```java
package kocel.<domain>.fallback;

import jnpf.base.ActionResult;
import kocel.<domain>.PlanProductOrderApi;
import kocel.<domain>.model.dto.PlanProductionOrderDTO;
import org.springframework.stereotype.Component;

@Component
public class PlanProductOrderApiFallback implements PlanProductOrderApi {
    @Override
    public ActionResult<PlanProductionOrderDTO> getByOrderNo(String productionOrderNo) {
        return ActionResult.fail("根据订单号查询订单信息失败，服务不可用");
    }
}
```

---

## 5. biz 层

包 `kocel`（不带域名）。DDD 风格分层：application（编排）/ infrastructure（数据访问）/ convertor（转换）/ exception（断言异常）/ config / utils。

### 5.1 application（应用服务）

`application/` 放 Service 接口，`application/impl/` 放实现。

**接口**：方法签名用 DTO/VO/Request + 框架分页类型（`PageRequest`/`PageListVO`）。

```java
package kocel.application;

import jnpf.base.PageRequest;
import jnpf.base.vo.PageListVO;
import kocel.<domain>.model.dto.PlanProductionOrderDTO;
import kocel.<domain>.model.request.OrderEndEventReq;
import kocel.<domain>.model.request.PlanProductionOrderPageReq;
import kocel.<domain>.model.vo.PlanProductionOrderVO;
import java.util.List;

public interface PlanProductionOrderService {

    /**
     * 生产订单分页查询
     *
     * @param request 分页请求
     * @return 分页结果
     */
    PageListVO<PlanProductionOrderVO> getPageList(PageRequest<PlanProductionOrderPageReq> request);

    /**
     * 根据生产订单号查询订单
     *
     * @param productionOrderNo 生产订单号
     * @return 生产订单 DTO
     */
    PlanProductionOrderDTO getByOrderNo(String productionOrderNo);

    /**
     * 生产订单审核-发起事件，按变更类型发起审批流程
     *
     * @param req 发起事件请求
     */
    void initiateEvent(OrderInitiateEventReq req);
}
```

**实现**：`@Slf4j @Service @RequiredArgsConstructor`，构造器注入（final 字段 + Lombok）。注入 Mapper、Repository、跨服务 API（如 `TemplateApi`/`TaskApi`）、`UserProvider`。

```java
package kocel.application.impl;

import jnpf.base.PageRequest;
import jnpf.base.vo.PageListVO;
import jnpf.base.vo.PaginationVO;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import kocel.application.PlanProductionOrderService;
import kocel.convertor.PlanProductionOrderConvertor;
import kocel.exception.Assert;
import kocel.exception.BizException;
import kocel.infrastructure.PlanProductionOrderRepository;
import kocel.infrastructure.mapper.PlanProductionOrderMapper;
import kocel.<domain>.model.entity.PlanProductionOrderEntity;
import kocel.<domain>.model.request.PlanProductionOrderPageReq;
import kocel.<domain>.model.vo.PlanProductionOrderVO;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.stereotype.Service;
import java.util.List;

@Slf4j
@Service
@RequiredArgsConstructor
public class PlanProductionOrderServiceImpl implements PlanProductionOrderService {

    private final PlanProductionOrderMapper planProductionOrderMapper;
    private final PlanProductionOrderRepository planProductionOrderRepository;

    @Override
    public PageListVO<PlanProductionOrderVO> getPageList(PageRequest<PlanProductionOrderPageReq> request) {
        PlanProductionOrderPageReq query = request.getData() == null
                ? new PlanProductionOrderPageReq() : request.getData();
        long currentPage = request.getCurrentPage() == null || request.getCurrentPage() <= 0
                ? 1L : request.getCurrentPage();
        long pageSize = request.getPageSize() == null || request.getPageSize() <= 0
                ? 20L : request.getPageSize();

        Page<PlanProductionOrderEntity> page = new Page<>(currentPage, pageSize);
        Page<PlanProductionOrderEntity> result = planProductionOrderMapper.selectOrderPage(page, query);

        List<PlanProductionOrderVO> voList = PlanProductionOrderConvertor.toVOList(result.getRecords());

        PaginationVO pagination = new PaginationVO();
        pagination.setTotal(Math.toIntExact(result.getTotal()));
        pagination.setCurrentPage(currentPage);
        pagination.setPageSize(pageSize);

        PageListVO<PlanProductionOrderVO> pageListVO = new PageListVO<>();
        pageListVO.setList(voList);
        pageListVO.setPagination(pagination);
        return pageListVO;
    }

    @Override
    public void endEvent(OrderEndEventReq req) {
        log.info("生产订单审核-结束事件, 订单id:{}", req.getOrderId());
        PlanProductionOrderEntity order = planProductionOrderRepository.getById(req.getOrderId());
        Assert.notNull(order, "存在找不到的生产订单");
        // …状态判定 + 持久化…
        planProductionOrderRepository.updateOrderStatus(order.getId(), targetStatus, updateUser);
    }
}
```

要点：
- 业务编排四步：**校验入参（Assert）→ 状态/规则判定（枚举 + switch）→ 调 infrastructure 或跨服务 API → convertor 转 VO/DTO 返回**。
- 异常用 `Assert` / `BizException`，不要 `throw new RuntimeException`。跨服务调用 `try-catch` 包成 `BizException`。
- 事务边界在 application 层；需要事务的方法自行加 `@Transactional`，方法命名无强制后缀。
- 拿到 Entity 后用 Convertor 转 VO/DTO，不在 Service 里手写字段拷贝。
- Service 接口的每个方法必须加中文 Javadoc，统一多行格式（`/**` 单起一行，描述、`@param`、`@return` 各占一行，`*/` 收尾），一句话说明"做什么"，描述与对应 Controller 的 `@Operation(summary)` 保持一致，并补 `@param`/`@return`（coding-conventions §九 FOM3 衔接）。
- Service impl 方法在关键位置加行注释 `//`（写在被注释语句上方）：状态/规则判定、跨服务调用、委托 extension、分支边界（如早返回）、持久化等，解释"为什么这样做"或"这一步在做什么"，不注释显而易见的代码。
- 某个 Service 方法逻辑过重时，把可独立、可复用的处理步骤抽到 `application/extension/`（见 [§5.1.1](#511-extension扩展处理类)），impl 通过构造注入并委托，保持 impl 主干精简。

#### 5.1.1 extension（扩展处理类）

`application/extension/` 放从 impl 拆出的、可独立成块的处理逻辑。典型场景：一个 Service 方法要"查模板 → 组装流程模型 → 提交审批 → 断言结果"多步串联，把其中可独立、可复用的处理步骤抽成一个类放到这里，impl 通过构造注入并委托调用。

- 类名以 `Handler`（处理一段同步流程）或 `Task`（执行一个独立任务）结尾，如 `CommonFlowHandler`、`OrderSyncTask`。
- 标 `@Component`（区别于 Service 的 `@Service`），`@RequiredArgsConstructor` 构造注入自身依赖（Repository、跨服务 `*Api` 等）。
- impl 里 `private final CommonFlowHandler commonFlowHandler;` 注入，方法体里 `commonFlowHandler.handle(...)` 委托。
- 拆分的目的是让 impl 保持精简——每个 Service 方法读起来仍是"校验 → 判定 → 委托/持久化 → 返回"的主干，细节处理交给 extension；可复用的处理（如通用的审批发起）也借此在多个 Service 间共享，并便于单独单测。
- extension 不开新事务边界，事务仍由 application 层的 Service 方法把控；extension 只做处理，不做跨域持久化（跨域数据仍走 Feign `-api`）。
- 类头和每个公开方法必须加中文 Javadoc：类头说明职责（如"通用审批流程处理类"），公开方法说明"做什么"并用 `@param` 标注关键参数（coding-conventions §九 FOM3 衔接）。

模板（基于 `CommonFlowHandler`，封装"按流程编码发起审批"）：

```java
package kocel.application.extension;

import jnpf.base.ActionResult;
import jnpf.base.UserInfo;
import jnpf.flowable.TaskApi;
import jnpf.flowable.TemplateApi;
import jnpf.flowable.entity.TemplateEntity;
import jnpf.flowable.model.task.FlowModel;
import kocel.exception.Assert;
import kocel.exception.BizException;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Component;

import java.util.Map;

@Component
@RequiredArgsConstructor
public class CommonFlowHandler {

    private final TemplateApi templateApi;
    private final TaskApi taskApi;

    public void handle(Map<String, Object> formData, String flowCode, UserInfo userInfo) {
        TemplateEntity templateEntity = templateApi.getInfoByEnCode(flowCode);
        Assert.notNull(templateEntity, "审批流程不存在");

        FlowModel flowModel = new FlowModel();
        flowModel.setFlowId(templateEntity.getFlowId());
        flowModel.setFormData(formData);
        flowModel.setUserInfo(userInfo);
        // status：0-暂存  1-提交
        flowModel.setStatus(1);
        // flowUrgent：1-普通 2-重要 3-紧急
        flowModel.setFlowUrgent(2);
        // isFlow：0-菜单 1-发起
        flowModel.setIsFlow(1);

        ActionResult actionResult;
        try {
            actionResult = taskApi.saveOrSubmit(flowModel);
        } catch (Exception e) {
            throw new BizException("流程提交失败: " + e.getMessage(), e);
        }
        Assert.isTrue(actionResult.getCode() == 200, actionResult.getMsg());
    }
}
```

impl 端委托（`PlanProductionOrderServiceImpl.initiateEvent` 在状态/规则判定之后收尾）：

```java
private final CommonFlowHandler commonFlowHandler;

// …校验 + 状态/规则判定（枚举 + switch 得到 approval）…
UserInfo userInfo = userProvider.get();
commonFlowHandler.handle(Toolkit.beanToMap(order), approval.getCode(), userInfo);
```

### 5.2 infrastructure（仓储）

`infrastructure/` 放 Repository，`@Service`，`extends ServiceImpl<Mapper, Entity>`。封装 MyBatis-Plus 查询（LambdaQueryWrapper / LambdaUpdateWrapper）。

```java
package kocel.infrastructure;

import com.baomidou.mybatisplus.core.conditions.query.LambdaQueryWrapper;
import com.baomidou.mybatisplus.core.conditions.update.LambdaUpdateWrapper;
import com.baomidou.mybatisplus.core.toolkit.Wrappers;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import kocel.<domain>.model.entity.PlanProductionOrderEntity;
import kocel.infrastructure.mapper.PlanProductionOrderMapper;
import org.springframework.stereotype.Service;
import java.time.LocalDateTime;
import java.util.List;

@Service
public class PlanProductionOrderRepository extends ServiceImpl<PlanProductionOrderMapper, PlanProductionOrderEntity> {

    public PlanProductionOrderEntity getByOrderNo(String productionOrderNo) {
        LambdaQueryWrapper<PlanProductionOrderEntity> wrapper = Wrappers.lambdaQuery();
        wrapper.eq(PlanProductionOrderEntity::getProductionOrderNo, productionOrderNo);
        return this.getOne(wrapper);
    }

    public void updateOrderStatus(Long id, String orderStatus, Long updateUser) {
        LambdaUpdateWrapper<PlanProductionOrderEntity> wrapper = Wrappers.lambdaUpdate();
        wrapper.eq(PlanProductionOrderEntity::getId, id)
                .set(PlanProductionOrderEntity::getOrderStatus, orderStatus)
                .set(PlanProductionOrderEntity::getUpdateTime, LocalDateTime.now())
                .set(PlanProductionOrderEntity::getUpdateUser, updateUser);
        this.update(wrapper);
    }
}
```

要点：
- Repository 是数据访问的唯一出口，Service 只通过 Repository 操作持久化数据（简单单表 CRUD 也可直接用 `ServiceImpl` 继承来的方法，复杂查询在此封装）。
- 跨域只读仓储（归属别域、共用数据源直查）单独建 Repository，类 Javadoc 注明归属与直查原因。
- 游标分页批处理：`select(id).gt(id, lastId).orderByAsc(id).last("LIMIT " + size)`。

### 5.3 mapper（数据访问接口）

`infrastructure/mapper/`，`extends BaseMapper<Entity>`。只放接口签名，复杂 SQL 走 XML。

```java
package kocel.infrastructure.mapper;

import com.baomidou.mybatisplus.core.mapper.BaseMapper;
import com.baomidou.mybatisplus.extension.plugins.pagination.Page;
import kocel.<domain>.model.entity.PlanProductionOrderEntity;
import kocel.<domain>.model.request.PlanProductionOrderPageReq;
import org.apache.ibatis.annotations.Param;

public interface PlanProductionOrderMapper extends BaseMapper<PlanProductionOrderEntity> {
    Page<PlanProductionOrderEntity> selectOrderPage(
            Page<PlanProductionOrderEntity> page, @Param("query") PlanProductionOrderPageReq query);
}
```

- 启动类 `@MapperScan(basePackages = "kocel.infrastructure.mapper")` 扫描此包。
- 简单单表 CRUD 直接用 `BaseMapper` 自带方法，不重复定义。
- 自定义查询用 `@Param` 传参，XML 里用 `#{query.xxx}` 取值。

### 5.4 convertor（对象转换器）

`convertor/`，静态工具类，`Entity → VO/DTO`，用 `BeanUtils.copyProperties`。

```java
package kocel.convertor;

import kocel.<domain>.model.dto.PlanProductionOrderDTO;
import kocel.<domain>.model.entity.PlanProductionOrderEntity;
import kocel.<domain>.model.vo.PlanProductionOrderVO;
import org.springframework.beans.BeanUtils;
import java.util.Collections;
import java.util.List;
import java.util.stream.Collectors;

public class PlanProductionOrderConvertor {
    public static PlanProductionOrderVO toVO(PlanProductionOrderEntity entity) {
        if (entity == null) return null;
        PlanProductionOrderVO vo = new PlanProductionOrderVO();
        BeanUtils.copyProperties(entity, vo);
        return vo;
    }
    public static List<PlanProductionOrderVO> toVOList(List<PlanProductionOrderEntity> list) {
        if (list == null || list.isEmpty()) return Collections.emptyList();
        return list.stream().map(PlanProductionOrderConvertor::toVO).collect(Collectors.toList());
    }
    public static PlanProductionOrderDTO toDTO(PlanProductionOrderEntity entity) {
        if (entity == null) return null;
        PlanProductionOrderDTO dto = new PlanProductionOrderDTO();
        BeanUtils.copyProperties(entity, dto);
        return dto;
    }
}
```

- 字段名一致的批量拷贝交给 `BeanUtils`，不一致的个别字段手动 set，不要在 Service 里散写拷贝逻辑。

### 5.5 exception（断言与业务异常）

`exception/`，`Assert` 断言工具 + `BizException` 业务异常。校验失败抛 `BizException`，**不要**用 `RuntimeException`。

```java
// Assert：校验失败抛 BizException / DataException
Assert.notNull(order, "存在找不到的生产订单");
Assert.isTrue(actionResult.getCode() == 200, actionResult.getMsg());
Assert.isFalse(isComplete, "该订单已完成,无法冻结: " + productionOrderNo);

// BizException：业务异常，已知错误不需重试
throw new BizException("流程提交失败: " + e.getMessage(), e);
```

- `Assert.notNull/isTrue/isFalse/notEmpty`，重载支持 `(condition, errCode, msg)` 与 `(condition, msg)`。
- 跨服务调用异常 `try-catch` 包成 `BizException(msg, e)` 保留原因链。

### 5.6 config（配置类）

`config/`，`@Data @Component @RefreshScope`，`@Value` 读 Nacos 配置，带默认值 `:`。

```java
package kocel.config;

import lombok.Data;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.cloud.context.config.annotation.RefreshScope;
import org.springframework.stereotype.Component;

@Data
@Component
@RefreshScope
public class ExampleConfig {
    /** erp接口地址 */
    @Value("${config.erp.host:}")
    private String erpHost;
}
```

- `@RefreshScope` 让 Nacos 配置变更后动态刷新，不要漏。
- 真正的运行时配置（数据源/Redis/MQ/LiteFlow）在 Nacos，不在仓库；`bootstrap.yml` 拉取 data-id。

### 5.7 utils（工具类）

`utils/`，静态工具类。如 `Toolkit.beanToMap`、`Toolkit.sortByKey` 等。纯静态方法，无状态。

---

## 6. controller 层

包 `kocel`，子包 `controller` / `feign` / `job`。

### 6.1 controller（REST 控制器）

`@Slf4j @RestController @RequestMapping("/<aggregate>") @Tag`。`@Autowired` 注入 Service。返回 `ActionResult` 包装。

```java
package kocel.controller;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jakarta.validation.Valid;
import jnpf.base.ActionResult;
import jnpf.base.PageRequest;
import jnpf.base.vo.PageListVO;
import kocel.application.PlanProductionOrderService;
import kocel.<domain>.model.request.OrderEndEventReq;
import kocel.<domain>.model.request.PlanProductionOrderPageReq;
import kocel.<domain>.model.vo.PlanProductionOrderVO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@Slf4j
@RestController
@RequestMapping("/productionOrder")
@Tag(name = "生产订单", description = "<domain>")
public class PlanProductionOrderController {

    @Autowired
    private PlanProductionOrderService planProductionOrderService;

    @Operation(summary = "生产订单分页查询")
    @PostMapping("/page")
    public ActionResult<PageListVO<PlanProductionOrderVO>> getPageList(
            @RequestBody PageRequest<PlanProductionOrderPageReq> request) {
        PageListVO<PlanProductionOrderVO> vo = planProductionOrderService.getPageList(request);
        return ActionResult.page(vo.getList(), vo.getPagination());
    }

    @Operation(summary = "生产订单审核-结束事件")
    @PostMapping("/endEvent")
    public ActionResult<Void> endEvent(@RequestBody @Valid OrderEndEventReq req) {
        planProductionOrderService.endEvent(req);
        return ActionResult.success();
    }
}
```

要点：
- **控制器只做协议转换**：HTTP → 调 Service → `ActionResult` 包装。不写业务逻辑。
- 分页返回 `ActionResult.page(list, pagination)`，普通返回 `ActionResult.success(data)`，无返回 `ActionResult.success()`。
- 入参带校验用 `@Valid`，请求体 `@RequestBody`，单参 `@RequestParam`。
- `@Tag(name=..., description="<domain>")` 标注所属领域，Knife4j 文档按此分组。

### 6.2 feign（Feign 服务端实现）

`implements <Aggregate>Api`（api 层定义的 Feign 契约），委托给 Service。路径 `/feign/<aggregate>`。

```java
package kocel.feign;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jnpf.base.ActionResult;
import kocel.application.PlanProductionOrderService;
import kocel.<domain>.PlanProductOrderApi;
import kocel.<domain>.model.dto.PlanProductionOrderDTO;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RestController;

@Slf4j
@RestController
@Tag(name = "生产订单-对外API", description = "<domain>")
public class PlanProductionOrderClient implements PlanProductOrderApi {
    @Autowired
    private PlanProductionOrderService planProductionOrderService;

    @Override
    @Operation(summary = "根据订单号查询订单信息")
    public ActionResult<PlanProductionOrderDTO> getByOrderNo(String productionOrderNo) {
        return ActionResult.success(planProductionOrderService.getByOrderNo(productionOrderNo));
    }
}
```

- Feign 契约（api 层 `@FeignClient` 的 path）与服务端 `@RequestMapping` 路径须一致（`/feign/<aggregate>`）。
- 出入参类型须与 api 层契约完全一致。

### 6.3 job（定时任务入口）

`@RestController @RequestMapping("/job/<aggregate>")`，由外部调度系统 HTTP 触发。批处理用游标分页。

```java
package kocel.job;

import io.swagger.v3.oas.annotations.Operation;
import io.swagger.v3.oas.annotations.tags.Tag;
import jnpf.base.ActionResult;
import kocel.infrastructure.PlanProductionOrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.web.bind.annotation.*;
import java.util.List;

@Slf4j
@RestController
@RequestMapping("/job/productionOrder")
@RequiredArgsConstructor
@Tag(name = "生产订单-定时任务", description = "<domain>")
public class PlanProductionOrderJob {

    private static final int PAGE_SIZE = 500;
    private final PlanProductionOrderRepository planProductionOrderRepository;

    /**
     * 生产订单推送状态同步
     * 游标分页批量更新未同步订单的推送状态，由外部调度系统 HTTP 触发
     *
     * @return 本次累计处理的订单数量
     */
    @Operation(summary = "生产订单推送")
    @PostMapping("/syncOrder")
    public ActionResult<Long> syncOrder() {
        long total = 0L;
        long lastId = 0L;
        while (true) {
            List<Long> ids = planProductionOrderRepository.pageUnsyncedIds(lastId, PAGE_SIZE);
            if (ids == null || ids.isEmpty()) break;
            try {
                planProductionOrderRepository.updateSyncedByIds(ids);
                total += ids.size();
            } catch (Exception e) {
                log.error("[生产订单推送状态同步] 更新失败, lastId={}, 条数={}", lastId, ids.size(), e);
            }
            lastId = ids.get(ids.size() - 1);
        }
        log.info("[生产订单推送状态同步] 完成, 共处理 {} 条", total);
        return ActionResult.success(total);
    }
}
```

- 游标分页（`gt(id, lastId)` + `LIMIT`）避免深度分页性能问题。
- 单页处理失败 `try-catch` 记日志继续，不中断整批。
- 每个端点方法必须加多行 Javadoc 说明接口用途（做什么、由谁触发），并补 `@return`，与 Service 接口注释格式一致（coding-conventions §九 FOM3 衔接）。

---

## 7. server 层

包 `kocel`，启动类 `Fom<Domain>Application`。

```java
package kocel;

import org.mybatis.spring.annotation.MapperScan;
import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.cloud.openfeign.EnableFeignClients;
import org.springframework.context.annotation.ComponentScan;

@SpringBootApplication
@ComponentScan(basePackages = {"kocel","jnpf"})
@MapperScan(basePackages = "kocel.infrastructure.mapper")
@EnableFeignClients(basePackages = {"kocel","jnpf"})
public class FomExampleApplication {
    public static void main(String[] args) {
        SpringApplication.run(FomExampleApplication.class, args);
        System.out.println("<domain>启动成功");
    }
}
```

- 2个扫描注解缺一不可：`kocel`（本仓库）/ `jnpf`（框架）。
- `@MapperScan` 固定指向 `kocel.infrastructure.mapper`。
- `bootstrap.yml`：`spring.application.name=fom-<domain>` + Nacos 地址/namespace + data-id 列表（datasource/resources/mq/liteflow 等，`refresh: true`）。
- `application.yml`：只放本地 `server.port`、日志级别、FOM 许可配置。DB/Redis/MQ 改 Nacos，不改仓库文件。
- 启动校验工作目录下 `license.lic`；JDK 9+ 需 `--add-opens=java.base/...=ALL-UNNAMED` VM 参数。

---

## 8. 字段约定

Entity 通常包含以下字段（数据库表建表时对应加上），其中审计字段与逻辑删除字段为约定字段，流程/租户字段按需添加：

**审计字段**（业务侧维护）：

| Java 字段 | 数据库列 | 说明 |
|-----------|---------|------|
| `createTime` | `create_time` | 创建时间 |
| `updateTime` | `update_time` | 更新时间 |
| `createUser` | `create_user` | 创建人id |
| `updateUser` | `update_user` | 更新人id |

**逻辑删除字段**（`f_` 前缀，约定）：

| Java 字段 | 数据库列 | 注解 | 说明 |
|-----------|---------|------|------|
| `fDeleteMark` | `f_delete_mark` | `@TableLogic` | 逻辑删除 null:未删除 1:已删除 |
| `fDeleteTime` | `f_delete_time` | | 删除时间 |
| `fDeleteUserId` | `f_delete_user_id` | | 删除用户 |

**流程/租户字段**（`f_` 前缀，按需——仅当表接入审批流程或多租户时才加，不强制）：

| Java 字段 | 数据库列 | 说明 |
|-----------|---------|------|
| `fTenantId` | `f_tenant_id` | 租户id |
| `fFlowId` | `f_flow_id` | 流程id |
| `fFlowTaskId` | `f_flow_task_id` | 流程任务主键 |
| `fFlowNodeCode` | `f_flow_node_code` | 流程节点编码 |

- 主键 `id` 用 `Long` + `IdType.ASSIGN_ID`（雪花）。
- 所有 `@TableField` 显式写列名，不依赖驼峰自动映射。
- 查询带逻辑删除的表，条件加 `WHERE f_delete_mark IS NULL`（MyBatis-Plus 对 `@TableLogic` 字段自动追加，自定义 XML 需手写）。

---

## 9. Mapper XML 规范

位置：`fom-<domain>-biz/src/main/resources/mapper/<Aggregate>Mapper.xml`。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="kocel.infrastructure.mapper.PlanProductionOrderMapper">

    <select id="selectOrderPage" resultType="kocel.<domain>.model.entity.PlanProductionOrderEntity">
        SELECT * FROM plan_production_order
        WHERE f_delete_mark IS NULL
        <if test="query.productionOrderNo != null and query.productionOrderNo != ''">
            AND production_order_no = #{query.productionOrderNo}
        </if>
        ORDER BY create_time DESC
    </select>

</mapper>
```

- `namespace` 指向 `kocel.infrastructure.mapper.<Aggregate>Mapper`。
- `resultType` 用 entity 全限定名；复杂结果映射用 `resultMap`。
- 动态条件用 `<if>`，取值 `#{query.xxx}`（对应 Mapper 方法 `@Param("query")`）。
- 涉及逻辑删除表，手写 SQL 显式加 `WHERE f_delete_mark IS NULL`。
- 列名 snake_case，参数占位符统一 `#{}`，禁止字符串拼接防注入。

---

## 10. 依赖红线

以下是**禁止**的依赖方向，违反会导致循环依赖或构件污染：

| ❌ 禁止 | 原因 |
|---------|------|
| entity 依赖 biz/controller/api | entity 是最底层，须零业务依赖 |
| api 依赖 biz/controller | api 是对外契约，只含 Feign 接口 + DTO/VO |
| controller 直接依赖 entity/api/server | controller 只依赖 biz，通过 Service 间接用 |
| biz 依赖别域的 `-biz`/`-controller`/`-entity` | 跨服务只依赖别域 `-api` jar |
| server 依赖 biz/entity/api（跳过 controller） | server 只依赖 controller，由其传递依赖 |
| 在 controller/job 里写业务逻辑 | 业务逻辑属于 application 层 |
| 在 Service 里直接 `new Wrapper` 操作别域表 | 跨域数据走 Feign，本域只操作自家表（共用数据源的跨域只读表例外，须注释说明） |
| 修改 `jnpf.*` 包下的代码 | 来自外部 jar，不可改 |

新增跨领域调用时：在消费方 `-biz` 的 pom 引入 `<被调领域>-api`，用 `@FeignClient` 调用；**不要**直接查别域表（除非共用数据源的跨域只读场景，且 Repository 类 Javadoc 注明）。
