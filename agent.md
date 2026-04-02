# 项目上下文（智能体持久记忆）

本文档用于给后续对话提供长期上下文。默认假设未来每次协作前都会先参考本文件，因此这里记录的是“已经确认过的项目事实”、关键业务链路、环境依赖、已知风险点，以及后续修改时的协作约定。

## 一、仓库整体定位

这是一个混合型仓库，核心上由三部分组成：

1. `heima/hm-dianping`
   一个 Java 后端练习项目，技术栈偏典型互联网面试项目，主题是“点评/探店/优惠券秒杀/社交互动”。
2. `AI大模型RAG与智能体开发_Agent项目`
   一个 Python 练习项目，主题是“扫地机器人智能客服 + RAG 检索 + Agent 工具调用 + 使用报告生成”。
3. `面试记录`、`面试总结`、`heima/简历-md版.md`
   个人面试资料区与简历资料区，用于面试复盘、问题整理、项目表达沉淀。

可以把这个仓库理解成“项目练习 + 面试整理 + 简历素材”的统一工作区。

## 二、当前仓库结构认知

顶层主要目录：

- `heima/`
  包含 Java 项目和一份 Markdown 简历。
- `AI大模型RAG与智能体开发_Agent项目/`
  包含 Python Agent/RAG 项目全部源码、配置、数据、向量库和日志。
- `面试记录/`
  原始面试文本记录，偏逐字或半逐字。
- `面试总结/`
  结构化复盘文档，偏 Markdown 归纳版。
- `工作日志.md`
  与 Codex 协作的问题索引文件，重点记录“用户问过什么问题”以及对应的简短结论，便于后续快速检索历史问题。
- `.idea/`
  IDE 工程文件，当前是未跟踪状态，默认视为用户本地配置，不主动修改。

当前 git 认知：

- 当前分支是 `develop-1`。
- 用户已有修改：`面试总结/2026-3-16腾讯后端1面复盘.md`。
- `agent.md` 是后续上下文文件，可以持续维护。
- `.idea/` 当前为未跟踪目录，默认不碰。

## 三、Java 项目认知：`heima/hm-dianping`

### 1. 项目定位

这是一个典型的 Spring Boot 后端练手项目，和简历中的“雅鉴生活志”高度相关，核心业务包括：

- 手机号验证码登录
- 商铺查询
- 商铺分类
- 探店博客
- 点赞
- 关注与共同关注
- 粉丝收件箱时间线流
- 优惠券与秒杀下单
- Redis 缓存、GEO、Stream、Lua、分布式锁等面试高频点

从业务表达上看，这个项目非常适合拿来讲：

- Redis 缓存穿透/击穿/逻辑过期
- 一人一单与防超卖
- Redis Stream 异步下单
- GEO 附近店铺
- 社交关系和时间线流

### 2. 技术栈

从 `pom.xml` 可确认：

- `Spring Boot 2.3.12.RELEASE`
- `Java 8`
- `Spring Web`
- `Spring Data Redis`
- `Lettuce`
- `MyBatis-Plus 3.4.3`
- `MySQL 5.1.47`
- `Hutool`
- `Redisson 3.13.6`
- `Lombok`

### 3. 代码组织

Java 源码目录位于 `heima/hm-dianping/src/main/java/com/hmdp`，分层比较标准：

- `controller`
  提供 HTTP 接口。
- `service` / `service/impl`
  核心业务逻辑实现。
- `mapper`
  MyBatis-Plus 持久层。
- `entity`
  数据库实体。
- `dto`
  接口入参/出参对象。
- `config`
  MVC、MyBatis、Redisson、异常处理等配置。
- `utils`
  缓存、锁、Redis ID、拦截器、正则、ThreadLocal 用户上下文等通用组件。

主启动类：

- `com.hmdp.HmDianPingApplication`

### 4. 核心业务模块认知

#### 4.1 用户登录与登录态

相关文件：

- `UserController`
- `UserServiceImpl`
- `RefreshTokenInterceptor`
- `LoginInterceptor`
- `UserHolder`

已确认链路：

