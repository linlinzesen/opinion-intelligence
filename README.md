一、课程设计目的

《大型程序设计实践》是网络空间安全学院面向本科生开设的综合实践课程，旨在通过一个完整的软件工程项目，让学生综合运用所学知识培养Web后端开发能力(Flask框架设计、SQLite)、现代前端开发能力(Vue 3 + Element Plus + ECharts技术栈)、数据工程基础能力（爬虫、NLP）、大模型应用开发能力(OpenAI兼容API的调用方式:利用APIkey、Prompt)的能力。本课程要求学生以团队协作的方式，使用Git/GitHub进行分支管理、代码审查与版本控制，遵循约定式提交规范，完成一个具备实际应用价值的Web信息系统。


我们小组选择的课题为“网络舆情事件智能分析系统”。该课题的目标是设计并开发一个从多个信息源（微博、B站、今日头条、小红书）采集并识别热点舆情事件的智能分析平台，能够自动分析事件的传播趋势、群众情感倾向分布和风险等级，并借助大语言模型（LLM）提供智能问答、趋势预测与自动报告生成等高级功能。


二、需求分析

2.1 项目背景

在信息时代，网络舆情事件的传播速度和影响力呈指数级增长。政府部门、企业和公关机构需要对网络舆情进行实时监测和科学分析，以便及时了解公众情绪、预判事件走向并制定应对策略。然而，传统的舆情监测方式存在数据源单一、分析维度有限、响应速度慢等痛点。因此，开发一个能够自动采集多平台数据、进行多维分析并辅助决策的智能舆情分析系统具有重要的现实意义。

2.2 功能需求

根据项目目标，系统需要满足以下功能需求：
（1）用户认证模块：支持用户名+密码登录，使用JWT令牌进行身份认证，保护API接口安全。默认管理员账号为admin/123456，4个分析员账号2024212876、2024212591、2024212581、2024212496,初始密码都为123456。
（2）系统首页与概览看板：展示全网舆情态势总览，包括监测事件总数、热点事件数、高风险事件数、监测平台数、综合情感指数等关键指标；提供全网热度走势折线图和重点风险事件快捷入口。
（3）事件看板：以卡片列表形式展示所有监测事件，支持按关键词搜索、按风险等级筛选、按时间/热度排序，方便用户快速定位目标事件。
（4）事件详情页：针对单个事件，提供事件概述信息（风险等级、情感倾向、热度指数、首发平台、发生时间）；使用ECharts展示发展趋势折线图、情感分布饼图、平台分布柱状图和关键词词云。
（5）智能问答模块：基于当前事件上下文，用户可以自由提问，大模型（或规则引擎）返回分析回答。系统内置问题相关性判断，对无关问题礼貌拒绝。
（7）趋势预测模块：基于历史热度数据，调用numpy库，预测未来趋势。
（8）自动报告生成模块：基于事件的结构化数据，自动生成包含事件概述、传播趋势分析、情感倾向分析、风险等级研判和处置建议五个部分的结构化Markdown舆情分析报告。
（9）爬取控制：可以精确控制爬取评论数和目标平台，采用定时启动和前端轮询的方法，实时对选中的舆情事件进行追踪。

2.3 非功能需求
可靠性：所有大模型相关功能均设计有降级兜底机制（LLM APIkey不可用时自动切换为规则式/Mock模式），确保课程演示过程中不会因外部服务故障而中断；
可用性：前端界面采用响应式设计，操作流程简洁直观，关键操作（搜索、排序、问答、报告生成）均提供loading状态反馈（提高与用户的交互能力）；
可维护性：后端采用Blueprint模块化架构，前端采用组件化开发，数据层与业务层分离，各模块独立可测；
安全性：所有API接口（除登录外）均需携带JWT令牌访问，密码使用SHA-256加盐哈希存储，不保存明文；
兼容性：LLM服务兼容OpenAI API标准，支持通义千问、DeepSeek、ChatGpt等多种大模型后端。

三、详细设计
我们设计的工作流是：
数据采集层（爬虫）→ 数据处理层（NLP分析）→ JSON中间文件 → 数据导入层 → SQLite数据库 → Flask API服务层 → Vue前端展示层。

