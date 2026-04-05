# 面试项目工作上下文（Agent Memory）

## 文档目的

这份 `agent.md` 是当前整个“面试项目”仓库的统一上下文说明，供后续继续分析面试表现、还原项目产出、回答面试官追问时使用。

后续所有分析都要以这份文档为入口，并结合仓库里的真实文件交叉验证，避免空口编故事。

重点说明：

- 本仓库不是单一项目，而是“简历 + 项目源码 + 面试记录 + 面试复盘”的组合工作区。
- `crm_api`、`expr`、`house-spider-googlesheets`、`hunt2018`、`service_api` 这 5 个目录，视为“异乡好居后端开发实习生”阶段的核心源码材料。
- `heima/hm-dianping` 对应个人项目“雅鉴生活志”。
- `面试记录/` 与 `面试总结/` 是后续复盘面试表现的第一手材料。

## 当前用户画像

- 姓名：王康欣
- 学校：北京邮电大学，本科，计算机科学与技术，2023-2027
- 目标方向：Java 后端 / 高并发 / 缓存 / 数据库 / 消息队列 / 工程化后端
- 简历主线：
  - 实习：异乡好居，后端开发实习生（2025.08 - 2025.11）
  - 项目：雅鉴生活志（Spring Boot + Redis + 秒杀链路）

## 仓库总览

### 1. 简历与总述

- `简历-md版.md`：当前主简历，后续回答默认以它为对外口径。
- `heima/简历-md版.md`：疑似旧副本或练习副本，优先级低于根目录简历。

### 2. 面试材料

- `面试记录/*.txt`：原始面试记录。
- `面试总结/*.md`：每场面试的复盘、高分回答、补强点。
- `面试总结/面试问题预测.md`：高频知识点与预测题汇总。

### 3. 个人项目源码

- `heima/hm-dianping`：个人项目“雅鉴生活志”源码。

### 4. 实习相关源码

- `crm_api`：CRM API，ThinkPHP 风格，典型 Controller-Service-Model 分层。
- `service_api`：服务 API，中台/服务化接口集合，含 `service`、`partner`、`passport`、`property` 等应用域。
- `hunt2018`：控制台/定时任务/爬虫/同步服务，偏后端批处理与任务调度。
- `house-spider-googlesheets`：Go 实现的 Google Sheets 房源同步工具，配置驱动 Pipeline 架构。
- `expr`：Go 表达式引擎源码，用于 `house-spider-googlesheets` 中的动态字段映射与表达式执行。

## 后续分析时的资料优先级

1. `简历-md版.md`
2. 本文件 `agent.md`
3. `面试总结/*.md`
4. `面试记录/*.txt`
5. 相关源码目录

说明：

- 需要回答“我简历里这句话怎么落到代码”时，优先从源码找证据。
- 需要分析“我这场面试哪里答差了”时，优先看对应 `面试总结/*.md`，必要时回看 `面试记录/*.txt`。

## 实习源码全景理解

### `crm_api`

定位：

- ThinkPHP 风格的 CRM 后端 API。
- 大量 `apps/crmapi/controller/*`、`apps/common/service/*`、`apps/common/model/*`。
- 接口文档体系完整，配有 `apidoc.json` 与 `apps/*/schema/*.json`。

核心特征：

- 强 Controller-Service 分层。
- 明显的 RESTful API 开发痕迹。
- 存在统一缓存、统一消息队列发送、统一多语言加载与 schema 文档。

关键证据文件：

- `crm_api/apps/crmapi/controller/BaseApi.php`
  - 全局初始化、多语言、分页、认证等基础能力。
- `crm_api/apps/crmapi/controller/HouseCheckLibrary.php`
  - 典型 RESTful 接口控制器，包含参数校验、统一返回、调用 Service。
- `crm_api/apps/common/service/HouseCheckLibraryService.php`
  - 多表 JOIN、查房库搜索、失效处理、业务规则较重。
- `crm_api/apps/common/service/WorkbenchService.php`
  - 统计、聚合、跨表查询明显，是“复杂查询/统计型接口”的证据。
- `crm_api/apps/common/service/PartnerService.php`
  - 涉及组织、来源、日志、队列通知，是业务服务编排的代表。
- `crm_api/apps/common/traits/model/CacheSupport.php`
  - 缓存 Trait，key 已带语言维度。
- `crm_api/configs/extra/cacheRedisKey.php`
  - Redis key 统一管理。