1. 用户通过 `/user/code` 获取验证码。
2. 验证码不存 Session，实际存 Redis，key 前缀是 `login:code:`。
3. 用户通过 `/user/login` 登录，校验验证码。
4. 若手机号未注册，会自动创建新用户。
5. 登录成功后生成 token，把 `UserDTO` 转成 Hash 存入 Redis，key 前缀是 `login:token:`。
6. `RefreshTokenInterceptor` 会从请求头 `authorization` 中取 token，查 Redis，写入 `UserHolder`，并刷新 token TTL。
7. `LoginInterceptor` 只做“是否已登录”校验，未登录直接返回 `401`。

这套设计是“Redis 登录态 + ThreadLocal 用户上下文”的典型面试实现。

#### 4.2 店铺查询、缓存与 GEO

相关文件：

- `ShopController`
- `ShopServiceImpl`
- `CacheClient`
- `RedisConstants`

已确认链路：

- 店铺详情查询优先走 Redis。
- `CacheClient` 封装了三种典型缓存策略：
  - 缓存穿透：`queryWithPassThrough`
  - 互斥锁防击穿：`queryWithMutex`
  - 逻辑过期防击穿：`queryWithLogicalExpire`
- 当前 `ShopServiceImpl.queryById()` 默认实际使用的是“缓存空值防穿透”方案。
- 店铺更新采取“先更新数据库，再删除缓存”的双写一致性常见写法。
- `queryShopByType()` 支持两种模式：
  - 无坐标：走数据库分页查询
  - 有坐标：走 Redis GEO 搜索，按距离排序并分页
- GEO key 前缀是 `shop:geo:`

这是整个项目里最典型的缓存面试模块。

#### 4.3 博客、点赞与 Feed 流

相关文件：

- `BlogController`
- `BlogServiceImpl`
- `FollowServiceImpl`

已确认链路：

- 热门博客按 `liked` 字段倒序分页。
- 用户点赞信息存 Redis ZSet，key 前缀 `blog:liked:`。
- 点赞时：
  - DB 中 `liked = liked + 1`
  - Redis ZSet 记录用户 ID，score 使用时间戳
- 取消点赞时做反向操作。
- 查询点赞前 5 用户时，会从 ZSet 取前 5，再按原顺序查用户。
- 发布博客后，会把博客 ID 推送到所有粉丝的收件箱。
- 粉丝收件箱使用 Redis ZSet，key 前缀 `feed:`。
- 滚动分页 `queryBlogOfFollow()` 使用的是按 score 倒序 + offset 的滚动分页方式。

这部分可以用于面试中讲“点赞去重”“时间线推送”“滚动分页”。

#### 4.4 关注与共同关注

相关文件：

- `FollowController`
- `FollowServiceImpl`

已确认链路：

- 关注关系会同时落数据库和 Redis Set。
- Redis key 形如 `follows:{userId}`。
- 共同关注是直接取两个 Redis Set 的交集，再回表查用户。

这部分是 Redis Set 典型使用场景。

#### 4.5 优惠券秒杀与异步下单

相关文件：

- `VoucherController`
- `VoucherOrderController`
- `VoucherServiceImpl`
- `VoucherOrderServiceImpl`
- `RedisIdWorker`
- `seckill.lua`
- `RedissonConfig`

这是 Java 项目最值得重点记住的模块。

已确认秒杀主链路：

1. 用户发起秒杀请求。
2. `VoucherOrderServiceImpl.seckillVoucher()` 先生成订单 ID。
3. 调用 `seckill.lua` 在 Redis 内完成原子操作：
   - 判断库存是否充足
   - 判断用户是否重复下单
   - 扣减库存
   - 记录已下单用户
   - 向 `stream.orders` 写入订单消息
4. Lua 返回成功后，接口立即返回订单 ID，主线程不做落库。
5. `@PostConstruct` 启动单线程消费器 `VoucherOrderHandler`，持续消费 Redis Stream。
6. 后台线程拿到消息后，调用 `createVoucherOrder()`：
   - 用 Redisson 按用户加锁
   - 检查一人一单
   - 扣减数据库库存
   - 保存订单
7. 处理成功后 ACK 消息。

