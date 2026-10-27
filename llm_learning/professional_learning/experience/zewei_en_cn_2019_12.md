# Ze Wei / 韦泽

> Extracted from [zewei_en_cn_2019_12.pdf](./zewei_en_cn_2019_12.pdf), a five-page bilingual résumé. Headings, lists, spacing, compatibility glyphs, and line-break hyphenation have been normalized for Markdown. Source wording and conflicting dates, titles, and metrics are retained. "Present" and "至今" refer to the source document, not the current date.

## English Version

### Contact

- **Name:** Ze Wei
- **Mobile:** 15221329512
- **Email:** ze.wei1989@gmail.com
- **Address:** #302, No.31, Lane 201, Shangpu Road, Shanghai, Pudong District, 200124, China.

### Summary

- Solid Algorithm skills, good machine learning foundation and problem solving skills.
- Follow recent trend in machine learning and NLP, experience in model (parallel) training and serving with frameworks including Tensorflow, Pytorch and Scikit-learn.
- Experience in bigdata and stream processing with open source tools including Spark, Kafka, Elasticsearch, OpenTSDB, HBase and MySQL.
- Experience with Python, Java, Go, C#, PHP, C/C++.
- Experience with Linux, Docker and Kubernetes.
- Good software architectural and detail designing skills. Agile process management skills with Scrum.

### Education

| Period | Degree | Institution | Major |
|---|---|---|---|
| 2011 - 2013 | Master | Nanjing University | Software Engineering |
| 2007 - 2011 | Bachelor | Nanjing University | Software Engineering |

### Work Experience

#### Cootek — Staff Algorithm Engineer

**Oct 2018 - present**

Design and develop several systems including calling robots, DMP, product searching & recommending system and speech recognition.

#### Helium2 — Senior Algorithm Engineer

**Feb 2017 - Sept 2018**

Design and develop monitor and analyze system for Kubernetes-based clusters and services.

#### Baidu — Senior Software Engineer

**Jul 2013 - Jan 2017**

Design and develop mobile, desktop and server-side applications for multiple platforms including Android, Windows and Linux.

### Project Experience

#### Advertisement Platform

**Oct 2018 - present**

An ad platform, with bidding systems, data management platform, product/ad search & recommendation system and various prediction models running behind them.

- Build product search engine, besides with intent classification model, CTR/CVR prediction models and item&user embedding models.
- Fine-tune BERT to extract entities and attributes from product titles or descriptions.
- Build online and offline ETL pipelines to collect massive user attributes, actions and feedbacks from various data sources, with billions of data per day.
- Build models for user attributes prediction to increase coverage in real time traffic.
- Build large scale embedding models to sketch user interests and measure item similarities. Item vectors are trained based on sampling from their co-occurrences in user sessions. Users are embedded into the same space based on items they clicked, bought or installed.
- Build large scale Approximate Nearest Neighbor search system based on LSH algorithm, which is used in lookalike user targeting, product recalling and recommending. The approximate nearby join between two datasets with millions lines of rows each can be finished within 20 minutes, more than 5 times performance improvement compared with straightforward approach.
- Build deep&wide model to predict CTR. Aside from hand-crafted features, item embeddings and user embeddings are also included to achieve better generalization. AUC is increased by 0.005, compared with LR baseline.

#### Calling robot

**Oct 2018 - Aug 2019**

A system to make phone calls and multi-round conversations. The main components of the system includes speech recognition, NLU, dialog management and NLG.

- Build real time speech recognition decoder based on Kaldi. Improving multi-thread decoding performance (RTF) by 30%.
- Build Voice Activity Detection model based on Tensorflow, reaching 90.54 in F1, a 10% increase compared with baseline(WebRTC), bigger under noisy enviroments. Also provide online prediction, using CGo to bridge the gap between TF c-lib and golang based service, which has an average prediction latency less than 1ms.
- Build LSTM based auto punctuation model for speech input engine, reaching 69.7 in F1.
- Build calling robots, working in areas including speech recognition, user intent classification and dialog state management.

#### Monitor and analyze system

**Feb 2017 - Sept 2018**

A system to monitor Kubernetes-based clusters. It provides the ability to monitor resource usage on machine and container levels, to trace distributed systems, to collect and analyze application logs and trigger alerts when exceptions are detected.

