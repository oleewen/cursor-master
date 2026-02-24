# [AI Master Restaurant Order] AI Agent 指南

> 最后更新: 2026-02-24
> 分析进度: [持续维护]

## 项目概述

构建餐厅点餐核心交易引擎，实现从用户下单、库存锁定到订单全生命周期管理的可靠后端服务。

| 属性 | 值 |
|------|-----|
| 项目名称 | ai-master |
| 业务领域 | 餐饮点餐交易引擎 (Restaurant Order) |
| 技术栈摘要 | Java 17, Spring Boot 2.7.10, MyBatis, MySQL, MapStruct, Lombok |
| 代码规模 | ~54 文件 / ~1.7 KLOC |
| 核心模块数 | 8 (boot, api, application, client, common, domain, infrastructure, service) |
| 领域上下文数 | 3 (Order 订单, Goods 商品, Inventory 库存) + 1 (User 用户) |
| 启动类 | `com.only.ai.master.boot.ApplicationStarter` |
| 二方依赖 | Transformer v1.2.10 (内部框架) |
| 启动年份（推测） | 2020 年，基于 `ApplicationStarter` 中 `@since 2020-05-22` 注解 |

## 快速导航

| 导航项 | 路径 | 说明 |
|--------|------|------|
| 核心入口 | [ApplicationStarter.java](ai-master-boot/src/main/java/com/only/ai/master/boot/ApplicationStarter.java) | Spring Boot 启动类 |
| 配置文件 | [application.properties](ai-master-boot/src/main/resources/application.properties) | 环境配置 |
| 产品需求 | [docs/prd.md](docs/prd.md) | 产品需求说明书 |
| 架构设计 | [docs/add.md](docs/add.md) | 架构设计说明书 |
| 测试设计 | [docs/tdd.md](docs/tdd.md) | 测试设计说明书 |
| API文档 | [docs/api/](docs/api/) | API 接口文档 |

## 技术架构

### 技术栈

#### 后端
| 类别 | 技术 | 版本 | 备注 |
|------|------|------|------|
| 语言 | Java | 17 | LTS 版本 |
| 框架 | Spring Boot | 2.7.10 | 核心容器与 Web 框架 |
| ORM | MyBatis | 2.3.2 (starter) | 数据持久层 |
| 数据库 | MySQL | Driver 5.1.30 | 关系型数据库 |
| 连接池 | Druid | 1.2.23 | 阿里巴巴数据库连接池 |
| 对象映射 | MapStruct | 1.5.0.Final | 编译时对象转换 |
| 工具库 | Lombok | 1.18.34 | 简化 POJO 编写 |
| API文档 | Swagger | 2.7.0 | springfox-swagger2 |
| 序列化 | Jackson | (Spring Boot 内置) | SNAKE_CASE 命名策略 |
| 二方库 | Transformer | 1.2.10 | 内部框架 (call, common, dubbo, util, exception, dao) |

#### 基础设施
| 类别 | 技术 | 用途 |
|------|------|------|
| 构建 | Maven 3.8+ | 多模块构建与依赖管理 |
| 部署 | Spring Boot Fat Jar | java -jar 运行 |
| 监控 | `@Call` 注解 | Transformer 提供的调用日志与性能监控 |
| 缓存 | Spring Cache `@Cacheable` | 商品查询缓存 |

### 模块结构

```
ai-master-restaurant-order-openspec/
├── ai-master-boot/              # 启动入口，配置装载，JSON序列化配置
├── ai-master-api/               # API 接口定义 (接口、DTO、Request、Response)
├── ai-master-service/           # 服务适配层 (RPC Provider、HTTP Controller、工厂转换)
├── ai-master-application/       # 应用层 (用例编排 ApplicationService、Action、Command、Result)
├── ai-master-domain/            # 领域层 (聚合根、实体、值对象、领域服务、仓储接口、领域事件)
├── ai-master-infrastructure/    # 基础设施层 (DAO 实现、Mapper、Entity、外部调用适配)
├── ai-master-client/            # 客户端 SDK (远程调用客户端，尚未实现)
├── ai-master-common/            # 通用工具类与基础组件 (暂无源文件，通用模型在 domain 中)
├── docs/                        # 知识库文档
└── openspec/                    # OpenSpec 配置 (AI Agent 相关)
```

### 模块依赖关系

```mermaid
graph TD
    boot["ai-master-boot<br/>启动入口"]
    service["ai-master-service<br/>服务适配层"]
    api["ai-master-api<br/>接口定义"]
    app["ai-master-application<br/>应用层"]
    domain["ai-master-domain<br/>领域层"]
    infra["ai-master-infrastructure<br/>基础设施层"]
    client["ai-master-client<br/>客户端SDK"]
    common["ai-master-common<br/>通用组件"]

    boot --> service
    boot --> infra
    service --> api
    service --> app
    client --> api
    app --> domain
    infra --> domain
    domain --> common
    api --> common
```

