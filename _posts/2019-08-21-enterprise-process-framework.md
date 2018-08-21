---
layout: post
title: 企业流程架构五步法
date: 2018-08-21
categories: business
tags: [business]
description: 企业流程架构定义和构建方法论
---

当企业进行IT系统建设和改造的时候，一般会选择先从内部进行流程的梳理，厘定业务结构和组织架构设计，最后再对IT架构进行定义。EPF（Enterprise Process Framework）是一套关于流程梳理和IT项目实施的方法论。我自己也参与过若干个采用EPF作为理论指导的大型项目并取得了不错的成果，在这边记录一下EPF的实现过程和个人的心得。

## EPF的意义

对于业务方，EPF可以作为结构化的业务蓝图，识别出业务的边界，并帮助业务聚焦在流程优化的重点，定义运营的标准。

对于组织结构，则指知道组织架构设计，定义"干什么"，"谁来干"，建立流程化组织

对于IT而言，则是提供IT架构建设的关键支撑，尤其是应用系统架构（AA）以及数据架构（DA），分别对应“干什么”和“有哪些数据支持和系统支持”

好的流程架构需要具备下面的要素

1. 业务覆盖面全
    - 覆盖企业运营的不同层次（总公司，分公司）
    - 覆盖业务运营的不同领域（业务价值链，管理管控职能）
2. 体系结构严谨
    - 流程模块之间有必要的层次和逻辑关系
    - 流程模块之间全覆盖，并且没有交集
3. 反映业务逻辑
    - 价值链的业务总体顺序与关系
    - 运营领域内的生命周期
    - 业务开展的逻辑方法
4. 体现业务差异
    - 对于业务价值链流程，各种业务场景模式下，需要不同的业务能力、流程模块

## 主要概念

EPF的核心是定义从L1到L5的五级流程，其中L1-L4为流程模块，L5为具体流程

（L1-L4）流程模块：

    - 定义：逻辑上的工作模块，定义“干什么”
    - 目的：组成流程框架，定义流程之间的边界，指导具体流程的建设
    - 特点：
        - 对业务覆盖全面；
        - 结构严谨，流程模块有必要的层次和逻辑关系，全覆盖，无交集；
        - 反映业务逻辑；
        - 不直接体现现有组织架构与职责划分，但有指导意义；
        - 具有流程业务目的、输入、输出、关键业务控制点、业务逻辑等信息

（L5）具体流程：

    - 定义：工作模块操作的具体描述，主要回答“怎么干”，“谁来干”，“什么时候干”，“用什么干”
    - 目的：作为行为手册，指导具体日常运营的过程
    - 特点：
        - 挂靠在流程架构的流程模块之下，取决于具体流程涵盖的范围，可挂靠在不同层级的流程模块之下
        - 本身是一个流程图，具有具体的工作过程和职责定义
        - 提出业务层面清晰的规则逻辑

* 一级流程（L1）：业务流程主干，同行业内基本一致，属于业务价值链，例如供应链
* 二级流程（L2）：运营层面的业务子流程，因业务场景不同而差异化，例如订单、仓储、运输
* 三级流程（L3）：实现运营模式，根据实际需要锁安排的业务活动，与具体的IT系统没有直接的关系，例如订单管理、合同管理
* 四级流程（L4）：结合特定的IT系统，描述业务和IT系统的主要交互过程、工作流，例如订单管理的主要流程为：订单接收、订单确认、订单分解、订单下发、回单、计费
* 五级流程（L5）：集合特定的IT系统，描述业务与IT系统的详细交互过程、工作流、业务规则、输入输出等，例如订单的接收，包括关键数据校验、数据mapping等

一个完整的L1到L5级流程示例如下