- Provide different types of data collecting agents to collect machine-level, container-level and app-level data including resource usage, logs, route tracing, performance and business metrics.
- Develop spark-based applications to clean, transform and aggregate data. Fine-tuning spark cluster to provide high-usable data streaming service.
- Mine event patterns from data to continuously extract events from online systems, and furthermore, using Machine Learning methods to build analyzing models to detect anomalies, to predict failures, to analyze correlations and to find root causes.
- Develop distributed tracing system to help track time lines of internal requests and processing procedures, and furthermore, help analyze system-wide bottlenecks, traffic flowing and dependency topology.
- Define data model to store different types of data in OpenTSDB and ElasticSearch.
- Develop rule engine to trigger, group, filter and send out alerts.

#### Baidu Mobile schedule center

**Jan 2015 - Jan 2017**

A system to schedule distributed task executing terminals. The types of terminals include Windows PC, Linux hosts and mobile phones. The types of tasks include server-accessibility checking and mobile app testing (monkey, uiautomator etc.).

- Verify and implement new techniques such as interactive remote mobile controlling system, deep app exploration tool and control locating method based on image recognition.
- Design and develop a WebSocket server to push and receive messages, which provides services to register and schedule devices from both data center and crowd-sourcing users.
- Develop common framework on client-side, including task execution and thread pool scheduling.
- Continuously optimize overall performance.
- Apply Scrum to the team and continuously improve the development process for better productivity and code quality.

#### Baidu Crowd-sourcing Platform

**Sept 2013 - Nov 2014**