### 各层数据流转与对象映射

```
Request(api)  →  Command(application)  →  Domain Model(domain)  →  Entity(infrastructure)  →  DB
                   ↓                           ↓
              Result(application)        Domain Event(domain)
                   ↓
             Response(api/DTO)
```

| 转换步骤 | 方向 | 转换工具 | 位置 |
|----------|------|----------|------|
| Request → Command | 入参转换 | `BeanHelper.copyProperties` (Transformer) | `OrderCommandFactory` (service) |
| Command → Domain | 创建领域对象 | `Order.create()` 工厂方法 | `OrderDomainService` (domain) |
| Domain → Entity | 持久化 | `OrderFactory.instance()` 手动映射 | `OrderFactory` (infrastructure) |
| Entity → Domain | 查询还原 | `GoodsFactory.valueOf()` 手动映射 | `GoodsFactory` (infrastructure) |
| Domain → Result | 结果封装 | `OrderBuyResultFactory` (MapStruct) | `OrderBuyResultFactory` (application) |
| Result → Response/DTO | 输出转换 | `OrderBuyDTOFactory` (MapStruct) | `OrderResultFactory` (service) |

## 核心业务

### 功能模块清单

| 模块 | 上下文 | 路径 | 职责 | 核心类/文件 | 状态 |
|------|--------|------|------|------------|------|
| 交易服务 | Order | `ai-master-service` | 接收下单请求，参数校验，协议转换 | `OrderServiceProvider`, `OrderController` | ✅ 已实现 |
| 订单管理 | Order | `ai-master-domain/order` | 订单创建、状态流转、金额计算 | `Order`, `OrderDomainService`, `OrderStatus` | ✅ 已实现 |
| 库存管理 | Inventory | `ai-master-domain/inventory` | 商品库存锁定 | `Inventory`, `InventoryDomainService` | ✅ 已实现 |
| 商品查询 | Goods | `ai-master-domain/goods` | 商品信息查询与价格计算 | `Goods`, `Price`, `ItemQueryFacade` | ✅ 已实现 |
| 订单查询 | Order | `ai-master-domain/order/facade` | 订单详情查询 | `OrderQueryFacade` | ⬜ 仅占位 |
| 客户端SDK | Order | `ai-master-client` | 远程调用客户端 | `OrderClient` | ⬜ 未实现 |

### 关键业务流程

#### 下单流程 (完整调用链)

```mermaid
sequenceDiagram
    participant Client as 客户端
    participant Controller as OrderController<br/>POST /api/order/buy
    participant Provider as OrderServiceProvider<br/>RPC
    participant Factory as OrderCommandFactory
    participant AppSvc as OrderApplicationService
    participant CreateAction as OrderCreateAction
    participant ItemFacade as ItemQueryFacade
    participant OrderDomain as OrderDomainService
    participant LockAction as InventoryLockAction
    participant InvDomain as InventoryDomainService
    participant EnableAction as OrderEnableAction
    participant OrderRepo as OrderDao
    participant DB as MySQL

    Client ->> Controller: POST /api/order/buy (OrderBuyRequest)
    Note over Controller: 1. 参数校验 (BindingResult)
    Controller ->> Factory: OrderCommandFactory.asCommand(request)
    Factory -->> Controller: OrderBuyCommand
    Controller ->> AppSvc: doBuy(buyCommand)

    Note over AppSvc: === 应用编排开始 ===

    AppSvc ->> CreateAction: create(buy)
    CreateAction ->> ItemFacade: requireGoods(goodsId)
    Note over ItemFacade: @Cacheable 缓存
    ItemFacade -->> CreateAction: Goods
    CreateAction ->> OrderDomain: create(buyerId, goods, count)
    Note over OrderDomain: Order.create() 工厂方法<br/>计算金额 goods.calculateAmount()<br/>状态=NEW
    OrderDomain ->> OrderRepo: create(order)
    OrderRepo ->> DB: INSERT INTO order
    DB -->> OrderRepo: orderId
    OrderRepo -->> OrderDomain: order (含orderId)
    OrderDomain -->> CreateAction: Order

    AppSvc ->> LockAction: lock(order)
    LockAction ->> InvDomain: lock(goodsId, itemCount)
    Note over InvDomain: Inventory.createLock()
    InvDomain -->> LockAction: boolean (locked)

    alt 库存锁定成功
        AppSvc ->> EnableAction: enable(order)
        EnableAction ->> OrderDomain: enable(order)
        Note over OrderDomain: order.enable()<br/>OrderStatus: NEW → CREATED
        OrderDomain ->> OrderRepo: enable(order)
        OrderRepo ->> DB: UPDATE order SET status=CREATED
    end

    Note over AppSvc: === 应用编排结束 ===

    AppSvc -->> Controller: OrderBuyResult (MapStruct)
    Controller -->> Client: OrderBuyResponse
```

