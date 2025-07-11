+++
date = '2025-05-26T20:48:12+08:00'
draft = false
title = 'DDD重构老架构后端代码'
+++
# DDD重构老架构后端代码

## 引言: 遗留系统

### 遗留系统的一些明显特征
* 代码质量一言难尽
* 架构混乱，模块之间职责不明，改一个简单的功能也会改到几个服务
* 缺乏单元测试，集成测试
* 老旧的技术和工具
* 上手难度高，业务知识难以获取
* 缺乏CI/CD

### 为什么对遗留系统现代化？
#### 业务价值层面
* 遗留系统蕴含了大量的数据资产
* 遗留系统中还藏匿着丰富的业务知识
* 仍然有大量用户在使用，为公司带来收益

#### 业务交付层面
* 缩短需求交付周期
* 增加需求交付质量
* 减少系统故障的概率
* 减少线上工单数量

### 遗留系统现代化的技术建设
* 代码现代化，对遗留系统的代码进行安全的可测试化重构
* 架构现代化，微服务架构或云原生架构，通常和代码现代化并行进行。对遗留系统内部改造的同时，将内部模块拆分到外面，或者将新需求实现到遗留系统外部
* DevOps现代化以应对频繁的迭代/hotfix
* 监控告警现代化，增加系统的可观测性
* 对老旧的技术/工具进行升级


## DDD
### 什么是DDD(领域驱动设计)
DDD 是一种架构设计方法论，的核心思想是通过领域驱动设计方法定义领域模型，从而确定业务和应用边界，保证业务模型与代码模型的一致性。

### DDD为什么适合微服务
它可以帮我我们设计出清晰的领域和应用边界， 更容易实现业务和架构上的演进， 当然DDD实际上也被广泛适用于单体应用

### DDD的战术设计和战略设计

#### DDD的战略设计
战略设计主要从业务视角出发，建立业务领域模型，划分领域边界。使用限界上下文确定业务边界

#### DDD的战术设计
战术设计则从技术视角出发，侧重于领域模型的技术实现，完成软件开发和落地，包括：聚合根、实体、值对象、领域服务、应用服务和资源库等代码逻辑的设计和实现。

### DDD核心概念
* 领域：限定业务范围
* 子领域：领域可以划分成为多个子领域，每个子域对应一个更小的问题域或更小的业务范围
* 核心域：公司核心竞争力的子域
* 通用域：多个子域使用的通用功能子域是通用域
* 支撑域：为核心业务提供辅助功能的子域
* 事件风暴：事件风暴是一项团队活动，领域专家与项目团队通过头脑风暴的形式，罗列出领域中所有的领域事件，整合之后形成最终的领域事件集合，然后对每一个事件，标注出导致该事件的命令，再为每一个事件标注出命令发起方的角色。命令可以是用户发起，也可以是第三方系统调用或者定时器触发等，最后对事件进行分类，整理出实体、聚合、聚合根以及限界上下文。
* 限界上下文：保证在领域之内的一些术语、业务相关对象有一个确切的含义，没有二义性。确保通用语言语义的唯一性
* 实体：拥有唯一标识符，且标识符在历经各种状态变更后仍能保持一致, 比如订单，商品
* 值对象：值对象是描述事物的状态或属性的对象，它没有唯一标识，并且通常是不可变的，值对象上不会挂复杂业务逻辑
* 聚合：而能让实体和值对象协同工作的组织就是聚合
* 聚合根：聚合不仅是实体，也是聚合的入口点和管理者
* 仓储：又称repo， 一个聚合对应一个repo, 你可以理解为实现了领域对象到数据库对象转换的dao层
* 领域服务：当业务逻辑在单一聚合无法实现的时候需要，组合多个聚合实现业务逻辑，一般来说一个领域服务对应一个聚合即可
* 领域事件：领域模型中发生的事件，用于减少领域之间的耦合。
* 防腐层(ACL): 使用防腐层来隔离新旧系统，做一些协议/命名的转换，防止遗留系统的schema变更影响到新的领域服务



### 毛菜DDD战略设计
按战略设计思想基本划分毛菜的核心，通用，支撑域

![老架构领域上下文](/assets/ddd_domain.png "ddd domain")

