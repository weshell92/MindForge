# MindForge 系统功能设计与技术设计文档（生产级）

## 1. 文档目标与范围

- **产品目标**：构建“训练优先”的 AI 思维教练系统，帮助用户通过持续训练形成结构化思考、判断与复盘能力。
- **文档用途**：指导架构设计、研发实施、测试验收与运维部署。
- **覆盖范围**：Web/H5 客户端、后端服务、AI 编排层、数据层、消息系统、监控与运维体系。

---

## 2. 系统功能设计

### 2.1 系统架构总体设计

系统采用“前后端分离 + 领域服务化 + AI 编排中台”架构：

1. **客户端层**：
   - 用户端（Web/H5）：体检、训练、反馈、决策日志、成长面板。
   - 运营端（Admin）：题库管理、Rubric 管理、模型策略管理、用户分群运营。
2. **网关层**：
   - API Gateway：鉴权、限流、路由、灰度、审计日志。
3. **业务服务层**：
   - 用户与身份服务
   - 训练编排服务
   - 反馈评分服务
   - 画像与推荐服务
   - 决策复盘服务
   - 成长与等级服务
   - 内容与题库服务
4. **AI 能力层**：
   - Prompt 模板引擎
   - LLM Router（多模型路由与降级）
   - 安全护栏（敏感内容检测、提示注入防护）
   - 评估与回放（离线评测、在线抽检）
5. **数据与基础设施层**：
   - MySQL（交易与业务数据）
   - Redis（缓存/会话/限流）
   - Kafka/RabbitMQ（异步事件）
   - 对象存储（日志附件、报表快照）
   - 向量库（语义检索与相似样本召回）

### 2.2 模块功能拆解

#### A. 思维体检模块
- 8 维能力诊断（事实观点区分、结构拆解、因果分析等）。
- 场景化微任务执行与动态追问。
- 输出：能力雷达、短板标签、初始等级、14 天训练计划。

#### B. 每日训练模块
- 训练题型：观察、拆解、决策、论证、反驳、复盘。
- 任务编排：每日 1~3 题，按用户画像动态调整难度。
- 支持重答、版本对比、用时追踪。

#### C. AI 对练与反馈模块
- 角色模式：教练 / 反方辩手 / 逻辑审稿人 / 复盘教练。
- 输出结构：总分、维度分、问题定位、一步改进建议。
- 句级证据定位：指出“哪一句/哪一段”存在逻辑缺陷。

#### D. 决策与复盘模块
- 决策日志模板：目标、约束、备选方案、权重、风险、验证点。
- 复盘模板：结果偏差、错误类型、流程修正建议。
- 定时提醒与回访机制（D+7 / D+30 校准）。

#### E. 成长面板模块
- 等级体系（L1~L10）与经验值。
- 能力雷达图、错误分布、训练热力图。
- 周报/月报生成与可分享摘要。

#### F. 运营与配置模块
- 题库管理、标签体系、难度策略。
- Rubric 版本管理与 A/B 实验。
- 风险用户识别（低活跃、高挫败）与运营触达。

### 2.3 核心业务流程

#### 2.3.1 Onboarding 流程
1. 注册登录（手机号/邮箱/OAuth）。
2. 采集基础信息（目标、场景偏好、训练频率）。
3. 触发体检任务（异步创建任务集）。
4. 体检完成后生成用户画像与初始训练计划。
5. 首页下发“今日首训”。

#### 2.3.2 训练流程
1. 客户端拉取今日训练包。
2. 用户作答并提交草稿/最终答案。
3. 反馈服务调用 AI 编排进行评分与纠偏。
4. 写入训练结果、更新能力分与经验值。
5. 若低于阈值触发“重写建议”与补练任务。

#### 2.3.3 反馈流程
1. 训练结果进入评分流水。
2. Rubric 引擎计算规则分（结构完整度、证据密度等）。
3. LLM 给出语义分与改写建议。
4. 融合分模型输出最终分数与问题标签。
5. 返回可执行建议并进入用户历史样本池。