- `crm_api/apps/common/helper/MnsClient.php`
  - MNS 队列封装。
- `crm_api/apps/common/traits/MnsClientTraits.php`
  - 统一消息发送 Trait。

可讲方向：

- CRM 接口开发
- 查房库/合作方/工作台类业务接口
- Redis 缓存治理
- 多语言缓存隔离
- MNS 异步通知

### `service_api`

定位：

- 服务层 API 聚合服务，模块比 `crm_api` 更广，包含 `service`、`partner`、`passport`、`property`。
- 既有业务接口，也有地图、自动申请、爬虫入库、PDF 解析、账号认证等服务。

核心特征：

- Controller 数量大，应用域多。
- `apps/common/service/*` 中沉淀了大量可复用服务。
- 同样有缓存 Trait、MNS Trait、schema/apidoc。

关键证据文件：

- `service_api/apps/service/controller/HouseSpider.php`
  - 提供 `createHouse/createUnit/createSpiderRoom/createHouseNotice/saveHouseFees` 等房源同步接口。
- `service_api/apps/common/service/HouseSpiderService.php`
  - 房源、房型、图片、设施、租期等入库逻辑。
- `service_api/apps/service/controller/Map.php`
  - 地址解析、POI 搜索、周边设施、静态地图、交通时间。
- `service_api/apps/common/service/MapService.php`
  - 周边设施聚合、交通路径计算、缓存。
- `service_api/apps/partner/controller/AutomaApply.php`
  - 自动申请流程入口，串联供应商规则与外部 Python/Node 服务。
- `service_api/apps/common/service/AutomaApplyService.php`
  - 自动申请表单数据装配、模板、组件、订单与客户信息聚合。
- `service_api/apps/common/traits/model/CacheSupport.php`
  - 较旧版本缓存 Trait，key 未带语言维度。
- `service_api/apps/common/helper/GoogleMap.php`
  - 地图服务与缓存。
- `service_api/apps/common/traits/MnsClientTraits.php`
  - 统一异步消息封装。

可讲方向：

- 中台服务 API 开发
- 爬虫结果入库接口
- 地图类接口封装与缓存
- 自动申请/外部服务编排
- 服务层复用与模块拆分

### `hunt2018`

定位：

- 控制台服务 + 定时任务 + 爬虫 + 同步任务。
- 偏“批处理、后台任务、调度系统、数据同步”。

核心特征：

- `apps/hunt/command/*` 大量命令。
- `pm2/`、`crontab`、`testing_crontab` 体现真实定时任务部署方式。
- 大量 spider/sync/statistics/daily 类任务。

关键证据文件：

- `hunt2018/README.md`
  - 明确该项目是 Console 服务。
- `hunt2018/apps/common/helper/MnsClient.php`
  - MNS 队列发送封装。
- `hunt2018/apps/common/helper/QueueUtils.php`
  - 队列优先级、开发告警等统一工具。
- `hunt2018/apps/common/service/SpiderHouseService.php`
  - 爬虫数据同步、邮件通知、缓存。
- `hunt2018/apps/common/service/HouseCheckSpiderService.php`
  - 查房库/房源抓取相关逻辑。
- `hunt2018/apps/hunt/command/spider/CronGrapHouse.php`
  - 多进程抓取、解析 HTML、发队列同步查房库。
- `hunt2018/pm2/testing_crontab`
  - 测试环境大量 cron 任务。
- `hunt2018/pm2/crontab/m216_cn-beijing`
  - 生产环境 cron 任务，能看出真实线上任务密度与分类。

可讲方向：

- 后台任务/定时任务体系
- 爬虫抓取 + 入库/同步
- 队列解耦与通知
- 定时任务调度与批量统计
- 多进程/批处理场景

### `house-spider-googlesheets`

定位：

- Go 写的房源数据同步工具。
- 将 Google Sheets/Excel 中的供应商数据，按配置抽取、转换、序列化、转 MQ。

核心特征：

- Pipeline 架构：extract -> transform -> marshal -> mq_convert -> mq_publisher。
- 配置驱动。
- 支持 expr 表达式映射、AI 解析、MQ 发布、dry-run、重试退避。

关键证据文件：

- `house-spider-googlesheets/README.md`
  - 项目简介、分层架构、快速开始。
- `house-spider-googlesheets/cmd/spider/main.go`
  - 命令行入口。