这条链路里融合了：

- Lua 原子校验
- Redis Stream 削峰
- Redisson 分布式锁
- Redis 分布式 ID
- 异步下单

非常适合做项目亮点。

### 5. Redis 关键常量认知

`RedisConstants` 中已确认的关键 key：

- `login:code:`
- `login:token:`
- `cache:shop:`
- `lock:shop:`
- `seckill:stock:`
- `blog:liked:`
- `feed:`
- `shop:geo:`
- `sign:`

签到模块也已经实现：

- `UserServiceImpl.sign()`
- `UserServiceImpl.signCount()`

它使用 Redis Bitmap 记录签到，用 `BITFIELD` 统计连续签到天数。

### 6. 配置与外部依赖

从 `application.yaml` 和配置类可确认：

- 服务端口：`8081`
- MySQL：
  - 地址：`127.0.0.1:3306/hmdp`
  - 用户名：`root`
  - 密码：`123`
- Redis：
  - 地址：`192.168.150.101:6379`
  - 密码：`123321`
- Redisson 也连接同一个 Redis。

这说明 Java 项目对本地/局域网依赖比较强，想真正启动需要：

- 可用的 MySQL
- 可用的 Redis
- 已导入 `hmdp.sql`

此外还有一个平台相关风险：

- `SystemConstants.IMAGE_UPLOAD_DIR` 目前写死为 Windows 路径：
  `D:\\lesson\\nginx-1.18.0\\html\\hmdp\\imgs\\`

在 macOS/Linux 环境下如果使用上传功能，需要额外调整。

### 7. 数据库认知

SQL 脚本位于：

- `heima/hm-dianping/src/main/resources/db/hmdp.sql`

已确认主要表：

- `tb_blog`
- `tb_blog_comments`
- `tb_follow`
- `tb_seckill_voucher`
- `tb_shop`
- `tb_shop_type`
- `tb_user`
- `tb_user_info`
- `tb_voucher`
- `tb_voucher_order`

数据内容是“商铺/探店/用户/优惠券”一整套演示数据。

### 8. 测试认知

测试类位于：

- `HmDianPingApplicationTests`
- `NormalTest`
- `RedissonTest`

`HmDianPingApplicationTests` 里已确认有以下用途：

- 测 Redis ID 生成性能
- 预热逻辑过期缓存
- 批量导入店铺 GEO 数据
- HyperLogLog 统计测试

### 9. 当前已知注意点

以下是我读代码时确认到、后续排障时应优先关注的点：

1. `VoucherOrderServiceImpl` 中 Redis Stream 的读写流名主要使用的是 `stream.orders`，但 ACK 时使用了 `"s1"`，存在名称不一致的风险，后续如果排查秒杀消息确认异常，这里应优先检查。
2. Java 项目当前仓库里没有 `mvnw`，当前机器上也没有 `mvn`，因此现阶段不能直接在本机完成 Maven 编译验证。
3. 项目对 Redis/MySQL 的地址是固定配置，不是环境变量化配置，迁移环境时要先改配置。

## 四、Python 项目认知：`AI大模型RAG与智能体开发_Agent项目`

### 1. 项目定位

这是一个面向“扫地机器人/扫拖一体机器人”的智能客服 Agent 练习项目，融合了：

- Streamlit 聊天界面
- LangChain Agent
- RAG 检索
- 工具调用
- 动态提示词切换
- 外部 CSV 数据补充
- 报告生成场景

它不是单纯问答，而是一个“可调用工具的客服智能体演示项目”。

### 2. 技术栈

从源码导入可确认：

- `Streamlit`
- `LangChain`
- `LangGraph`
- `langchain-chroma`
- `Chroma`
- `DashScopeEmbeddings`
- `ChatTongyi`
- `PyPDFLoader`
- `TextLoader`
- `yaml`

模型层由 `model/factory.py` 创建：

- 聊天模型：`qwen3-max`
- 向量模型：`text-embedding-v4`

也就是说，这个项目默认依赖阿里云通义/百炼相关生态。

### 3. 目录结构

核心目录如下：

- `app.py`
  Streamlit 前端入口。