#### 2.3.4 复盘流程
1. 到达验证日期或用户主动发起复盘。
2. 对比“决策时假设”与“实际结果”。
3. 错误分类（证据不足/权重失衡/偏差驱动等）。
4. 生成下次决策清单与训练推荐。

### 2.4 用户状态机与数据流

#### 2.4.1 用户状态机
- `NEW`（新注册）
- `ONBOARDING`（体检中）
- `READY`（已建档可训练）
- `TRAINING_ACTIVE`（稳定训练中）
- `AT_RISK`（连续中断或低完成率）
- `REENGAGED`（召回后恢复训练）
- `CHURNED`（超阈值未活跃）

状态迁移示例：
- `NEW -> ONBOARDING -> READY`
- `READY -> TRAINING_ACTIVE -> AT_RISK -> REENGAGED`
- `AT_RISK -> CHURNED`

#### 2.4.2 数据流
1. 行为事件（答题、停留、重写）进入埋点流。
2. 业务写库（训练记录、评分结果、画像快照）。
3. 事件总线广播（`TRAINING_COMPLETED`、`LEVEL_UP` 等）。
4. 推荐服务消费事件更新用户特征。
5. 成长面板服务聚合指标并回传前端。

### 2.5 集成点与第三方服务

- 身份认证：Auth0/自建 OAuth2。
- 支付：Stripe/微信支付（订阅）。
- 推送：短信、邮件、站内信（SendGrid/阿里云短信）。
- LLM：OpenAI / Azure OpenAI / 国产模型（路由层统一抽象）。
- 可观测性：Prometheus + Grafana + Sentry + ELK。

---

## 3. 技术设计

### 3.1 技术栈选型与理由

- **前端**：Next.js + TypeScript + Tailwind
  - 理由：SSR/ISR 兼顾 SEO 与首屏性能，工程成熟。
- **后端**：Go（Gin/Fiber）或 Java（Spring Boot）
  - 理由：高并发、生态成熟、便于服务化治理。
- **数据库**：MySQL 8
  - 理由：事务一致性强，适合训练记录与计费场景。
- **缓存**：Redis
  - 理由：低延迟、支持排行榜/会话/分布式锁。
- **消息队列**：Kafka（高吞吐）
  - 理由：训练事件流、异步解耦、可回放。
- **AI 编排**：LangChain/LlamaIndex + 自定义 Router
  - 理由：快速接入多模型、可观测与策略化路由。
- **部署**：Kubernetes + Helm
  - 理由：弹性扩缩容、灰度发布、环境一致性。

### 3.2 系统架构图（文字描述）

客户端请求进入 API Gateway，经鉴权后路由至对应业务服务；
训练请求同步调用反馈评分服务，评分服务再通过 AI 编排层请求 LLM；
训练完成事件写入 Kafka，推荐服务与成长服务异步消费更新；
聚合数据写回 MySQL/Redis，并通过报表服务输出周报。

### 3.3 微服务/模块划分

1. `user-service`：账号、订阅、权限、用户状态。
2. `assessment-service`：体检任务、题目编排、体检报告。
3. `training-service`：每日训练下发、作答管理、重答版本。
4. `feedback-service`：Rubric 评分、LLM 反馈、融合结果。
5. `recommendation-service`：训练推荐、难度调节、召回策略。
6. `review-service`：决策日志与复盘管理。
7. `growth-service`：等级计算、指标聚合、周报生成。
8. `content-service`：题库、标签、模板管理。
9. `ai-orchestrator`：Prompt、模型路由、降级、审计。

### 3.4 数据库设计（ERD 与表结构）

#### 3.4.1 ERD（文字）
- `users` 1:N `user_profiles`
- `users` 1:N `training_sessions`
- `training_sessions` 1:N `training_answers`
- `training_answers` 1:1 `feedback_reports`
- `users` 1:N `decision_logs`
- `decision_logs` 1:N `review_records`
- `users` 1:N `level_history`
- `question_bank` N:M `training_sessions`（通过 `session_questions`）