- `house-spider-googlesheets/internal/pipeline/manager.go`
  - Pipeline Manager，负责 stage 生命周期与统计。
- `house-spider-googlesheets/internal/pipeline/stage/transform/mapping.go`
  - expr 驱动的字段转换。
- `house-spider-googlesheets/internal/pipeline/stage/transform/expr.go`
  - 自定义函数：`parsePrice`、`sqftToSqm`、`parseFacing`、`buildFacilities` 等。
- `house-spider-googlesheets/internal/model/property.go`
  - Property / Unit / Room / Tenancy 标准结构。
- `house-spider-googlesheets/internal/pipeline/stage/mq/convert.go`
  - Property 转 MQ Job。
- `house-spider-googlesheets/internal/pipeline/stage/mq/converters.go`
  - house/unit/room 不同层级消息转换器。
- `house-spider-googlesheets/internal/pipeline/stage/mq/publisher.go`
  - MQ 发布、dry-run、统计。
- `house-spider-googlesheets/pkg/mq/publisher.go`
  - MQ 发布器、重试、退避、适配器抽象。
- `house-spider-googlesheets/configs/spiders/rocklyn.yaml`
  - 很好的真实配置样例。

可讲方向：

- 配置驱动的数据同步工具
- 多供应商 Excel/Google Sheets 标准化
- Go Pipeline 架构
- MQ 任务拆分与发布
- 动态表达式映射

### `expr`

定位：

- Go 表达式引擎源码。
- 在 `house-spider-googlesheets` 中承担表达式编译与执行能力。

重要提醒：

- 当前仓库中 `expr` 看起来是完整表达式引擎源码副本/依赖源码。
- 后续回答时，不要默认宣称“这是我独立从零实现的表达式引擎”，除非用户明确确认自己确实对其做过核心开发或定制。
- 更稳妥的说法是：
  - “我在实习链路中使用/阅读/接入了 expr 表达式能力，用它实现配置驱动的数据转换。”
  - “我理解它的 parser-checker-compiler-vm 执行链路。”

关键证据文件：

- `expr/expr.go`
  - 暴露 `Compile`、配置 Option、Env、Optimize 等入口。
- `expr/parser/parser.go`
  - 词法/语法分析，生成 AST。
- `expr/compiler/compiler.go`
  - AST 编译为字节码程序。
- `expr/vm/vm.go`
  - 虚拟机执行字节码。
- `expr/conf/config.go`
  - 编译配置、内存预算、节点限制、内建函数表。

可讲方向：

- 表达式引擎基本执行链路
- 为什么适合配置驱动转换
- 为什么它比硬编码 if/else 更适合多供应商异构表格

## 简历中 4 条实习经历，与源码证据的映射

### 1. 数据库查询优化（MySQL）

可用源码证据：

- `crm_api/apps/common/service/HouseCheckLibraryService.php`
  - 多次使用 `join`、条件过滤、分组搜索。
- `crm_api/apps/common/service/WorkbenchService.php`
  - 统计类查询、跨表 JOIN、汇总与业务规则计算。
- `service_api/apps/common/service/AutomaApplyService.php`
  - 多模型数据装配与聚合查询。

面试时建议口径：

- 可以把这条讲成“我在实习里主要处理的是复杂业务查询和统计型接口，典型场景包括查房库搜索、工作台统计、自动申请资料聚合；慢查询定位后会从 N+1、JOIN 重写、索引命中、减少重复查询几个方向优化。”
- `500ms -> 50ms` 这个数字来自简历口径，目前仓库里没有直接保留 slow log 或优化前后 diff，所以要把它当作“你的实习结果表述”，不要伪装成代码能直接证明。

### 2. Redis 缓存优化与重构

强证据：

- `crm_api/apps/common/traits/model/CacheSupport.php`
  - key 形如 `20251009CACHE_%s_ID_%d_lang_%s`，已经把语言维度纳入 key。
- `service_api/apps/common/traits/model/CacheSupport.php`
  - 旧 key 形如 `CACHE_2024_%s_ID_%d`，没有语言维度。
- `crm_api/configs/extra/cacheRedisKey.php`
  - Redis key 规范集中管理。
- `service_api/apps/common/helper/GoogleMap.php`
- `service_api/apps/common/service/MapService.php`
  - 对外部地图/周边/路径数据做缓存。

推荐讲法：