毛菜的核心域是订单，进销存，配送，采购。所以需要重构重点聚焦于红色的核心领域代码质量很差的部分，比如分拣, 采购。


### DDD战术设计
#### DDD分层架构

以下的代码分层为一个限界上下文（微服务）的分层
![ddd分层](/assets/ddd_layer.png "ddd layer")

* 用户接口层: webApi, 单元测试，前端模版的渲染
* 应用层: 微服务，领域服务的编排，协调，认证，权限校验，事务控制，领域事件发布订阅，这一层越薄越好。
* 领域层: 包含包含聚合根、实体、值对象、领域服务
* 基础层: 数据库，事件总线，缓存，等第三方工具的具体实现

领域服务只能被应用服务调用，而应用服务只能被用户接口层调用，服务是逐层对外封装或组合的

#### 老架构DDD工程分包结构
```
project/
│
├── src/
│   ├── facade/
│   │   ├── web 
│   │   │   ├── controller/
│   │   │   ├── dto/
│   │   │   ├── converter/
│   │   │   └── adapter/
│   │   ├── tools/
│   │   └── asynctask/
│   │
│   ├── application/
│   │   ├── event/  
│   │   │   ├── publish/
│   │   │   └── subscribe/
│   │   ├── service/   
│   │   └── client/
│   │       ├── acl
│   │       └── dto
│   │
│   ├──domain/
│   │   ├── aggregate01/
│   │   │   ├── event/
│   │   │   ├── entity/
│   │   │   ├── repository/
│   │   │   ├── exception/
│   │   │   └── service/
│   │   │       └── vo/
│   │   └── aggregate02/
│   │
│   ├── infrastructure/
│   │    ├── repo/
│   │    │   ├── aggregate01/
│   │    │   │   └── po.py
│   │    │   └── aggregate02/
│   │    ├── cache/
│   │    ├── persistent/
│   │    ├── eventbus/
│   │    ├── log/
│   │    ├── web/
│   │    ├── client/
│   │    │   ├── grpc
│   │    │   └── http
│   │    ├── utils/
│   │    └── config/
│   │
│   └── main/
│   
├── test/
│   │
│   ├── unit/
│   │   ├── application/
│   │   │   ├── service/
│   │   │   │   └── test_xxxx.py
│   │   │   └── asynctask/
│   │   │        └── test_xxxx.py
│   │   │
│   │   └── domain/
│   │       ├── aggregate01/
│   │       │   └── test_aggregate01_xxxx.py
│   │       └── aggregate02/
│   │
│   └── intergration
│       ├── infrastructure/
│       ├── repo/
│       │   └── aggregate01/
│       │       └── test_aggregate01.py
│       └── eventbus/
│           └── test_pub_sub.py
│
└── .gitlab-ci.yaml
```

#### DDD设计下微服务的调用
![ddd分层](/assets/ddd_service_call.png "service call")



## 老架构重构策略

### 增量演进
通过增量演进，新旧系统并存的方式逐渐改掉遗留服务，通过路由级别的灰度发布验证新系统，针对特定接口配置路由规则。


### 如何拆分数据？数据归属问题？
暂时不要拆分，先共享数据库。

我们的系统里面到处都是所有模块都可以随意访问任意的表，操作这些数据的业务逻辑散落在各个服务中，在重构出清晰的领域模型之前，很难找出它真正属于谁，我们需要先以修缮者模式重构代码，再去拆分逻辑/物理数据库

有以下几种方式去在新拆分的领域服务里面去读取遗留系统的数据库：
* 通过ACL防腐层实现repo直接读取
![ddd分层](/assets/acl_01.png "acl")

* 通过订阅数据库CDC(目前mongodb低版本不支持),事件总线的方式去把遗留系统的数据持久化在自己的领域, 当然也需要ACL来实现

* 通过事件总线的方式订阅遗留系统数据
![ddd分层](/assets/acl_03.png "acl")

* 通过调用遗留系统API，ACL转换协议和schema
![ddd分层](/assets/acl_02.png "acl")

## 重构+新需求实战：采购损耗差异
以采购损耗差异这个需求看看开发流程是如何的，具体代码该怎么写，这个需求是一个比较好的例子，它会涉及到对老系统领域模型的梳理。

PRD文档: https://www.yuque.com/guanmai/vb2acm/vkgx5wwffws75rwv?singleDoc#