#### 调用链路总结

```
HTTP/RPC入口 → 参数校验 → DTO→Command转换
    → OrderApplicationService.doBuy() [用例编排]
        → OrderCreateAction.create()
            → ItemQueryFacade.requireGoods() [@Cacheable 商品查询]
            → OrderDomainService.create() [创建订单+持久化]
        → InventoryLockAction.lock()
            → InventoryDomainService.lock() [库存锁定]
        → OrderEnableAction.enable() [条件：锁定成功]
            → OrderDomainService.enable() [状态 NEW→CREATED + 持久化]
    → OrderBuyResultFactory.toResult() [MapStruct 结果映射]
→ OrderResultFactory.asResponse() [Result→Response转换]
```

### 领域模型

#### 领域对象全景

| 领域对象 | DDD 类型 | 所属上下文 | 职责 | 关键属性 | 代码位置 |
|----------|----------|-----------|------|----------|----------|
| `Order` | **聚合根** | Order | 订单一致性边界，含工厂方法与状态机 | orderId, goodsId, buyerId, sellerId, itemCount, amount, status | [Order.java](ai-master-domain/src/main/java/com/only/ai/master/order/domain/model/Order.java) |
| `OrderId` | 值对象 | Order | 订单唯一标识 | id (Long, >0) | [OrderId.java](ai-master-domain/src/main/java/com/only/ai/master/order/domain/model/OrderId.java) |
| `OrderAmount` | 值对象 | Order | 订单金额封装 | amount (MonetaryAmount) | [OrderAmount.java](ai-master-domain/src/main/java/com/only/ai/master/order/domain/model/OrderAmount.java) |
| `OrderStatus` | 枚举 | Order | 订单状态机 | NEW(0)→CREATED(1)→PAID(2)→... | [OrderStatus.java](ai-master-domain/src/main/java/com/only/ai/master/order/domain/model/OrderStatus.java) |
| `OrderCreatedEvent` | 领域事件 | Order | 订单创建事件（已定义，未使用） | 继承 OrderEvent | [OrderCreatedEvent.java](ai-master-domain/src/main/java/com/only/ai/master/order/domain/event/OrderCreatedEvent.java) |
| `Goods` | **聚合根** | Goods | 商品信息与金额计算 | goodsId, title, price, sellerId | [Goods.java](ai-master-domain/src/main/java/com/only/ai/master/goods/domain/model/Goods.java) |
| `GoodsId` | 值对象 | Goods | 商品唯一标识 | id (Long, >0) | [GoodsId.java](ai-master-domain/src/main/java/com/only/ai/master/goods/domain/model/GoodsId.java) |
| `Price` | 值对象 | Goods | 商品单价，含溢出检查 | amount (MonetaryAmount) | [Price.java](ai-master-domain/src/main/java/com/only/ai/master/goods/domain/model/Price.java) |
| `Inventory` | 值对象 | Inventory | 库存模型 (可用/占用/已售/锁定) | goodsId, available, locked, sold, lock | [Inventory.java](ai-master-domain/src/main/java/com/only/ai/master/inventory/domain/model/Inventory.java) |
| `MonetaryAmount` | 值对象 | Common | 不可变金额对象 (元/分双单位，支持折扣/加减) | monetaryAmount (BigDecimal), scale | [MonetaryAmount.java](ai-master-domain/src/main/java/com/only/ai/master/common/domain/MonetaryAmount.java) |
| `Id` | 基类 | Common | ID 抽象基类 (>0 校验) | id (Long) | [Id.java](ai-master-domain/src/main/java/com/only/ai/master/common/domain/Id.java) |
| `ValueObject<T>` | 接口 | Common | 值对象基接口 | `T value()` | [ValueObject.java](ai-master-domain/src/main/java/com/only/ai/master/common/domain/ValueObject.java) |
| `BuyerId` | 值对象 | User | 买家标识 | 继承 UserId | [BuyerId.java](ai-master-domain/src/main/java/com/only/ai/master/user/domain/model/BuyerId.java) |
| `SellerId` | 值对象 | User | 卖家标识 | 继承 UserId | [SellerId.java](ai-master-domain/src/main/java/com/only/ai/master/user/domain/model/SellerId.java) |
| `UserId` | 值对象 | User | 用户标识基类 | 继承 Id | [UserId.java](ai-master-domain/src/main/java/com/only/ai/master/user/domain/model/UserId.java) |