The general outdoor data collection platform for Baidu Crowd Test (http://test.baidu.com One of the biggest crowd-sourcing platforms in the world), based on Android.

- Develop web tools to collect datasets for machine learning programs.
- Design and develop a dynamic UI configuration system which makes it possible to publish different forms of collecting jobs without updating the app.
- Develop Restful-APIs to support job publishing, data uploading and other supporting functions.
- Provide SDK to support third parties to build and integrate their own data collection plug-ins to our platform with low effort.
- Develop automatic integration platform to increase team productivity.

## 中文版本

### 联系方式

- **姓名：** 韦泽
- **手机：** 15221329512
- **邮箱：** ze.wei1989@gmail.com

### 教育背景

- 南京大学，软件工程，硕士，2011 - 2013
- 南京大学，软件工程，本科，2007 - 2011

### 专业技能

机器学习算法基础扎实，持续跟进传统机器学习、深度学习和NLP等领域前沿进展，热爱人工智能理论及应用。(https://weize07.github.io/)

熟练使用Tensorflow、Keras、Pytorch、Scikit-learn等框架，手写过常见的机器学习方法，掌握训练、调参及部署技巧，有分布式数据管道、并行训练的经验，能够定位错误和调优性能瓶颈。

擅长大数据处理及流式计算，具有Spark，Kafka，ElasticSearch，OpenTSDB，MySQL等开源技术栈开发经验。

使用多门编程语言，包括Python，Java，Go，PHP，C#，C/C++等。熟练使用Kubernetes，Linux，Docker等部署环境及工具，有较丰富的应用程序架构及面向对象程序设计经验，熟悉Scrum等敏捷流程。英文阅读写作能力强，又语流利。

### 工作&项目经历

#### 触宝科技

**就职：触宝科技　2017.2 ~ 2018.9　职位：资深算法工程师**

**2018.10 ~ 至今　AILAB&广告事业部**

从事NLP、语音识别、商品检索系统、人群画像等方面工作。

重点工作内容：

- 商品检索系统搭建，用户意图识别，点击、转化率预估模型，商品嵌入预训练模型，微调BERT模型做实体识别，进行主干词、修饰词提取等。
- 用户画像系统搭建，收集性别、年龄、职业、内容点击阅读行为、广告点击行为、App使用行为等各方面的数据，原始数据每天亿级别。
- 基于用户行为的基础属性扩展，其中性别模型F1 85+，年龄70+，学生识别85+
- 基于App使用、广告点击、和内容点击三种数据，进行用户向量嵌入；基于用户内共现采样做商品、App的嵌入；并把用户向量、item向量作为特征引入deep&wide模型，将CTR模型AUC提升5个千分位。
- 在嵌入空间进行相似度度量，基于spark开发大规模LSH近邻搜索，几十分钟内完成百万级数据集间之间的近邻交叉匹配，单次查询用时到毫秒级别。用于lookalike人群扩展、相似商品召回等场景。
- 基于kaldi进行语音识别系统解码器搭建，并进行性能优化，RTF提升30%。
- 基于Tensorflow开发TDNN的VAD模型，F1达到90.54，较基线webrtc提升10%以上，在低信噪比环境下提升更加显著，且使用cgo加载的模型预测延迟低于1ms，满足高实时性要求。
- 开发基于LSTM的自动断句及标点符号预测模型，F1达到69.7左右。
- 参与语音外呼机器人系统搭建，涉及语音识别、用户意图识别、对话状态管理等。

#### 新氦大数据

**就职：新氦大数据　2017.2 ~ 2018.9　职位：高级算法工程师**

**2017.2 ~ 2018.9　智能监控平台**

搭建基于Kubernetes微服务云平台的一体化监控平台。包括数据采集，数据转化与清洗，实时监控报警，应用多维度智能分析，持久化与报表系统等。

重点工作内容：

- 数据采集模块，接入多种异构数据源。采集机器级别、容器级别、应用级别数据，包括资源使用、日志、调用链路、性能、业务指标等不同类型。
- 基于Spark集群搭建离线、在线计算平台，进行各类格式转换、数据清洗、聚合运算任务。调优Spark性能，充分发挥其水平扩展和自恢复能力。
- 对应用日志进行自动事件模版抽取及事件挖掘，并构建时序数据预测模型进行异常检测、故障预判、关联性分析、根因分析等。
- 分布式链路追踪系统，用于全链路性能分析、集群流量流转分析、应用拓扑关联等。
- 基于Kafka的数据总线进行限流、数据流整合和负载均衡
- 基于OpenTSDB和ElasticSearch进行持久存储。
- 监控报表系统，报警规则引擎，自愈系统。

#### 百度

**就职：百度　2013.7 ~ 2017.2　职位：软件工程师**

##### 百度云App任务调度中心

**2015.1 ~ 2017.2**

参与百度云的移动应用测试调度中心的架构和研发工作。期间获评百度“季度之星”。

重点工作内容：

- 新技术验证与落地，包括跨平台手机远程操控，应用遍历，基于图像识别的手机控件定位等。
- 系统架构与开发，微服务规划和长短连接混合架构，用事件驱动和异步IO的方式提供高吞吐及低延时的任务下发系统。
- 整合技术栈异构的系统，合理运用反向代理做服务代理与转发。
- 机房、分布式挂机用户手机资源调度，整合任务执行框架和任务调度系统。
- 长连接程序结构设计，模块划分，接又抽象与任务线程池基础建设。
- 敏捷流程管理，参与外部合作、技术交流，组织技术分享与积累。

##### 百度众测平台

**2013.9 ~ 2016.10**

参与搭建全球最大的众包平台之一的百度众测平台。包括场内外的数据采集工具的开发工作。期间获评百度代码规范Java good coder，并申请专利《监控服务质量的方法、服务端、客户端及系统》

重点工作内容：

- 图像识别、自动驾驶、搜索引擎优化标注数据采集，提供过亿级别的标注数据。
- 移动端外场采集众包App，支持界面组件化及全面可配置。
- 移动端第三方性能监控任务发布平台。
- 持续集成及性能优化，社交分享、消息推送等基础设施建设。

## Extraction Notes

- The Chinese Cootek employment heading says `2017.2 ~ 2018.9`, but its project heading and the English employment entry start in October 2018.
- The English Baidu employment and scheduling project end in January 2017; the Chinese entries end in February 2017.
- The English crowd-sourcing project ends in November 2014; the Chinese project ends in October 2016.
- The English and Chinese ETL descriptions use different volume scales: billions per day and hundreds of millions per day, respectively.
- The extracted strings `又语流利` and `接又抽象` are retained. Their intended readings may be `口语流利` and `接口抽象`; the differences could come from the source text or its PDF character mapping.