![efp](http://ooi50usvb.bkt.clouddn.com/_efp_1534822300_2275.png)

具体在做流程建模的时候，也可以根据实际的业务特征进行缩减，例如没有L1和L2。

## 流程建模的方法

流程建模的方法，分成两种，分别是自上而下和自下而上，分别对应思维模式中的演绎（从一般到个别）和归纳（从个别到一般）

1. 自上而下（演绎）：

    - 方法
        - 首先构建流程全貌，然后层层分解到细节流程
    - 优点
        - 能够建立流程体系的全貌，容易检验和修订
        - 可以避免流程的重复、重叠和不能进行衔接
        - 最大限度保证了流程的共享和复用
    - 不足
        - 需要清晰的L0框架作为输入
        - 对原有的流程体系进行比较大的修改
    - 总结
        - 适合建立（To Be 也就是理想目标）的业务流程，也就是预定一个业务目标，将流程规划的目标设定为这个业务目标服务

2. 自下而上（归纳）：
    - 方法
        - 从细节流程入手，然后挑选、合并进行处理，生成上一级流程
    - 优点
        - 建立基于实际的流程应用
    - 不足
        - 不能看到流程的全貌
        - 通过底层合并非常困难，很难形成完成的流程体系
    - 总结
        - 适合基于现有流程（AS-IS 也就是客观现状）建立体系结构，也就是在已有的业务上进行变革，寻找流程之间的共性

3. 两种方法会结合使用
    - 从上而下建立流程高阶方案
    - 自下而上进行流程匹配，识别出和高阶方案的矛盾，制定新流程，或者调整流程高阶方案
    - 兼顾流程的完整性和业务场景的选择

## 总体建设方法

![fivestep](http://ooi50usvb.bkt.clouddn.com/_fivestep_1534821901_26289.png)

### 梳理业务模式及场景

![vision](http://ooi50usvb.bkt.clouddn.com/_vision_1534833075_6154.png)

宏观的供应链包括：计划、采购、生成、仓储、运输、退货等各个环节，每一个环节都对应一个业务模式，一般可以直接参考行业划分。

不同企业的特殊性，在于上下游涉及到诸多参与者，会针对差异化经营指定不同的业务规则，以及其他影响业务模式的因素。

![pattern](http://ooi50usvb.bkt.clouddn.com/_pattern_1534832306_4664.png)

在仓储这个业务模型下，有存在N个影响业务场景的维度，识别这些业务维度，可以划分出不同的业务场景，在梳理业务场景这个领域，则要重点关注，企业的主营业务方向，按照客户类型、产品类型、销售渠道、服务分类、人员分工等等进行划分，只要会影响业务场景，而且无法合并的因素都可以作为区分维度。

梳理出业务模式和业务场景，就可以根据业务场景来构建流程架构，特别是对于L4和L5流程，需要能够准确地找到与之对应的业务场景。

### 搭建业务流程架构

第一步中得到的业务场景细分，将作为构建业务流程架构的输入，在这个步骤中，集合业界的最佳实践，输出整体的业务流程框架（L1-L4）

**流程架构**

![template](http://ooi50usvb.bkt.clouddn.com/_template_1534833694_16684.png)

流程架构，不能和系统的功能菜单划等号，它表达的是业务场景，以及业务场景的分成和分类，分别对应L1到L4流程。

例如，当有一个业务场景是“仓库接收到出库任务，生成出库订单，当需要进行订单出库的时候，首先指定拣货计划，再从指定的库区库位中拣取对应的获取到包装区”，那么，L2可以定义为“仓储”；L3定义为“出库单管理”；L4定义为“拣货”

**成熟度对标**

![maturecompare](http://ooi50usvb.bkt.clouddn.com/_maturecomp_1534833762_16897.png)

- 绿色：流程很成熟
- 黄色：有流程，但是成熟度不高，需要优化
- 红色：没有流程，成熟度低

成熟度的判断，一般需要通过对标来判断，一般要进行流程改造都会进行业界的对标，或者由资深的业务专家对现有的业务流程进行评估。

**信息化分析**

![infoanalysis](http://ooi50usvb.bkt.clouddn.com/_infoanalys_1534834478_9189.png)

- 绿色：强依赖，只能在系统中完成操作
- 黄色：一般依赖，人工操作和系统操作协同，或者需要再系统中登记和录入信息
- 灰色：无依赖，一般为线下操作行为

**流程架构信息化分布分析**

![itdistrub](http://ooi50usvb.bkt.clouddn.com/_itdistrub_1534834970_11673.png)

- 绿色：ABC系统
- 黄色：EFG系统

一个信息系统可能只会包括多个流程，也有可能一个业务流程会对应到多个系统。在设计的时候，最好是可以让流程在系统内部完成闭环，因此L4和L5流程在实施之前，也需要评估系统的可落地性。

**IT系统支撑优先级分析**

![support](http://ooi50usvb.bkt.clouddn.com/_support_1534835944_8267.png)

在实践中，一般重心都集中在对IT系统以来程度高的部分，在制定计划时，主要的关注点应该在成熟度较低的部分，因为其可能是系统实施的潜在风险点

### 定义L4流程能力

L4流程是对业务要完成功能的总结

核心要素如下

1. 流程名称
2. 流程代号
3. 业务目标：流程需要完成的业务总结，如“根据订单的客户信息以及库存信息，锁定库存，并生成拣货和发运任务”
4. 输入：流程运行所需要的数据和文档，如“订单数据，库存数据，审批规则”
5. 输出：作为流程运行的结果，输出的数据和文档，如“拣货任务，审批订单，发运任务”
6. 主要活动：流程链路上，主要进行的业务操作，如“提交订单；查询库存；生成任务”
7. 关键业务逻辑规则：执行过程的业务逻辑判断和依赖的规则，如“订单审核通过，并且库存充足”
8. IT系统：承载流程运行的后台IT系统

例如：

![L4ability](http://ooi50usvb.bkt.clouddn.com/_l4ability_1534860248_82073205.png)

### L4流程串联

L4流程串联的目的是要表达，流程要达到的目的和结构，体现什么业务活动，涉及哪些角色，最后绘制流程图。

L2高阶串联

![L2](http://ooi50usvb.bkt.clouddn.com/_l2_1534839509_5337.png)

L3高阶串联

![L3](http://ooi50usvb.bkt.clouddn.com/_l3_1534839565_8986.png)

L4串联

![L4](http://ooi50usvb.bkt.clouddn.com/_l4_1534839729_20361.png)

在L4串联中，一般能够发现一些作业环节衔接的问题，这时候应该做整体的拉通，解决这些冲突，并添加文字说明。

### L5流程展开

L5流程是承载业务落地和IT系统实施的基础，从概念上，L5流程应该具备下面的特点

1. 是现实的抽象，需要反映实际的生产作业流程和步骤
2. 只显示与解决给定问题相关的功能，又不是现实的镜像，应该只局限于特点的问题，避免过度复杂化
3. 具备通用性和可移植性，对于类似的流程，只需要略作改动即可
4. 有侧重的特定视角，如果涉及多方的业务，需要从一个特定的视角加予阐释
5. 简单易懂

从描述上，L5流程有下面5个角度

1. 工具：每个活动的工具载体，比如IT系统，或者手动进行（包括电子邮件、传真、电话等）
2. 数据、文档：流程步骤中涉及的输入，数据和输出文档
3. 参与者、组织：组织、团队、个人按照角色进行活动
4. 时间：活动发生的时间，顺序
5. 活动：流程中的行为列表和行为的结果

![L5](http://ooi50usvb.bkt.clouddn.com/_l5_1534842011_15503.png)  

## 总结

EPF的主要方法论就简单介绍到这边，需要强调的是，EPF的Owner一般都是业务人员或者系统分析师，而不是技术人员，所得到到L1-L5流程图，也都是站在业务角度的思考当前的流程以及可以改进的地方，并不是系统设计和系统交互逻辑。因此一般我们也称其为业务高阶方案。而技术人员需要根据这些业务高阶方案，编写对应的技术高阶方案，作为业务高阶方案的系统解决手段来与之对应。这个转换过程，也有一整套方法论支撑，可以留待以后再做叙述。