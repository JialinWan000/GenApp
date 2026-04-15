# GenApp — AI 零代码应用生成平台

> 基于 **Spring Boot 3 + LangChain4j + LangGraph4j** 构建的企业级 AI 代码生成平台。
> 用户输入自然语言需求，AI Agent 自动完成素材搜集、代码生成、质量检查、项目构建的全流程，一键部署为可访问的 Web 应用。

---

## 项目架构

```
genapp/                          # Spring Boot 3 单体后端（Java 21）
├── src/main/java/com/jialin/genapp/
│   ├── ai/                      # LangChain4j AI 服务、输入护轨、模型封装
│   ├── langgraph4j/             # LangGraph4j 工作流节点与状态机
│   │   └── node/concurrent/     # 并行图片搜集节点
│   ├── ratelimter/              # 自定义 @RateLimit 注解 + Redisson 令牌桶切面
│   ├── aop/                     # @AuthCheck 权限拦截切面
│   ├── core/                    # 代码文件构建器、解析器、保存器
│   ├── controller/              # REST + SSE 流式接口
│   └── service/                 # 业务服务（含 AI 策略工厂）
genapp-frontend/                 # Vue 3 + Vite + Ant Design Vue 前端
genapp-microservice/             # Spring Cloud Alibaba + Dubbo 微服务版本
sql/                             # 数据库初始化脚本
```

---

## 技术亮点

### 1. AI 工作流（LangGraph4j）

基于 LangGraph4j 构建有向图工作流，将生成任务解耦为独立节点：

```
RouterNode → PromptEnhancerNode → ImagePlanNode
                                       ↓ (并行)
    ContentImageCollector  DiagramCollector  IllustrationCollector  LogoCollector
                                       ↓ (聚合)
                                ImageAggregatorNode
                                       ↓
                    CodeGeneratorNode → CodeQualityCheckNode → ProjectBuilderNode
```

- 图片搜集阶段四路并行，耗时缩短约 **75%**
- 每个节点职责单一，通过统一 State 传递上下文

### 2. AI 智能动态路由

`RouterNode` 使用轻量模型（Qwen-Turbo，max_tokens=100）对用户意图分类，输出 `codeGenType`：

| 类型 | 场景 | 生成策略 |
|---|---|---|
| `HTML_SINGLE` | 简单展示页 | 单文件原生 HTML |
| `HTML_MULTI` | 复杂多页面 | 原生多文件 HTML/JS/CSS |
| `VUE` | 交互型应用 | Vue 3 + Vite 脚手架 |

路由决策以极低的 Token 消耗完成，主生成任务再使用 DeepSeek 推理模型，**在保证生成效果的同时优化模型推理成本**。

### 3. 分布式限流（Redisson RRateLimiter）

通过自定义注解 `@RateLimit` + AOP 切面，对 AI 接口实现三级令牌桶限流：

```java
@RateLimit(key = "USER", rate = 5, rateInterval = 60, message = "请求过于频繁")
public Flux<String> generateApp(...) { ... }
```

- 支持 **用户级 / IP 级 / 接口级** 粒度
- 基于 Redisson 分布式 RRateLimiter，跨实例共享限流状态
- 令牌桶 1 小时自动过期，避免 Redis key 堆积

### 4. AI 输入安全护轨

基于 LangChain4j 的 `InputGuardrail` 接口，在调用 AI 前对用户 Prompt 进行安全检查，拦截敏感词和 Prompt 注入攻击，有效提升 AI 调用安全性。

### 5. 高性能缓存与会话

- **Redis** 存储分布式 Session（30 天 TTL）和 AI 对话记忆（LangChain4j ChatMemoryStore）
- **Caffeine** 本地二级缓存，热点数据无需回源 Redis
- 构建任务、截图上传、封面生成等 I/O 密集操作通过 Java 21 **虚拟线程**异步处理，避免线程阻塞

### 6. 监控体系

- Spring Boot Actuator + Micrometer → Prometheus 采集指标
- Grafana 可视化大盘（`grafana/` 目录含预置 Dashboard）
- 自定义业务指标（AI 调用次数、生成成功率等）

---

## 核心技术栈

| 分类 | 技术 |
|---|---|
| 后端框架 | Spring Boot 3.5 / Java 21 / AOP |
| AI 框架 | LangChain4j 1.1 / LangGraph4j 1.6 |
| 大模型 | DeepSeek Chat / DeepSeek Reasoner / Qwen-Turbo |
| 数据访问 | MyBatis-Flex / MySQL 8 / HikariCP |
| 缓存 & 会话 | Redis / Spring Session / Caffeine |
| 分布式 | Redisson（分布式锁 & 限流）|
| 存储 & 截图 | 腾讯云 COS / Selenium + WebDriverManager |
| 图片素材 | Pexels API / 阿里云 DashScope 文生图 |
| 前端 | Vue 3 / Vite / Pinia / Ant Design Vue / TypeScript |
| 微服务 | Spring Cloud Alibaba / Apache Dubbo / Nacos |
| 接口文档 | Knife4j (OpenAPI 3) |
| 监控 | Prometheus / Grafana / Micrometer |

---

## 快速启动

### 环境要求

- JDK 21+
- MySQL 8.0+
- Redis 6.0+
- Node.js 20+

### 后端

```bash
# 1. 初始化数据库
mysql -u root -p < sql/create_table.sql

# 2. 配置 application.yml（填写 API Key、数据库、Redis）
vim src/main/resources/application.yml

# 3. 启动
./mvnw spring-boot:run
# 接口文档：http://localhost:8123/api/doc.html
```

### 前端

```bash
cd genapp-frontend
npm install
npm run dev
# 访问：http://localhost:5173
```

---

## 数据库设计

| 表名 | 说明 |
|---|---|
| `user` | 用户信息，支持 user/admin 角色 |
| `app` | 生成的应用，含 `deployKey`（唯一部署标识）、`codeGenType`、封面图 |
| `chat_history` | 对话历史，含游标查询复合索引 `(appId, createTime)` |

---

## 项目截图

> 用户输入自然语言需求 → AI 流式输出生成过程 → 实时预览 → 一键部署

（截图待补充）

---

## 联系

- GitHub：[JialinWan000](https://github.com/JialinWan000)