层级	技术选型	说明
前端	Vue 3 + Element Plus + ECharts	组件化开发，丰富的UI组件和图表支持
后端	Flask (Python) + Blueprint	轻量级框架，模块化路由管理
数据库	SQLite	零配置部署，适合中小规模数据
数据采集	requests + BeautifulSoup	HTTP请求与HTML解析
NLP分析	Jieba + SnowNLP + TF-IDF	中文分词、情感分析与关键词提取
大模型	通义千问 / DeepSeek 	国内可用，接入便捷



3.1 数据采集与NLP处理流程
数据采集与NLP处理模块负责从多个信息源采集舆情数据并进行离线分析，最终产出JSON格式的结构化数据供后端导入。整体流程分为6个阶段：
热榜抓取（crawler.py）：从B站、微博、今日头条、小红书的热搜榜获取热点标题，调用LLM（DeepSeek/OpenAI兼容接口）或规则式方法提取事件关键词；
多源搜索：基于每个关键词，在三个平台并行搜索相关评论内容，保存为raw_comments.csv；
关键词提取：使用Jieba分词 + TF-IDF算法从评论内容中提取高频关键词；
情绪标注（sentiment.py）：对每条评论进行情绪分析，采用词典法+SnowNLP+LLM（仅对可疑样本）标注为positive/neutral/negative。
热度计算（analyzer.py）：使用加权公式 heat_index = like_count × 0.6 + reply_count × 0.4 + |sentiment_score| × 2 计算每条评论的热度指数；
事件聚合：按关键词将评论聚合成舆情事件，统计每个事件的总评论数、平台分布、情感分析和Top关键词，按日汇总，最终输出analysis_result.json。