- `agent/`
  Agent 封装与工具定义。
- `rag/`
  向量库与 RAG 总结逻辑。
- `model/`
  模型工厂。
- `utils/`
  配置、路径、日志、文件加载等工具函数。
- `config/`
  YAML 配置。
- `prompts/`
  提示词模板。
- `data/`
  知识库文本/PDF和外部 CSV 数据。
- `chroma_db/`
  持久化向量库目录之一。
- `logs/`
  运行日志。

### 4. 运行链路认知

#### 4.1 Streamlit UI

`app.py` 已确认做了以下事情：

- 渲染标题“智扫通机器人智能客服”
- 使用 `st.session_state` 保存 `agent` 和对话消息
- 通过 `st.chat_input()` 接收用户输入
- 调用 `ReactAgent.execute_stream(prompt)` 流式返回结果
- 把生成内容逐字符打印，形成打字机效果

这个项目前端非常轻，核心逻辑都在 Agent 层。

#### 4.2 Agent 主体

`agent/react_agent.py` 已确认：

- 通过 `create_agent()` 创建 Agent
- 注入系统提示词
- 注册多个工具
- 注册多个 middleware
- 执行时通过 `self.agent.stream(..., context={"report": False})` 传入运行时上下文

当前已接入工具：

- `rag_summarize`
- `get_weather`
- `get_user_location`
- `get_user_id`
- `get_current_month`
- `fetch_external_data`
- `fill_context_for_report`

当前已接入中间件：

- `monitor_tool`
- `log_before_model`
- `report_prompt_switch`

#### 4.3 动态提示词切换

这是该项目一个比较重要的设计点。

中间件 `report_prompt_switch` 会根据 `runtime.context["report"]` 决定系统提示词：

- `False` 时使用普通客服提示词 `main_prompt.txt`
- `True` 时切换到报告生成提示词 `report_prompt.txt`

而 `fill_context_for_report()` 这个工具本身不返回业务数据，它的意义是：

- 被调用后，通过 `monitor_tool` 把 `request.runtime.context["report"]` 改成 `True`
- 为之后的报告场景切换提示词

也就是说：

- “生成报告”不是普通问答逻辑
- 它通过“调用一个上下文注入工具”来触发提示词切换

这是整个 Python 项目最值得记住的设计亮点。

### 5. RAG 部分认知

#### 5.1 RAG 工作方式

`rag/rag_service.py` 已确认链路：

1. 创建 `VectorStoreService`
2. 获取 retriever
3. 加载 RAG 提示词模板
4. 检索相关文档
5. 把检索结果拼成 `context`
6. 用模型生成“基于参考资料的总结回答”

`rag_summarize.txt` 对输出限制较严：

- 必须基于参考资料
- 不编造
- 只用中文
- 仅输出概括内容本身

#### 5.2 向量库与分片

`rag/vector_store.py` 已确认：

- 向量库实现：`Chroma`
- 分片器：`RecursiveCharacterTextSplitter`
- 当前配置：
  - `chunk_size = 200`
  - `chunk_overlap = 20`
  - `k = 3`
- 允许导入的知识库类型：
  - `txt`
  - `pdf`

知识导入时还做了 MD5 去重：

- 已处理文件的 MD5 记录在 `md5.text`
- 重复文件不会重复写入向量库

#### 5.3 知识源

`data/` 下已确认存在的知识文件包括：

- `扫地机器人100问.pdf`
- `扫拖一体机器人100问.txt`
- `扫地机器人100问2.txt`
- `维护保养.txt`
- `故障排除.txt`
- `选购指南.txt`

可见知识库主题集中在：

- 使用说明
- 故障排查
- 维护保养
- 选购建议

### 6. 工具层认知

`agent/tools/agent_tools.py` 中工具并不都是真实外部接口，很多是 Demo/模拟实现：

#### 6.1 已确认的模拟/伪造能力

- `get_weather(city)`
  返回固定格式天气字符串，不是真实天气 API。
- `get_user_location()`
  从 `深圳/合肥/杭州` 中随机返回一个。
- `get_user_id()`
  从 `1001~1010` 随机返回一个。