#### 对象关系图

```mermaid
erDiagram
    Order ||--|| OrderId : "has"
    Order ||--|| OrderAmount : "has"
    Order ||--|| OrderStatus : "has"
    Order ||--|| GoodsId : "references"
    Order ||--|| BuyerId : "belongs to"
    Order ||--|| SellerId : "belongs to"
    OrderAmount ||--|| MonetaryAmount : "wraps"
    Goods ||--|| GoodsId : "has"
    Goods ||--|| Price : "has"
    Goods ||--|| SellerId : "belongs to"
    Price ||--|| MonetaryAmount : "wraps"
    Inventory ||--|| GoodsId : "belongs to"
    BuyerId }|--|| UserId : "extends"
    SellerId }|--|| UserId : "extends"
    UserId }|--|| Id : "extends"
    OrderId }|--|| Id : "extends"
    GoodsId }|--|| Id : "extends"
```

#### 状态机 (Order Status)

```mermaid
stateDiagram-v2
    [*] --> NEW : Order.create()
    NEW --> CREATED : order.enable() / OrderStatus.create()
    CREATED --> PAID : (未实现)
    PAID --> CHECKED : (未实现)
    CREATED --> CANCELED : (未实现)
    PAID --> REFUND : (未实现)
    CANCELED --> CLOSED : (未实现)
    REFUND --> CLOSED : (未实现)

    note right of NEW : 初始化态 (status=0)
    note right of CREATED : 生效态 (status=1)
```

**已实现流转**: NEW(0) → CREATED(1)（仅当 `this == NEW` 才允许调用 `create()`，否则抛 `IllegalStateException`）

**已定义未实现**: PAID(2), CHECKED(3), REFUND(4), CANCELED(5), CLOSED(6), NONE(-1)

### 领域术语表

| 术语 | 中文 | 定义 | 代码表现 |
|------|------|------|----|
| Order | 订单 | 一次交易的核心聚合，封装买家、商品、金额、状态 | `Order` 聚合根 |
| Goods | 商品 | 可购买的餐饮商品，含标题和价格 | `Goods` 聚合根 |
| Inventory | 库存 | 商品可售数量管理（可用/占用/已售/锁定） | `Inventory` 值对象 |
| MonetaryAmount | 金额 | 不可变金额对象，精确到分，支持折扣与加减 | `MonetaryAmount` final 类 |
| Price | 价格 | 商品单价，支持总价计算（含溢出检查） | `Price` 值对象 |
| Inventory Lock | 库存锁定 | 下单时的预扣减操作，可用→占用转移 | `InventoryDomainService.lock()` |
| Enable | 订单生效 | 库存锁定成功后，订单从初始化态转为有效态 | `order.enable()` → `OrderStatus.create()` |
| Action | 业务动作 | 应用层的原子业务步骤 | `OrderCreateAction`, `InventoryLockAction`, `OrderEnableAction` |
| Command | 命令 | 应用层的输入对象 (CQS 模式的 C) | `OrderBuyCommand` |
| Result | 结果 | 应用层的输出对象 | `OrderBuyResult` |
| Facade | 查询门面 | 领域层的查询入口 | `ItemQueryFacade`, `OrderQueryFacade` |
| `@Call` | 调用追踪 | Transformer 提供的方法级日志与性能监控注解 | `@Call(elapsed=1200, sample=10000)` |

### 核心实体模型

| 领域对象 | 数据表名(推测) | 持久化实体 | 主要字段 | 关联关系 |
|----------|--------------|-----------|----------|----------|
| `Order` | `order` | `OrderEntity` | id, goodsId, buyerId, sellerId, amount(分), status(int) | 聚合根，引用 GoodsId |
| `Goods` | 外部服务获取 | `GoodsEntity` | id, title, price(分) | 通过 `GoodsCall` 远程获取 |
| `Inventory` | `inventory`(推测) | (暂无Entity) | goodsId, available, locked, sold, lock | 弱关联 GoodsId |

### 仓储与基础设施

| 仓储接口 (Domain) | 实现类 (Infrastructure) | 数据源 | 说明 |
|-------------------|------------------------|--------|------|
| `OrderRepository` | `OrderDao` (@Repository) | MySQL (OrderMapper) | 订单 CRUD，使用 `OrderFactory` 进行 Domain↔Entity 转换 |
| `GoodsRepository` | `GoodsDal` (@Repository) | 外部服务 (GoodsCall) | 商品查询，通过 RPC/接口调用获取，使用 `GoodsFactory` 还原领域对象 |
| `InventoryRepository` | (未发现实现类) | 推测 MySQL | 库存锁定操作 |