3.2 前端架构设计
views/         ← 页面层（LoginPage, HomePage, EventBoardPage, EventDetailPage, ProfilePage）
components/     ← 通用组件（PageHeader, ChartPanel, EventCard, charts/*）
layouts/        ← 布局层（AppLayout: 侧栏 + 顶栏 + 内容区）
router/         ← 路由层（嵌套路由 + 懒加载 + 认证守卫）
stores/         ← 状态层（userStore: reactive 简易 Store）
api/            ← 通信层（http.js 主实例 + llm.js LLM 专用实例 + mock.js 离线数据）
assets/         ←展示层，整个前端应用的全局样式文件，负责定义页面的视觉风格、布局结构和响应式适配。

3.3 后端Flask API接口设计
API接口定义了一个具体的地址（URL）和方法（GET/POST等）。所有发往这个地址的请求，都会被这个API接收。

例如，定义了一个 /api/auth/login 的接口:
/api：这层意思是“这里是接口（API）区域”，用来区分网页页面（比如 /home）和纯数据接口。
/auth：这层意思是“这里是认证（Authentication）模块”，表示跟登录、注册、身份验证相关的功能都放在这个目录下。
/login：这层意思是“这里是登录（Login）操作”，明确表示这个地址处理的是“登录”这个具体动作。

JWT认证流程：客户端调用/api/auth/login获取token → 后续请求在Authorization头中携带 Bearer <token> → 后端login_required装饰器验证token有效性并注入g.user_id。

后端API遵循RESTful设计规范，所有接口统一以/api为前缀，响应格式为JSON，字符编码为UTF-8。接口清单如下：
模块	方法	路径	功能说明
认证	POST	/api/auth/login	用户登录，返回JWT令牌(Token)
认证	GET	/api/auth/me	获取当前用户信息
总览	GET	/api/overview/stats	获取首页统计数据
事件	GET	/api/events	事件列表（支持搜索/筛选/排序）
事件	GET	/api/events/<id>	事件基本信息
事件	GET	/api/events/<id>/trend	事件发展趋势数据
事件	GET	/api/events/<id>/sentiment	事件情感分布数据
事件	GET	/api/events/<id>/platforms	事件平台分布数据
事件	GET	/api/events/<id>/word-cloud	事件词云数据
事件	POST	/api/events/<id>/report	生成事件分析报告（回退接口）
个人中心	GET	/api/user/follow-platforms	获取关注平台列表
个人中心	POST	/api/user/follow-platforms	添加关注平台
个人中心	DELETE	/api/user/follow-platforms/<id>	删除关注平台
个人中心	GET/POST/DELETE	/api/user/follow-keywords*	关注关键词CRUD（同平台模式）
LLM服务	POST	/api/llm/ask	智能问答
LLM服务	POST	/api/llm/predict	趋势预测


3.4 数据库设计
数据库采用SQLite（零配置部署，适合中小规模数据），共设计5张核心表：
（1）users表：存储用户账号信息。字段包括id（自增主键）、username（唯一用户名）、password_hash（SHA-256加盐哈希）、role（角色，默认'分析员'）。

（2）events表：存储舆情事件的核心数据。字段包括id（主键，如EVT-001）、title（事件标题）、summary（事件摘要）、source（首发平台）、heat（热度指数）、risk_level（风险等级）、sentiment（情感倾向）、occur_time（发生时间）、update_time（更新时间）、keywords（JSON数组）、trend_data（JSON数组，趋势数据）、sentiment_data（JSON数组，情感分布）、platform_data（JSON数组，平台分布）、wordcloud_data（JSON数组，词云数据）。

（3）daily_stats表：存储每日全局统计数据。字段包括date（主键）、comment_count（评论总数）、positive/negative/neutral（情感分类计数）、heat_index（综合热度指数）。

（4）follow_platforms表：存储用户关注的平台信息。字段包括id（自增主键）、user_id（外键）、name（平台名称）、url（平台URL）、created_at（创建时间）。

（5）follow_keywords表：存储用户关注的关键词。字段包括id（自增主键）、user_id（外键）、word（关键词）、level（关注等级：高/中/低）、created_at（创建时间）。


3.5前后端的交互:
Vue (前端) 负责搭建页面、处理用户的点击和输入。
当用户需要登录或获取数据时，Vue 会发送一个网络请求给 Flask (后端)。
Flask 处理这个请求（比如查数据库），然后把处理结果（如用户信息）以 JSON 数据格式返回给 Vue。
Vue 收到数据后，自动更新页面上的内容，整个过程用户无需刷新浏览器。

3.6 大模型服务设计
大模型服务模块（llm_service包）以Flask Blueprint形式封装，通过/api/llm/*路径提供三个核心API：
POST /api/llm/ask — 智能问答：接收event_data和question参数，先校验问题与事件的关联性（正则匹配检查是否为写代码、问天气等无关问题），通过后调用大模型API。支持四种模式：api（真实大模型）、mock（演示模式）、guard（问题无关拦截）、fallback（API故障降级，到规则化回答），所有模式均返回HTTP 200，前端可统一处理；
POST /api/llm/predict — 趋势预测：接收trend_data数组，调用numpy库，预测未来热度。
POST /api/llm/report — 报告生成：接收event_data，调用大模型生成结构化Markdown报告或使用模板引擎生成固定格式报告。报告严格包含五个部分：事件概述、传播趋势分析、情感倾向分析、风险等级研判、处置建议。风险等级基于负面情感占比和预测趋势综合判定（负面>50%或负面>30%且趋势上升→高风险）。
LLM客户端（llm_client.py）使用Python标准库urllib实现OpenAI兼容的/chat/completions调用，无需第三方HTTP客户端依赖。Prompt管理（prompts.py）集中定义系统提示词，后续调优时无需修改业务代码。

3.7 爬取控制设计
前端通过 POST /api/crawl/trigger 发起请求，后端立刻返回"已接受"，不阻塞用户操作。前端再通过 GET /api/crawl/status 轮询进度，实时展示运行状态。调度器用状态锁保证同一时间只有一个任务在执行，防止重复触发。

调度器收到触发后，开后台线程先跑爬虫脚本（crawler/run_pipeline.py），超时 10 分钟自动终止；爬取成功后再自动调用 data_import.py 把结果 JSON 导入 SQLite 数据库。



设计成果：
<img width="756" height="368" alt="image" src="https://github.com/user-attachments/assets/6df12748-90fb-4050-a14e-df7abd80b8d4" />
<img width="756" height="380" alt="image" src="https://github.com/user-attachments/assets/e17378e0-fa7e-4735-a314-4a139dcf8820" />
<img width="755" height="384" alt="image" src="https://github.com/user-attachments/assets/e6078441-b10c-44d9-bc25-bb616cae71eb" />