- 这条可以很自然地讲成“我发现老缓存实现没有统一 key 规范，而且多语言数据容易混用；后来通过统一 key 模板、抽 Trait 复用、把语言维度放进 key，解决了中英文缓存串数据问题，也减少了重复代码。”
- 如果面试官追问“哪里体现多语言缓存隔离”，优先答 `crm_api` 的 `CacheSupport`，再对比 `service_api` 的旧版本实现。

### 3. 消息队列异步处理

强证据：

- `crm_api/apps/common/helper/MnsClient.php`
- `crm_api/apps/common/traits/MnsClientTraits.php`
- `service_api/apps/common/traits/MnsClientTraits.php`
- `hunt2018/apps/common/helper/MnsClient.php`
- `hunt2018/apps/common/helper/QueueUtils.php`
- 大量 `send_msn_message_queue(...)` 调用点
- `house-spider-googlesheets/pkg/mq/publisher.go`
  - Go 侧 MQ 发布、重试、退避。

推荐讲法：

- “我在实习里不是只会调用消息队列，而是看过并使用过项目里的统一封装，知道它怎样把邮件、短信、群机器人通知、异步任务提交到队列。”
- “在 PHP 老系统里主要是 MNS 封装和统一消息发送；在 Go 房源同步工具里，已经进一步抽象成 MQ 适配器、Publisher、retry/backoff。”

### 4. RESTful API 开发

强证据：

- `crm_api/apps/crmapi/controller/BaseApi.php`
- `crm_api/apps/crmapi/controller/HouseCheckLibrary.php`
- `service_api/apps/service/controller/HouseSpider.php`
- `service_api/apps/service/controller/Map.php`
- `service_api/apps/partner/controller/AutomaApply.php`
- `crm_api/apidoc.json`
- `service_api/apidoc.json`
- 大量 `apps/*/schema/*.json`

推荐讲法：

- “我参与的是典型的 Controller-Service-Model 三层接口开发，控制器负责参数校验和响应包装，Service 承载业务逻辑，schema/apidoc 保证接口文档可维护。”
- “我做过房源同步、查房库、地图、自动申请这类接口，不只是写 CRUD。”

## 个人项目“雅鉴生活志”源码定位

目录：

- `heima/hm-dianping`

核心证据：

- `src/main/java/com/hmdp/service/impl/VoucherOrderServiceImpl.java`
  - Lua + Redis Stream + 单线程消费者 + Redisson 锁 + DB 扣库存。
- `src/main/resources/seckill.lua`
  - 一人一单与库存判断的原子脚本。
- `src/main/java/com/hmdp/utils/CacheClient.java`
  - 缓存穿透、互斥锁防击穿、逻辑过期。
- `src/main/java/com/hmdp/service/impl/ShopServiceImpl.java`
  - Cache Aside + GEO 查询。
- `src/main/java/com/hmdp/utils/RedisIdWorker.java`
  - Redis 自增 + 时间戳的全局 ID。

后续面试分析时：

- 个人项目问题一律优先落到这些文件上，不要只复述面试题模板。

## `house-spider-googlesheets` 与 `expr` 的关系

这条关系以后很可能被追问，必须记住：

- `house-spider-googlesheets` 不是把每个供应商表格都写死处理逻辑。
- 它通过 `transform_mapping` stage 读取 YAML 里的表达式。
- `mapping.go` 会把表达式编译缓存起来，再调用 expr 执行。
- `expr.go` 提供了业务自定义函数，如：
  - `parsePrice`
  - `sqftToSqm`
  - `parseFacing`
  - `buildFacilities`
  - `encodeLeaseType`
- `rocklyn.yaml` 展示了完整从原始列到标准房源结构的转换链路。

一句话总结：

- `expr` 提供“表达式执行引擎”。
- `house-spider-googlesheets` 提供“供应商配置驱动的数据同步框架”。

## 后续回答面试官时的稳妥原则

### 必须坚持

- 代码里没有直接证明的数据，不要强行说成“源码明确显示”。
- 外部开源/依赖源码，不要默认说成自己从零开发。
- 说“参与过、负责过这条链路的一部分、在这个方向上做过优化”通常比“整个系统都是我做的”更稳。

### 可以放心说的

- 你确实可以把实习讲成“围绕房源、查房库、合作方、工作台、自动申请、地图、爬虫同步、异步通知”这些业务线展开。
- 你确实可以把技术主线讲成：
  - MySQL 复杂查询/统计
  - Redis 缓存治理
  - MNS/MQ 异步
  - RESTful API
  - 配置驱动数据同步