### 业务规则清单

| 规则ID | 描述 | 适用场景 | 代码实现位置 | 约束详情 |
|--------|------|----------|-------------|----------|
| BR-001 | 下单参数校验 | 下单入口 | [OrderBuyRequest.validator()](ai-master-api/src/main/java/com/only/ai/master/order/api/module/request/OrderBuyRequest.java) | buyerId、goodsId、itemCount 非空，itemCount > 0 |
| BR-002 | 初始订单状态 | 订单创建 | [Order.create()](ai-master-domain/src/main/java/com/only/ai/master/order/domain/model/Order.java) | 创建时状态默认 NEW(0) |
| BR-003 | 状态流转约束 | 订单生效 | [OrderStatus.create()](ai-master-domain/src/main/java/com/only/ai/master/order/domain/model/OrderStatus.java) | 仅 NEW 可流转为 CREATED，否则抛 IllegalStateException |
| BR-004 | 库存先锁后生效 | 下单编排 | [OrderApplicationService.doBuy()](ai-master-application/src/main/java/com/only/ai/master/order/application/service/OrderApplicationService.java) | 库存锁定成功 → 订单生效；失败 → 订单保持 NEW |
| BR-005 | 金额计算 | 订单创建 | [Goods.calculateAmount()](ai-master-domain/src/main/java/com/only/ai/master/goods/domain/model/Goods.java) → [Price.calculateAmount()](ai-master-domain/src/main/java/com/only/ai/master/goods/domain/model/Price.java) | 总价 = 单价(分) × 数量，含 Long 溢出检查 |
| BR-006 | ID 合法性 | 全局 | [Id (构造函数)](ai-master-domain/src/main/java/com/only/ai/master/common/domain/Id.java) | id 非空且 > 0 |
| BR-007 | 金额合法性 | 全局 | [MonetaryAmount.create()](ai-master-domain/src/main/java/com/only/ai/master/common/domain/MonetaryAmount.java) | amount/cent ≥ 0, scale ≥ 0 |
| BR-008 | 订单生效持久化 | 订单状态更新 | [OrderDao.enable()](ai-master-infrastructure/src/main/java/com/only/ai/master/order/infrastructure/dao/OrderDao.java) | 数据库更新失败 (count=0) 时抛 IllegalStateException |

## 开发规范

> 完整规范体系位于 `~/ai/rules/` 目录，以下为各领域核心要点摘录。

### 需求规范

> 详见 `~/ai/rules/requirement/requirement-template.md`

- **文档定位**: 产品说明书，而非需求说明文档
- **表达模式**: SCQA (Situation-Complication-Question-Answer)
- **组织形式**: 产品 → 模块 → 菜单 → 功能 → 操作
- **版本记录**: 每次变动带版本号 (`N` 新建 / `A` 增加 / `M` 更改 / `D` 删除)
- **目标定义**: OKR 格式，指标满足 SMART 原则
- **功能清单**: 包含使用场景、前提条件、处理规则、后置处理
- **非功能需求**: 覆盖权限、性能、安全、兼容性

### 设计规范

> 详见 `~/ai/rules/design/design-guidelines.md` 和 `~/ai/rules/design/architecture-template.md`

- **架构风格**: DDD 六边形架构 + 整洁架构 + CQRS
- **设计原则**: SOLID / KISS / DRY / YAGNI
- **DDD 实践**:
  - 统一语言：业务与技术共享同一术语体系
  - 限界上下文：明确业务边界，避免概念混淆
  - 聚合根：以聚合为中心设计业务一致性边界
  - 领域事件：事件驱动实现业务解耦
- **概要设计结构**: 业务分析 (名词定义 + 流程 + 领域模型) → 概要设计 (系统架构 + 能力定义 + 技术选型/VALET) → 详设映射
- **文档图表**: 统一使用 Mermaid 绘制 (C4 架构图、ER 图、时序图、状态图、流程图)
- **设计评审**: 完整性 (正确/完备/业务闭环) + 健壮性 (安全/容错/稳定/一致) + 质量属性 (可扩展/可维护/可测试/可部署)

### 代码规范

> 详见 `~/ai/rules/coding/java-guidelines.md`

- **命名约定**:
  - 类名 PascalCase / 方法名 camelCase / 常量名 UPPER_SNAKE_CASE
  - 包名全小写、域名反写: `com.only.ai.master.{context}.{layer}`
- **异常分层**: `BusinessException` (业务) / `SystemException` (系统) / `ValidationException` (参数) / `DataNotFoundException` (数据)
- **日志规范**: ERROR (系统异常) → WARN (业务异常) → INFO (关键路径) → DEBUG (调试流程)
- **JavaDoc**: 所有公共 API 必须包含 `@param` / `@return` / `@throws` 注释
- **领域模型**: 行为方法内聚于聚合根，状态变更通过领域行为方法触发