#### 3.4.2 核心表结构（示例）

**users**
- `id` bigint PK
- `email` varchar(128) unique
- `phone` varchar(32)
- `status` varchar(32)
- `current_level` int
- `created_at` datetime
- `updated_at` datetime

**user_profiles**
- `id` bigint PK
- `user_id` bigint FK
- `strength_tags` json
- `weakness_tags` json
- `goal` varchar(255)
- `preferred_scene` varchar(64)
- `snapshot_version` int
- `created_at` datetime

**training_sessions**
- `id` bigint PK
- `user_id` bigint FK
- `session_type` varchar(32)
- `status` varchar(32)
- `scheduled_at` datetime
- `started_at` datetime
- `completed_at` datetime

**training_answers**
- `id` bigint PK
- `session_id` bigint FK
- `question_id` bigint
- `answer_text` text
- `revision_no` int
- `duration_sec` int
- `submitted_at` datetime

**feedback_reports**
- `id` bigint PK
- `answer_id` bigint FK unique
- `overall_score` decimal(5,2)
- `dimension_scores` json
- `error_tags` json
- `improvement_actions` json
- `llm_trace_id` varchar(64)
- `created_at` datetime

**decision_logs**
- `id` bigint PK
- `user_id` bigint FK
- `topic` varchar(255)
- `options_json` json
- `weights_json` json
- `confidence` decimal(5,2)
- `review_due_at` datetime
- `created_at` datetime

**review_records**
- `id` bigint PK
- `decision_log_id` bigint FK
- `actual_outcome` text
- `error_category` varchar(64)
- `correction_plan` text
- `created_at` datetime

### 3.5 API 设计（主要接口列表）

#### 身份与用户
- `POST /api/v1/auth/register`
- `POST /api/v1/auth/login`
- `GET /api/v1/users/me`
- `PATCH /api/v1/users/me/profile`

#### 体检与训练
- `POST /api/v1/assessments/start`
- `GET /api/v1/assessments/{id}`
- `GET /api/v1/trainings/today`
- `POST /api/v1/trainings/{sessionId}/answers`
- `POST /api/v1/trainings/{sessionId}/submit`
- `POST /api/v1/trainings/{sessionId}/rewrite`

#### 反馈与推荐
- `GET /api/v1/feedback/{answerId}`
- `GET /api/v1/recommendations/daily-plan`

#### 决策与复盘
- `POST /api/v1/decisions`
- `GET /api/v1/decisions/{id}`
- `POST /api/v1/decisions/{id}/review`

#### 成长与报表
- `GET /api/v1/growth/dashboard`
- `GET /api/v1/reports/weekly`

### 3.6 AI/LLM 集成架构

1. **请求编排**：根据任务类型选择 Prompt 模板（体检/训练/复盘）。
2. **模型路由**：
   - 低成本模型：草稿点评、一般建议。
   - 高性能模型：复杂论证分析、关键评分。
3. **结果融合**：规则分（可解释） + LLM 语义分（上下文理解）。
4. **安全护栏**：
   - 输入输出敏感信息检测。
   - Prompt 注入关键词与模式拦截。
   - 高风险输出触发人工审计队列。
5. **观测追踪**：记录 `trace_id`、token 消耗、响应时延、失败原因。

### 3.7 缓存策略

- 用户主页聚合数据：TTL 5 分钟。
- 题库元数据：TTL 30 分钟 + 主动失效。
- 推荐结果：按用户维度缓存 10 分钟。
- 幂等键缓存：防重复提交（30 秒）。
- 热点排行榜：Redis Sorted Set，定时刷新。

### 3.8 消息队列设计

**核心 Topic**
- `training.completed`
- `feedback.generated`
- `user.level.changed`
- `review.due.reminder`

**消费原则**
- 至少一次投递 + 业务幂等。
- 死信队列（DLQ）处理异常消息。
- 关键事件使用顺序键（user_id）保障同用户顺序。

### 3.9 存储策略