### 特别注意

- `expr` 更适合讲“理解与接入能力”，不是默认讲“从零手写表达式引擎”。
- `house-spider-googlesheets` 与 `service_api`/`crm_api` 之间要讲“房源数据同步链路”而不是孤立工具。
- `service_api` 中 `HouseSpider` 控制器与 `hunt2018` 中 spider/cron 命令，可以一起构成“抓取 -> 同步 -> 入库 -> 通知”的完整故事。

## 面试中可直接复用的实习主线

### 主线 1：房源数据链路

- `hunt2018` 负责定时抓取、后台任务、同步触发。
- `service_api` 提供房源同步、房型/房间创建、地图与自动申请等服务接口。
- `crm_api` 提供查房库、合作方、工作台等业务接口。
- `house-spider-googlesheets` 提供 Google Sheets / Excel 的标准化导入能力。

### 主线 2：缓存与多语言

- 老系统已有缓存封装，但 key 规则不统一。
- 在 `crm_api` 里已经能看到按语言隔离缓存 key 的做法。
- 这很适合回答“多语言数据串了怎么办”“缓存怎么治理”的问题。

### 主线 3：异步与通知

- MNS 队列封装在多个 PHP 项目中反复出现。
- 业务里大量异步邮件、通知、群机器人推送。
- Go 工具中又进一步抽象出 MQ publisher/retry/backoff。

## 高频追问的回答锚点

### 问：你实习主要做过哪些模块？

优先锚点：

- `crm_api` 的 `HouseCheckLibrary`、`Workbench`、`Partner`
- `service_api` 的 `HouseSpider`、`Map`、`AutomaApply`
- `hunt2018` 的 spider/cron/sync

### 问：你做的接口和业务价值是什么？

建议答法：

- 查房库：提升房间/租期/媒体资源维护效率。
- Workbench：为业务和管理层提供统计与看板。
- HouseSpider：把外部房源数据结构化入库。
- AutomaApply：把申请流程自动化，减少人工操作。
- Map：给房源地址解析、周边设施、交通信息提供基础服务。

### 问：你做过缓存优化吗？

优先锚点：

- `crm_api/apps/common/traits/model/CacheSupport.php`
- `service_api/apps/common/traits/model/CacheSupport.php`
- `crm_api/configs/extra/cacheRedisKey.php`

### 问：你做过消息队列吗？

优先锚点：

- `crm_api/apps/common/helper/MnsClient.php`
- `crm_api/apps/common/traits/MnsClientTraits.php`
- `hunt2018/apps/common/helper/QueueUtils.php`
- `house-spider-googlesheets/pkg/mq/publisher.go`

### 问：你做过 Go 项目吗？

优先锚点：

- `house-spider-googlesheets`
- Pipeline 架构
- expr 驱动的 transform_mapping
- MQ convert + MQ publisher

### 问：expr 是什么？

稳妥答法：

- “它是 Go 里的表达式引擎，用于把配置文件里的表达式编译并执行。在我们的房源同步工具里，用它把不同供应商表格映射成统一字段结构。我重点理解并使用了它的 parser-checker-compiler-vm 这条执行链路。”

## 面试复盘时的工作方式

后续如果用户让我“分析某一场面试表现”，应按以下流程进行：

1. 先读对应的 `面试总结/*.md`
2. 必要时回看 `面试记录/*.txt`
3. 标出：
   - 哪些问题答偏了
   - 哪些问题答浅了
   - 哪些地方说得像背答案
   - 哪些地方缺少项目证据
4. 用源码补项目证据
5. 输出：
   - 面试失分点
   - 更好的回答版本
   - 可能的追问
   - 应对这些追问时要引用的文件/模块

## 当前仓库中的一个现实提醒

- IDE 里打开过 `工作日志.md`，但当前工作区中未找到该文件。
- 也就是说，后续分析时默认不要依赖“工作日志”作为证据，除非用户补充该文件内容。

## 最后结论

这个仓库现在可以被理解为三层：

1. 对外口径：`简历-md版.md`
2. 对内复盘：`面试记录/` + `面试总结/`
3. 证据源码：
   - 个人项目：`heima/hm-dianping`
   - 实习项目：`crm_api`、`service_api`、`hunt2018`、`house-spider-googlesheets`、`expr`

后续所有面试问题分析，都应该围绕这三层展开，而不是只从知识点模板出发。
