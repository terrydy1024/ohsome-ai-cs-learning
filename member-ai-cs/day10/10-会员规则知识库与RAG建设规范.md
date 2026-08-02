# Day 10｜会员规则知识库与 RAG 建设规范

## 1. 目标与边界

会员客服知识库解决的是“当前有效规则怎么解释”，不是“会员本人有多少积分”。个人资产必须来自会员 API；RAG 仅用于检索积分、等级、优惠券、权益、活动和异常处理规则。

一期目标：
- 高频会员问题有权威、当前有效、可追溯的知识依据；
- 检索前按市场、业务线、渠道、语言、版本和权限做过滤；
- 答案中的关键结论可以回到具体知识片；
- 没有有效证据时停止回答，而不是依靠模型常识补全。

## 2. 知识源分级

- A：正式制度、会员条款、已审批活动规则；
- B：官方客服 SOP、业务处理标准；
- C：官方 FAQ、标准话术；
- D：历史会话和运营经验，只用于发现问题，不直接作为线上事实。

发生冲突时，优先级不是由向量相似度决定，而是先按有效性和权威等级过滤、排序。

## 3. 知识入库流程

1. 收集原始规则并确定内容负责人；
2. 结构化为“主题、条件、结论、例外、时间、适用对象”；
3. 写入市场、业务线、渠道、语言、版本、生失效时间、权限等元数据；
4. 按文档结构进行语义切片；
5. 建立关键词和向量索引；
6. 用固定检索测试集做离线回归；
7. 审批后在指定生效时间发布；
8. 线上点踩、无答案和误答回灌测试集。

## 4. 切片原则

切片不是越小越好，也不是统一按固定字数截断。一个合格知识片应能独立表达一个可回答的规则单元，并保留：规则主题、适用市场、对象、条件、限制、例外和版本。

- 正式规则：按标题和原子规则切；
- FAQ：一问一答；
- SOP：按步骤和判断分支；
- 表格：保留表头和单位后按语义行切；
- 活动规则：活动 ID、时间、资格、奖励和渠道不能分散到无法关联的片段。

## 5. 检索链路

```text
用户问题
→ 识别主题/市场/语言/渠道
→ 权限、市场、业务线、状态和生效时间硬过滤
→ 查询改写与同义词扩展
→ 关键词 + 向量混合召回
→ 权威、时效、相关性重排
→ 最低证据阈值
→ 答案生成与逐结论引用
→ 无证据拒答/转人工
```

## 6. 错误定位

- 没有正确知识：内容缺口；
- 有知识但未召回：切片、查询、同义词、TopK；
- 召回了错误国家/旧版：元数据和过滤；
- 正确片在后排：精排和权威特征；
- 正确证据但回答错误：生成忠实度；
- 引用与结论不匹配：引用绑定；
- 没证据仍回答：拒答和护栏。

## 7. 验收标准

不能只看“回答准确率”。至少分层检查：知识覆盖率、过滤正确率、关键证据召回、前排精度、忠实度、引用正确率、正确拒答率和端到端问题解决率。

## 8. 当前假设

本方案中的文档量、切片大小、TopK 和阈值属于学习项目参数，不应在面试中描述成真实生产配置。真实上线需要使用实际知识文档和咨询数据调优。

## 9. 参考资料
- OpenAI Vector Stores API: https://platform.openai.com/docs/api-reference/vector-stores
- Azure AI Search RAG overview: https://learn.microsoft.com/en-us/azure/search/retrieval-augmented-generation-overview
- Azure RAG chunk enrichment: https://learn.microsoft.com/en-us/azure/architecture/ai-ml/guide/rag/rag-enrichment-phase
- Azure vector query filters: https://learn.microsoft.com/en-us/azure/search/vector-search-filters
- Azure document-level access control: https://learn.microsoft.com/en-us/azure/search/search-document-level-access-overview