- 结构化数据：MySQL 主从 + 分库分表预案（按 `user_id` 一致性哈希分片）。
  - 分片触发阈值：单表 > 5000 万行或单库 > 2TB。
  - 初始分片：8 分片（`hash(user_id) % 8`）。
  - 扩容策略：达到阈值后扩容为 16/32 分片；通过双写 + 回放迁移存量数据，校验一致后切流。
- 非结构化文本（长复盘、附件）：对象存储。
- 日志：冷热分层（7 天热存 + 90 天冷存）。
- 向量索引：存储高价值历史样本 embedding，用于相似案例召回。

### 3.10 安全性设计

- 鉴权：JWT + Refresh Token + 设备指纹。
- 授权：RBAC（用户/运营/管理员）。
- 传输：全链路 TLS，敏感字段加密存储（AES-256）。
- 数据安全：PII 脱敏、最小权限访问、审计日志不可篡改。
- 接口防护：WAF、限流、签名校验、重放攻击防御。
- AI 安全：越权提示防护、输出合规审查、敏感主题分级策略。

### 3.11 性能优化方案

- 读写分离与索引优化（按 user_id + created_at 复合索引）。
- 异步化非关键链路（周报生成、消息提醒）。
- 批处理聚合（离线任务生成成长指标快照）。
- LLM 调用并发池 + 超时熔断 + 重试退避。
- 客户端性能：首屏分包、懒加载、CDN 缓存。

### 3.12 可扩展性设计

- 领域服务解耦，可独立扩容训练/反馈服务。
- 模型供应商可插拔（统一 Adapter 接口）。
- Rubric 与题库配置化，支持低代码运营。
- 多租户预留：企业版可按 tenant_id 隔离数据与策略。

---

## 4. 核心算法与逻辑设计

### 4.1 思维体检诊断算法

**输入**：体检题作答文本、作答时长、重写次数、场景偏好。  
**处理**：
1. 规则特征提取（结构完整度、证据词密度、反方覆盖率）。
2. 语义特征提取（LLM embedding + 分类器）。
3. 维度评分归一化：
   - `score_dim = 0.6 * rule_score + 0.4 * semantic_score`
4. 标签映射：依据阈值输出强项/短板标签。  
   - 权重说明：0.6/0.4 设计为“规则稳定性优先 + 语义理解补充”。  
   - 可配置性：权重在配置中心按实验组动态调整。  
**输出**：8 维能力分、风险偏差标签、初始训练路径。

### 4.2 个性化训练推荐算法

采用“规则约束 + 上下文 bandit”混合策略：

- 规则层：确保每周覆盖关键能力维度，避免训练偏科。
- 排序层：
  - 特征：近期错误标签、完成率、题型偏好、难度容忍度。
  - 目标：最大化“完成率 × 提升率”。
- 探索利用平衡：
  - 80% 下发高收益题型
  - 20% 探索新题型/新难度

### 4.3 AI 反馈评分 Rubric

评分维度（100 分）：
- 问题定义（20）
- 结构清晰（20）
- 证据质量（20）
- 反方思考（15）
- 逻辑严谨（15）
- 可执行性（10）

**评分流程**
1. 规则校验（是否含结论/论据/反例）。
2. LLM 维度打分与理由生成。
3. 一致性校准（超差阈值触发二次评估）。
4. 输出“分数 + 证据位置 + 改进行动”。

### 4.4 等级升级计算

- 经验值：
  - 完成训练：+10
  - 高质量反馈（>=80）：+5
  - 连续训练奖励：阶梯加成
- 升级公式：
  - `level = floor(log2(total_xp / 100 + 1)) + 1`
  - 边界示例：`XP=0 -> L1`，`XP=100 -> L2`，`XP=300 -> L3`，`XP=51100 -> L10`（51100 为达到 L10 的最小经验值；超过 L10 后进入同级段位成长，不再升主等级）。
- 防刷机制：
  - 同题短时间重复作答收益衰减。
  - 仅“有效训练时长 >= 最小阈值”计入经验。