### 工程结构规范

> 详见 `~/ai/rules/coding/project-structure.md`

- **分层架构**: DDD 六边形分层，依赖方向单向向下
- **分层约束**:

| 层级 | 禁止事项 |
|------|---------|
| API 层 | 禁止直接暴露领域对象 |
| Service 层 | 不含业务逻辑，仅协议适配 |
| Application 层 | 只接收 Command/Query，只返回 Result，不直接操作领域对象内部状态 |
| Domain 层 | 不依赖任何技术框架 (除 Spring 注解)，纯业务逻辑 |
| Infrastructure 层 | 实现 Domain 层定义的仓储接口 |

- **依赖注入**: 领域层构造函数注入，应用层 `@Autowired` 字段注入
- **对象转换**: 使用 MapStruct，转换逻辑统一放在 Factory 类中
- **Maven 规范** (`~/ai/rules/coding/maven-guidelines.md`): 最小依赖原则 / 父 POM 统一版本管理 / 语义化版本号 / 零编译警告

### Git 工作流

> 详见 `~/ai/rules/coding/git-guidelines.md`

- **提交格式**: Conventional Commits — `<type>(<scope>): <subject>`
- **提交类型**:

| 类型 | 场景 |
|------|------|
| `feature` | 新增功能 |
| `fix` | 修复 Bug |
| `docs` | 文档注释 |
| `style` | 代码格式 (不影响运行) |
| `refactor` | 重构优化 |
| `performance` | 性能优化 |
| `test` | 增加测试 |
| `chore` | 构建工具、依赖管理 |
| `revert` | 回退代码 |

- **提交原则**: 原子性 (一个逻辑变更) / 完整性 (代码+测试) / 可回滚 / 可追溯 (关联需求编号)
- **大功能拆分**: 功能分支 `feature/{name}` → 多次提交 → `--squash` 合并
- **提交前检查**: 通过单元测试 + 编码规范 + 格式正确 + 关联需求

### 测试规范

> 详见 `~/ai/rules/testing/testing-guidelines.md`

- **质量门禁**:

| 指标 | 阈值 | 工具 |
|------|------|------|
| 单元测试覆盖率 | ≥80% | Jacoco |
| 代码重复率 | ≤5% | PMD/CPD |
| 严重漏洞 | 0 | SonarQube |
| 编译警告 | 0 | Maven/Compiler |

- **测试原则**: FIRST (Fast / Independent / Repeatable / Self-validating / Timely) + AAA 模式 (Arrange-Act-Assert) + 单一断言
- **命名规范**: 类 `{ClassName}Test` / 方法 `should{ExpectedBehavior}When{Condition}`
- **分层策略**:

| 层级 | 覆盖范围 | Mock 策略 |
|------|---------|----------|
| 单元测试 (P0) | 领域模型、值对象、领域服务 | 不 Mock 领域对象；Mock 仓储接口 |
| 集成测试 (P1) | 应用服务、仓储实现 | Mock 外部服务，使用 H2 内存库 |
| E2E 测试 (P2) | HTTP 接口完整链路 | Mock 外部服务，使用 H2 内存库 |

- **测试框架**: JUnit 5 + Mockito + AssertJ + Spring Boot Test (均 Spring Boot 内置)

### 命名规范 (基于项目实际)