- `get_current_month()`
  从 `2025-01 ~ 2025-12` 中随机返回一个。

因此这个项目当前更像“流程演示用 Agent”，而不是已接真实生产接口的系统。

#### 6.2 外部数据能力

`fetch_external_data(user_id, month)` 的数据源是：

- `data/external/records.csv`

CSV 已确认结构：

- 用户 ID
- 特征
- 清洁效率
- 耗材
- 对比
- 时间

用户范围是 `1001 ~ 1010`，时间范围覆盖 `2025-01 ~ 2025-12`。

工具读取逻辑：

- 首次调用时把整个 CSV 加载到内存字典 `external_data`
- 后续直接从内存中按 `user_id + month` 返回结果

### 7. 提示词认知

当前共有三套提示词：

1. `prompts/main_prompt.txt`
   普通客服主提示词，强调 ReAct 思考、工具调用规范、报告场景强约束。
2. `prompts/rag_summarize.txt`
   RAG 总结专用提示词，要求严格基于参考资料回答。
3. `prompts/report_prompt.txt`
   报告写作提示词，用于生成“黑马程序员扫地机器人使用情况报告与保养建议”。

主提示词里还特别规定：

- 如果识别为报告场景，必须按以下固定顺序执行：
  `get_user_id -> get_current_month -> fill_context_for_report -> fetch_external_data`

这说明项目作者在有意训练“基于工具链条完成复杂任务”的 Agent 行为。

### 8. 配置与路径认知

配置文件：

- `config/rag.yml`
- `config/chroma.yml`
- `config/prompts.yml`
- `config/agent.yml`

当前关键配置：

- 聊天模型：`qwen3-max`
- 向量模型：`text-embedding-v4`
- Chroma collection：`agent`
- 外部数据路径：`data/external/records.csv`

路径工具 `utils/path_tool.py` 已确认：

- 配置文件、提示词、数据文件等大多通过“项目根目录 + 相对路径”转绝对路径

但要特别记住一个细节：

- `VectorStoreService` 创建 Chroma 时，`persist_directory` 直接使用了配置里的相对路径 `chroma_db`，没有统一转成绝对路径。

这会带来一个实际后果：

- 向量库存储目录会受“当前启动工作目录”影响。

仓库里目前同时存在：

- `AI大模型RAG与智能体开发_Agent项目/chroma_db`
- `AI大模型RAG与智能体开发_Agent项目/agent/chroma_db`
- `AI大模型RAG与智能体开发_Agent项目/rag/chroma_db`

这很像是历史上在不同工作目录下运行后留下的产物。后续如果遇到“向量库不生效/数据不一致/检索不到内容”，要优先检查这里。

### 9. 日志与工具函数认知

`utils/logger_handler.py` 已确认：

- 日志同时输出控制台和文件
- 默认日志文件写到 `logs/agent_YYYYMMDD.log`

`utils/file_handler.py` 已确认：

- 支持 PDF 和 TXT 加载
- 提供文件 MD5 计算
- 提供按允许后缀筛文件

### 10. 启动与验证认知

#### 10.1 启动方式判断

虽然入口文件叫 `app.py`，但它使用的是 Streamlit 组件，因此更合理的启动方式应该是：

```bash
cd AI大模型RAG与智能体开发_Agent项目
streamlit run app.py
```

不是普通意义上的：

```bash
python app.py
```

#### 10.2 当前验证结论

我已经在当前机器上完成的事实性验证：

- `python3 -m compileall .` 在把 `PYTHONPYCACHEPREFIX` 指到 `/tmp` 后可以通过，说明当前 Python 源码至少不存在明显语法错误。
- 仓库里没有 `requirements.txt`、`pyproject.toml`、`poetry.lock` 等依赖清单文件。

因此后续如果要真正运行，需要手动确认：

- Python 环境版本
- 依赖安装方式
- DashScope / 通义模型相关鉴权
- Streamlit 是否已安装

### 11. 当前已知注意点