### 一. 事件风暴
标准参与者：研发，测试，产品, 业务人员

我们需要重点关注这类业务的语言和行为。比如某些业务动作或行为（事件）是否会触发下一个业务动作，这个动作（事件）的输入和输出是什么？是谁（实体）发出的什么动作（命令），触发了这个动作（事件）



事件风暴的要素：

![ddd分层](/assets/ddd_event_storming.png "ddd_event_storming")

进销存简化版事件风暴：

![ddd分层](/assets/ddd_event_storming_maocai.png "ddd_event_storming")

要重构哪个模块，就用事件风暴把哪个模块的事件，实体，命令理清楚

### 二. 技术设计，建立领域模型


进销存领域库存管理上下文入库单聚合
```mermaid
classDiagram
    
    class InStockSheet {
      +String ID # 聚合根id
      +ValueObject Supplier # 供应商值对象
      +String TotalAmount # 总金额
      +String TotalLossAmount # 总损耗金额
      +ValueObject Status # 入库单状态
      +String Remark
      +List[InStockSheetDetail] Details # 采购单项实体
      +List[InStockSheetDetail] InitDetails # 初始采购单项实体
      +ValueObject Purchaser # 采购员值对象
      +ValueObject User # 用户值对象
      +ValueObject PurchaseSheet # 采购单值对象
      +DateTime CommitTime
      +DateTime CreateTime
      +DateTime UpdateTime 
      +DateTime Deletetime
      +String StationID
      +String GroupID
      +Int Version

      +Create(CreateInStockSheetDo) InStockSheetCreatedEvent
      +Delete(DeleteInStockSheetDo) InStockSheetDeletedEvent
      +Update(UpdateInStockSheetDo) InStockSheetUpdatedEvent
      +Submit() InStockSheetSubmitedEvent
      +GetDetails()
      }

    class InStockSheetDetail {
      +String ID
      +ValueObject Sku # 商品值对象
      +ValueObject PurchaseSku # 采购sku值对象
      +String Quantity # 数量
      +String LossQuantity # 损耗数量
      +String UnitPrice # 单价
    }
    InStockSheetDetail "1" -- "n" InStockSheet
    Supplier "1" -- "n" InStockSheet
    Purchaser "1" -- "n" InStockSheet
```

```mermaid

```


其他出库单聚合
```mermaid
classDiagram
     class OtherOutStockSheet {
        + String ID
        + ValueObject Type
        + String BatchNumber
        + List[OtherOutStockSheetDetail] Details # 商品项实体 
     }
     class OtherOutStockSheetDetail {

     }

     OtherOutStockSheetDetail "n" -- "1" OtherOutStockSheet
```


供应商上下文供应商聚合
```mermaid
classDiagram
    class Supplier {
        +String ID # 聚合根id
        +String No
        +String Address
        +ValueObject Station # station值对象
        +Object BizInfo #业务信息实体
        +Bool IsAutoSyncPuchaseSheet
        +List[SupplyCategory] SupplyCategorys # 可供分类实体
        +List[String] # 可供分类Sku
        +List[Contract] # 合同实体


        +Create(CreateSupplierDo) SupplierCreatedEvent
        +AddContract(AddContractDo) ContractAddedEvent
        +AddSupplySku(String) SupplySkuAddedEvent
    }
    
```


### 三. 需求拆分任务

#### 任务的分解法


#### 1. 拆分用户故事
#### 2. 获取验收用例
#### 3. 完成了需求到用户故事到验收条件


参考： 软件工程的工序分解与实践 https://www.yuque.com/guanmai/hybrcb/hr8u58e47sem9d7x

### 四. 先根据拆分的任务写单元测试，再写代码

### 五. 写完代码后，自行review代码读出坏味道，重构代码

### 五. 根据时间充裕程度决定要不要补充集成测试
推荐集成测试工具：test-container，它提供可以在Docker容器中运行的任何东西的轻量级，一次性的实例。我们可以用它启动数据库，中间件去测试

https://testcontainers.com/


### 六. 代码review



## 结论
经过我的实践，给足够时间，以DDD方法论一定能把老架构改动。如果不给予足够的时间重构，以老架构的底子，确实难以实施，在有限时间使用新架构的底子实施可能会更好。