| 层级 | 组件类型 | 命名模式 | 示例 |
|------|----------|----------|------|
| API | 服务接口 | `{Aggregate}Service` | `OrderService` |
| API | 请求类 | `{Aggregate}{Action}Request` | `OrderBuyRequest` |
| API | 响应类 | `{Aggregate}{Action}Response` | `OrderBuyResponse` |
| API | DTO | `{Aggregate}{Action}DTO` | `OrderBuyDTO` |
| Service | RPC实现 | `{Aggregate}ServiceProvider` | `OrderServiceProvider` |
| Service | HTTP控制器 | `{Aggregate}Controller` | `OrderController` |
| Service | 转换工厂 | `{Aggregate}CommandFactory` / `{Aggregate}ResultFactory` | `OrderCommandFactory` |
| Application | 应用服务 | `{Aggregate}ApplicationService` | `OrderApplicationService` |
| Application | 业务动作 | `{Aggregate}{Action}Action` | `OrderCreateAction`, `InventoryLockAction` |
| Application | 命令 | `{Aggregate}{Action}Command` | `OrderBuyCommand` |
| Application | 结果 | `{Aggregate}{Action}Result` | `OrderBuyResult` |
| Application | 结果工厂 | `{Aggregate}{Action}ResultFactory` (MapStruct) | `OrderBuyResultFactory` |
| Domain | 聚合根 | `{Aggregate}` | `Order`, `Goods` |
| Domain | 值对象 | `{ValueObject}` | `OrderId`, `Price`, `MonetaryAmount` |
| Domain | 领域服务 | `{Aggregate}DomainService` | `OrderDomainService` |
| Domain | 查询门面 | `{Entity}QueryFacade` | `ItemQueryFacade`, `OrderQueryFacade` |
| Domain | 仓储接口 | `{Aggregate}Repository` | `OrderRepository` |
| Domain | 领域事件 | `{Aggregate}{Past}Event` | `OrderCreatedEvent` |
| Infra | DAO | `{Aggregate}Dao` / `{Aggregate}Dal` | `OrderDao`, `GoodsDal` |
| Infra | 数据实体 | `{Aggregate}Entity` | `OrderEntity`, `GoodsEntity` |
| Infra | Mapper | `{Aggregate}Mapper` | `OrderMapper` |
| Infra | 工厂 | `{Aggregate}Factory` | `OrderFactory`, `GoodsFactory` |

## 约定与约束

### 关键约束
- [x] 使用 Java 17 特性
- [x] JSON 序列化使用 SNAKE_CASE 命名策略
- [x] 忽略未知 JSON 字段 (`FAIL_ON_UNKNOWN_PROPERTIES=false`)
- [x] 空值不序列化 (`NON_NULL`)
- [x] 日期格式 `yyyy-MM-dd HH:mm:ss`
- [x] 金额单位为「分」(Long)，通过 `MonetaryAmount` 封装元/分转换
- [ ] 数据库驱动版本较低 (5.1.30)，需注意兼容性

### 接入层约定
- RPC 接口与 HTTP 接口双入口，共享同一应用服务
- RPC 端使用 `OrderBuyRequest.validator()` 方法校验
- HTTP 端使用 `@Valid` + `BindingResult` Bean Validation 校验
- 统一使用 `@Call(elapsed=1200, sample=10000)` 监控

### 文档同步规则
**自动更新触发条件:**
- 新增或修改 API 接口
- 数据模型变更（领域对象或数据库表）
- 业务规则发现或变更
- 重大架构调整

**更新要求:**
- 更新 AGENTS.md 相关内容和"最后更新"时间戳
- 同步更新 `docs/` 下相关文档
- 在"会话历史摘要"中记录变更

## 技术债务登记

| ID | 描述 | 位置 | 优先级 | 建议方案 |
|----|------|------|--------|----------|
| TD-001 | MySQL 驱动版本过旧 (5.1.30) | `ai-master-infrastructure/pom.xml` | P2 | 升级到 mysql-connector-j 8.x |
| TD-002 | `OrderEnableAction` 缺少 `@Component` 注解 | [OrderEnableAction.java](ai-master-application/src/main/java/com/only/ai/master/order/application/action/OrderEnableAction.java) | **P0** | 添加 `@Component` 注解，否则 Spring 无法注入 |
| TD-003 | `InventoryLockAction` 注释错误 ("优惠计算") | [InventoryLockAction.java](ai-master-application/src/main/java/com/only/ai/master/order/application/action/InventoryLockAction.java) | P3 | 修正注释为"库存锁定" |
| TD-004 | `OrderQueryFacade.requireOrder()` 返回 null (硬编码) | [OrderQueryFacade.java](ai-master-domain/src/main/java/com/only/ai/master/order/domain/facade/OrderQueryFacade.java) | P1 | 实现实际查询逻辑或移除占位代码 |
| TD-005 | `OrderClient.buy()` 抛 `UnsupportedOperationException` | [OrderClient.java](ai-master-client/src/main/java/com/only/ai/master/order/client/OrderClient.java) | P2 | 实现 RPC/HTTP 远程调用逻辑 |
| TD-006 | `InventoryRepository` 缺少基础设施层实现 | `ai-master-domain/inventory/domain/repository/` | P1 | 创建 `InventoryDal`/`InventoryDao` 实现类 |
| TD-007 | `OrderCreatedEvent` 已定义但未被使用 | `ai-master-domain/order/domain/event/` | P3 | 发布事件到 MQ 或 Spring Event |
| TD-008 | 测试覆盖率 0% | 全局 | **P0** | 按 `docs/tdd.md` 添加领域层单元测试 |
| TD-009 | `OrderBuyRequest.validator()` 返回值语义反转 | [OrderBuyRequest.java](ai-master-api/src/main/java/com/only/ai/master/order/api/module/request/OrderBuyRequest.java) | P1 | `validator()` 在参数无效时返回 `true`，语义易混淆 |
| TD-010 | `OrderServiceProvider` 缺少 `@Component`/`@Service` 注解 | [OrderServiceProvider.java](ai-master-service/src/main/java/com/only/ai/master/order/service/rpc/OrderServiceProvider.java) | P1 | 可能通过 Dubbo XML 配置注册，需确认 |
| TD-011 | `OrderBuyDTO` 为空类，无任何字段 | [OrderBuyDTO.java](ai-master-api/src/main/java/com/only/ai/master/order/api/module/dto/OrderBuyDTO.java) | P2 | 添加返回给客户端的订单信息字段 |

