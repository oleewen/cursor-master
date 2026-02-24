# AI Master Restaurant Order

> 基于 Spring Boot 与 DDD 架构构建的餐厅点餐交易引擎，实现从用户下单、库存锁定到订单全生命周期管理的核心后端服务。

## 目录

- [技术栈](#技术栈)
- [快速开始](#快速开始)
- [工程结构](#工程结构)
- [模块说明](#模块说明)
- [API 参考](#api-参考)
- [测试](#测试)
- [文档](#文档)
- [贡献指南](#贡献指南)
- [许可证](#许可证)

## 技术栈

| 类别 | 技术 | 版本 |
|------|------|------|
| 语言 | Java | 17 |
| 框架 | Spring Boot | 2.7.10 |
| ORM | MyBatis | 2.3.2 (starter) |
| 数据库 | MySQL | 5.7+ |
| 连接池 | Druid | 1.2.23 |
| 对象映射 | MapStruct | 1.5.0.Final |
| API 文档 | Swagger | 2.7.0 |
| 构建 | Maven | 3.8+ |
| 内部框架 | Transformer | 1.2.10 |

## 快速开始

### 环境要求

- JDK 17+
- Maven 3.8+
- MySQL 5.7+

### 克隆与构建

```bash
git clone <repository-url>
cd ai-master-restaurant-order-openspec
```

### 配置数据库

修改 `ai-master-boot/src/main/resources/application.properties`，添加数据库连接信息：

```properties
spring.datasource.url=jdbc:mysql://localhost:3306/ai_master?useUnicode=true&characterEncoding=utf-8
spring.datasource.username=root
spring.datasource.password=your_password
spring.datasource.driver-class-name=com.mysql.jdbc.Driver
```

### 编译打包

```bash
mvn clean package -DskipTests
```

### 启动服务

```bash
java -jar ai-master-boot/target/ai-master-boot.jar
```

服务默认端口 8080，管理端点端口 8888。

### 验证

```bash
# 健康检查
curl http://127.0.0.1:8888/actuator/health

# 下单测试
curl -X POST http://localhost:8080/api/order/buy \
  -H "Content-Type: application/json" \
  -d '{"buyer_id":100,"goods_id":1,"item_count":2}'
```

## 工程结构

```
ai-master-restaurant-order-openspec/
├── ai-master-boot/                # 启动入口，配置装载，Jackson 序列化配置
├── ai-master-api/                 # API 接口定义 (接口、DTO、Request、Response)
├── ai-master-service/             # 服务适配层 (HTTP Controller、RPC Provider、工厂转换)
├── ai-master-application/         # 应用层 (用例编排、Action、Command、Result)
├── ai-master-domain/              # 领域层 (聚合根、值对象、领域服务、仓储接口)
├── ai-master-infrastructure/      # 基础设施层 (DAO 实现、Mapper、Entity、外部调用)
├── ai-master-client/              # 客户端 SDK (远程调用客户端)
├── ai-master-common/              # 通用组件 (基础类型、工具)
├── docs/                          # 项目文档
│   ├── prd.md                     #   产品需求说明书
│   ├── add.md                     #   架构设计说明书
│   ├── tdd.md                     #   测试设计说明书
│   └── api/                       #   API 接口文档
└── pom.xml                        # Maven 父 POM
```

## 模块说明

项目采用 DDD 六边形架构，依赖方向严格自外向内：

```
boot → service → application → domain ← infrastructure
                    ↘ api ↙               ↙
                     common ← ← ← ← ← ←
```

| 模块 | 层级 | 职责 | 核心类 |
|------|------|------|--------|
| `ai-master-boot` | 启动层 | 应用启动，Jackson 配置 (SNAKE_CASE) | `ApplicationStarter` |
| `ai-master-api` | 接口层 | 服务契约定义，不含实现逻辑 | `OrderService`, `OrderBuyRequest`, `OrderBuyResponse` |
| `ai-master-service` | 适配层 | HTTP/RPC 入口，参数校验，DTO 转换 | `OrderController`, `OrderServiceProvider` |
| `ai-master-application` | 应用层 | 用例编排，协调 Action 完成业务流程 | `OrderApplicationService`, `OrderCreateAction` |
| `ai-master-domain` | 领域层 | 核心业务逻辑，聚合根，领域服务 | `Order`, `Goods`, `Inventory`, `MonetaryAmount` |
| `ai-master-infrastructure` | 基础设施层 | 仓储实现，数据库访问，外部服务适配 | `OrderDao`, `GoodsDal`, `OrderMapper` |
| `ai-master-client` | 客户端 | 远程调用 SDK | `OrderClient` |
| `ai-master-common` | 通用层 | 基础类型，值对象接口 | `Id`, `ValueObject` |

## API 参考

### 下单接口

**HTTP**

```
POST /api/order/buy
Content-Type: application/json
```

请求体：

```json
{
  "buyer_id": 100,
  "goods_id": 1,
  "item_count": 2
}
```

| 字段 | 类型 | 必填 | 说明 |
|------|------|------|------|
| `buyer_id` | Long | 是 | 买家 ID |
| `goods_id` | Long | 是 | 商品 ID |
| `item_count` | Integer | 是 | 购买数量 (>0) |

响应体：

```json
{
  "success": true,
  "module": { }
}
```

**RPC**

```java
OrderBuyResponse response = orderService.buy(orderBuyRequest);
```

完整 API 文档参见 [docs/api/README.md](docs/api/README.md)。

## 测试

当前测试覆盖率为 0%，测试用例设计已完成（46 条），详见 [docs/tdd.md](docs/tdd.md)。

```bash
# 单元测试 (领域层)
mvn test -Dtest=*Test -pl ai-master-domain

# 全量测试
mvn test

# 覆盖率报告
mvn test jacoco:report
```

## 文档

| 文档 | 路径 | 说明 |
|------|------|------|
| 项目知识库 | [AGENTS.md](AGENTS.md) | 项目全貌、开发规范、技术债务、工作约定 |
| 产品需求 | [docs/prd.md](docs/prd.md) | 业务需求、用户故事、状态机、库存模型 |
| 架构设计 | [docs/add.md](docs/add.md) | C4 架构图、分层设计、领域模型、ER 图 |
| 测试设计 | [docs/tdd.md](docs/tdd.md) | 测试策略、用例清单、Mock 策略、数据集 |
| API 文档 | [docs/api/](docs/api/) | 接口定义与协议说明 |

## 贡献指南

1. Fork 本仓库
2. 创建特性分支 (`git checkout -b feature/xxx`)
3. 提交变更，遵循 [Conventional Commits](https://www.conventionalcommits.org/) 规范
4. 推送分支 (`git push origin feature/xxx`)
5. 创建 Pull Request

提交格式：`<type>(<scope>): <subject>`

| 类型 | 说明 |
|------|------|
| `feat` | 新功能 |
| `fix` | 缺陷修复 |
| `refactor` | 重构 |
| `docs` | 文档更新 |
| `test` | 测试用例 |
| `chore` | 构建/工具变更 |

## 许可证

本项目仅供学习与参考使用。