### 4.5 错误分类与偏差识别逻辑

错误分类主类：
1. 事实-观点混淆
2. 因果倒置/相关性误判
3. 证据不足
4. 忽略反方
5. 目标与约束错配
6. 情绪主导决策

识别流程：
- 规则匹配（关键词/句式模式）
- LLM 分类器（多标签）
- 置信度融合（低置信度入人工标注池）
- 回流训练集迭代优化分类模型

---

## 5. 部署与运维设计

### 5.1 开发环境配置

- 本地：Docker Compose（MySQL/Redis/Kafka/MinIO）。
- 规范：`.env` 管理环境变量，禁止明文密钥入库。
- 密钥管理：统一托管于 Secrets Manager（如 AWS Secrets Manager / Vault）。
- 注入机制：通过 CI/CD 在运行时注入到 K8s Secret，并限制命名空间访问权限。
- 开发环境：使用本地加密模板文件，不使用生产密钥。
- 分支策略：`main`（生产）、`develop`（集成）、feature 分支；feature 必须通过 PR 合并，至少 1 名 Reviewer 通过，CI（lint/test/security scan）全绿后方可合并；默认使用 squash merge，`main` 启用 branch protection（禁止直推）。

### 5.2 测试策略

- 单元测试：核心算法、评分逻辑、状态机迁移。
- 集成测试：服务间调用、数据库事务、消息投递。
- 端到端测试：Onboarding -> 训练 -> 反馈 -> 复盘闭环。
- AI 评测：
  - 离线基准集（稳定性、一致性、偏见检测）
  - 在线抽检（人工复核样本）

### 5.3 CI/CD 流程

1. PR 阶段：lint + unit test + SAST + 依赖漏洞扫描。
2. 合并后：构建镜像、推送镜像仓库、自动部署到 staging。
3. 发布生产：灰度发布（5% -> 20% -> 100%）+ 自动回滚策略。
4. 发布校验：关键指标（错误率、P95 时延、训练完成率）达标后放量。

### 5.4 监控告警

- 技术指标：QPS、错误率、P95/P99、DB 慢查询、队列堆积。
- 业务指标：训练完成率、反馈生成成功率、次日留存、召回率。
- 告警策略：
  - P1：核心链路不可用（电话+IM）
  - P2：性能劣化（IM）
  - P3：非核心异常（工单）

### 5.5 容错机制

- 熔断降级：LLM 超时则退化为规则反馈模板。
- 重试策略：指数退避 + 最大重试次数。
- 幂等保障：请求幂等键 + 消息去重表。
- 灾备：跨可用区部署，数据库定时备份 + 演练恢复。

---

## 6. 里程碑与实施建议（工程落地）

### Phase 1（MVP，8~10 周）
- 交付：体检、每日训练、AI 反馈、成长面板基础版。
- 指标：训练完成率 > 45%，反馈生成成功率 > 99%。

### Phase 2（增强，6~8 周）
- 交付：决策日志、复盘系统、个性化推荐 v1、周报。
- 指标：7 日留存提升 20%, 复盘使用率 > 30%。

### Phase 3（规模化，8~12 周）
- 交付：多模型路由、企业版能力、多租户支持。
- 指标：单位训练成本下降 30%，系统可用性达 99.9%。

---

## 7. 风险与治理

- **风险 1：AI 反馈空泛** → 使用 Rubric + 句级证据定位 + 抽检闭环。
- **风险 2：成本过高** → 模型分层路由 + 缓存 + 长文本摘要复用。
- **风险 3：用户中断训练** → 风险分群召回 + 低门槛补练机制。
- **风险 4：数据合规压力** → 分级脱敏、数据最小化、可审计可删除。

---

## 8. 验收标准

1. 文档覆盖系统功能、技术、算法、运维四大部分。
2. 关键模块均有明确边界、数据结构与接口定义。
3. 能支撑工程团队完成架构评审、开发拆解与上线实施。
4. 可追踪关键技术决策（模型路由、安全、扩展性、容错）。