## 已知风险

| 风险 | 影响范围 | 严重度 | 规避建议 |
|------|----------|--------|----------|
| 订单创建与库存锁定缺少分布式事务保障 | 下单核心链路 | 高 | 引入本地消息表或 Saga 模式保证最终一致性 |
| `OrderEnableAction` 缺少 `@Component` 注解 (TD-002) | 下单流程运行时 | 高 | 启动后立即验证注入，或添加集成测试覆盖 |
| 测试覆盖率 0%，无任何自动化质量保障 (TD-008) | 全局代码质量 | 高 | 优先按 `docs/tdd.md` 补充 P0 领域层单元测试 |
| MySQL 驱动版本过旧 (5.1.30) | 数据库连接安全性与兼容性 | 中 | 升级到 mysql-connector-j 8.x，注意时区参数变更 |
| `InventoryRepository` 无基础设施层实现 (TD-006) | 库存锁定功能完整性 | 中 | 创建 `InventoryDao` 实现类，或确认是否有外部服务承接 |
| `OrderBuyRequest.validator()` 返回值语义反转 (TD-009) | RPC 入口参数校验 | 中 | 重构为 `isInvalid()` 或反转返回值语义 |
| 商品数据依赖外部服务 (`GoodsCall`) | 下单流程可用性 | 中 | 增加 `@Cacheable` 降级策略和超时熔断 |
| 单库单表架构，订单表无分库分表方案 | 业务增长后的数据库性能 | 低 | 预留分库分表扩展点，关注订单表数据增长 |

## 工作约定

### 会话开始时
1. 阅读本 `AGENTS.md` 了解项目全貌
2. 根据任务类型查阅 `docs/` 下相关文档:
   - 业务需求相关 → `docs/prd.md`
   - 架构设计相关 → `docs/add.md`
   - 测试实现相关 → `docs/tdd.md`
   - API 接口相关 → `docs/api/README.md`
3. 确认当前工作上下文和涉及的领域边界 (Order / Goods / Inventory)

### 会话进行中
- 遇到不明确的业务规则，记录到待确认清单
- 发现技术债务，登记到上方技术债务表
- 代码修改遵循 DDD 分层约束，不跨层直接依赖
- 重大修改必须遵循 SDD (Spec-Driven Development) 原则：先更新规范文档，再实现代码
- 新增 API 接口时同步更新 `docs/api/README.md`
- 领域模型变更时同步更新 `docs/add.md` 中的领域对象表和对象关系图

### 会话结束时
- 提炼发现的新业务规则，提示确认后更新到 `docs/` 相应文件
- 提炼发现的开发规范、约束，提示确认后更新到 `AGENTS.md`
- 更新本文件的会话历史摘要
- 标注未完成事项和后续建议

## 会话历史摘要

### 2026-02-24 - v3.0 结构化改进
- 标题从"系统知识库"更名为"AI Agent 指南"，对齐标准模板
- 开发规范章节从纯链接引用改为内联要点摘录 (需求/设计/代码/工程结构/Git/测试六大维度)
- 新增「已知风险」章节，登记 8 项已识别风险及规避建议
- 新增「工作约定」章节，定义会话开始/进行中/结束时的协作协议
- 项目概述补充「启动年份（推测）」字段
- 代码规模格式化为 KLOC 标准表示

### 2026-02-12 - v2.0 增强版 (全面逆向分析)
- 阅读全部 54 个 Java 源文件，建立完整代码理解
- 绘制完整调用链序列图和领域模型 ER 图
- 建立详细的领域对象全景表 (14个对象)
- 梳理 8 条业务规则并关联代码位置
- 发现并登记 11 项技术债务 (含 2 个 P0 级别潜在 Bug)
- 整理完整的命名规范对照表 (覆盖全部 7 层)
- 补充数据流转与对象映射关系

### 2026-02-11 - v1.0 初始化
- 创建知识库结构
- 识别技术栈: Java 17, Spring Boot 2.7.10
- 识别核心模块: boot, api, infrastructure, domain, etc.