1. 项目是演示性质，多个工具是随机值或固定返回值，不要误判成真实生产系统。
2. 报告生成依赖中间件切换提示词，若工具链顺序错了，结果会不对。
3. 向量库存储目录受启动路径影响，这是后续排障的高优先级检查点。
4. 该项目没有标准依赖清单文件，环境复现成本比普通 Python 项目高。

## 五、面试资料区认知

### 1. `面试记录/`

这里存的是原始面试记录，文件名按日期 + 公司命名，例如：

- `2026-3-13 美团后端1面.txt`
- `2026-3-16 腾讯后端1面.txt`

它们更像“原材料”。

### 2. `面试总结/`

这里存的是结构化复盘稿，文件名与面试记录一一对应，例如：

- `2026-3-16腾讯后端1面复盘.md`

此外还有：

- `面试问题预测.md`

这个文件不是单次复盘，而是对多场面试的高频问题做统一归类，已经整理成一份面试问答库，且明显以“雅鉴生活志 / Redis / 秒杀 / 缓存 / MQ / Java 基础”等面试方向为主。

### 3. `heima/简历-md版.md`

这是一份 Markdown 简历，已经明确把项目经验总结成：

- 实习：异乡好居
- 项目：雅鉴生活志

且项目描述和 `hm-dianping` 的业务/技术点高度一致。后续如果要做：

- 项目润色
- 面试问答对齐
- 简历与项目统一口径

应默认把 `hm-dianping` 视为“雅鉴生活志”的技术落地来源之一。

## 六、后续协作默认约定

后续所有对话，默认带着下面这些上下文和约定继续：

1. 这个仓库不是单一项目，而是“Java 后端项目 + Python Agent 项目 + 面试资料库”的组合工作区。
2. 如果用户提到“项目”，需要结合上下文判断指的是：
   - `hm-dianping`
   - Python Agent 项目
   - 面试总结文档
3. 修改面试文档时，优先保留用户原意和事实，不随意改写成模板腔。
4. 默认不主动修改：
   - `.idea/`
   - `__pycache__/`
   - `chroma_db/`
   - `logs/`
   - `md5.text`
   除非任务明确要求。
5. 如果要排查 Java 项目，先检查：
   - MySQL 是否存在 `hmdp`
   - Redis 是否可连
   - 本机是否有 `mvn`
   - 上传目录是否需要改成当前系统路径
6. 如果要排查 Python Agent 项目，先检查：
   - 启动方式是否为 `streamlit run app.py`
   - 当前工作目录是否正确
   - DashScope/通义模型鉴权是否齐全
   - 向量库是否读到了正确的 `chroma_db`
7. 后续每当用户提出一个新的问题，都应把该问题追加记录到仓库根目录的 `工作日志.md` 中：
   - 重点记录问题本身
   - 按主题归类
   - 只保留简短结论，不展开成长篇答案
   - 保持文件可检索、可持续追加
8. 若用户说“帮我初始化项目”，默认不仅是看目录，还要补充“关键业务链路、配置依赖、运行方式、注意点”到本文件。

## 七、后续最可能的工作方向

基于当前仓库内容，后续高概率任务会落在以下几类：

1. Java 项目面试表达
   包括秒杀链路、缓存策略、Redis 场景、项目亮点提炼。
2. Java 项目代码修复
   包括 Redis Stream、缓存逻辑、分布式锁、接口逻辑问题。
3. Python Agent 项目运行与调试
   包括依赖安装、启动方式、RAG 检索不生效、提示词切换不生效、向量库目录错乱等。
4. 面试文档润色
   包括复盘整理、答案标准化、STAR 表达、项目统一口径。
5. 简历与项目统一
   包括把 `hm-dianping` 的代码细节和“雅鉴生活志”简历表达对齐。

## 八、当前初始化结论

截至当前，我已经完成的初始化认知如下：

1. 已通读仓库主结构。
2. 已通读两个代码项目的关键入口、配置、核心模块和部分测试。
3. 已确认 Java 项目的主业务骨架和 Python 项目的 Agent/RAG 架构。
4. 已确认 Python 项目源码可通过语法级编译检查。
5. 已确认 Java 项目当前机器缺少 `mvn`，暂未完成编译级验证。
6. 已将上述事实沉淀到本文件，供后续所有对话复用。
