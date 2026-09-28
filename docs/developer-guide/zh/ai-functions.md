# AI 模型管理与 SQL 函数实现

本文依据当前仓库源码说明 `DBMS_AI_SERVICE` 的四个模型管理过程，以及 `AI_EMBED`、`AI_RERANK`、`AI_COMPLETE`、`AI_PROMPT` 四个 SQL 函数。以下示例使用 MySQL 模式。模型服务的 URL、密钥和模型名均为占位值；调用型函数需要先配置可用的外部服务端点。

## 整体调用流程

1. `CREATE_AI_MODEL` 将逻辑模型名、模型类型和服务端模型名写入 AI 模型 schema。
2. `CREATE_AI_MODEL_ENDPOINT` 将逻辑模型关联到服务端点，保存 URL、密钥、服务商等配置。
3. 执行 `AI_EMBED`、`AI_RERANK` 或 `AI_COMPLETE` 时，表达式按逻辑模型名获取模型信息和端点，按服务商协议组装请求，通过 HTTP POST 调用服务并解析响应。
4. `AI_PROMPT` 在 SQL 中构造 `{ "template": ..., "args": [...] }` JSON 对象；`AI_COMPLETE` 接收该对象时才展开 `{0}`、`{1}` 等占位符。

系统包的 SQL 声明和 C++ 入口分别位于 [包定义](../../../src/share/inner_table/sys_package/dbms_ai_service_mysql.sql)、[包体绑定](../../../src/share/inner_table/sys_package/dbms_ai_service_body_mysql.sql) 和 [过程实现](../../../src/pl/sys_package/ob_dbms_ai_service.cpp)。三个外部调用函数共用的模型信息、服务商请求和响应处理逻辑位于 [AI 函数工具](../../../src/sql/engine/expr/ob_expr_ai/ob_ai_func_utils.cpp)。

## DBMS_AI_SERVICE.CREATE_AI_MODEL

**功能。** 创建一个逻辑 AI 模型。第一个参数是模型名，第二个参数是 JSON 对象，必须包含 `type` 和 `model_name`。`type` 支持 `dense_embedding`、`sparse_embedding`、`completion` 和 `rerank`；本文涉及的三个调用型 SQL 函数分别使用 `dense_embedding`、`completion` 和 `rerank`。

**使用示例。**

```sql
CALL DBMS_AI_SERVICE.CREATE_AI_MODEL(
  'demo_embed', '{"type":"dense_embedding","model_name":"embedding-model"}'
);

SELECT NAME, TYPE, MODEL_NAME
FROM oceanbase.DBA_OB_AI_MODELS
WHERE NAME = 'demo_embed';
```

**实现方式。** 过程检查 `CREATE AI MODEL` 权限、参数个数和模型是否已存在，解析并校验 JSON，然后通过 root command service 发起模型 DDL；rootserver 将模型 schema 持久化。参见 [过程实现](../../../src/pl/sys_package/ob_dbms_ai_service.cpp)、[参数解析与类型校验](../../../src/share/ai_service/ob_ai_service_struct.cpp) 和 [模型 DDL 服务](../../../src/rootserver/ob_ai_model_ddl_service.cpp)。

## DBMS_AI_SERVICE.CREATE_AI_MODEL_ENDPOINT

**功能。** 为已创建的逻辑模型配置外部服务端点。JSON 参数中的 `ai_model_name` 指向逻辑模型；`url`、`access_key`、`provider` 分别指定服务地址、访问密钥和服务商。可选的 `request_model_name` 会覆盖调用请求中使用的 `model_name`。当前端点校验还要求 `scope` 为 `ALL`，省略时默认取 `ALL`；`parameters`、`request_transform_fn`、`response_transform_fn` 当前不能设置为非空值。

**使用示例。** 先执行上面的 `CREATE_AI_MODEL` 示例，再创建端点：

```sql
CALL DBMS_AI_SERVICE.CREATE_AI_MODEL_ENDPOINT(
  'demo_embed_endpoint',
  '{"ai_model_name":"demo_embed","url":"https://api.example.com/v1/embeddings","access_key":"replace-with-real-key","provider":"openai","request_model_name":"embedding-model"}'
);

SELECT ENDPOINT_NAME, AI_MODEL_NAME, URL, PROVIDER, REQUEST_MODEL_NAME
FROM oceanbase.DBA_OB_AI_MODEL_ENDPOINTS
WHERE ENDPOINT_NAME = 'demo_embed_endpoint';
```

**实现方式。** 过程检查 `CREATE AI MODEL` 权限并解析 JSON，调用端点管理接口；服务端执行端点创建与校验。支持的 `provider` 名称由校验代码限定为 `aliyun-openai`、`aliyun-dashscope`、`deepseek`、`siliconflow`、`cohere`、`hunyuan-openai` 和 `openai`，大小写不敏感。参见 [过程实现](../../../src/pl/sys_package/ob_dbms_ai_service.cpp)、[端点参数校验](../../../src/share/ai_service/ob_ai_service_struct.cpp) 和 [端点管理接口](../../../src/query/api/query/ai/ob_ai_endpoint_admin.h)。

## DBMS_AI_SERVICE.DROP_AI_MODEL

**功能。** 删除指定名称的逻辑 AI 模型。

**使用示例。** 在不再使用模型时执行：

```sql
CALL DBMS_AI_SERVICE.DROP_AI_MODEL('demo_embed');
```

**实现方式。** 过程检查 `DROP AI MODEL` 权限及模型是否存在，再通过 root command service 发起删除 DDL，最终删除模型 schema。参见 [过程实现](../../../src/pl/sys_package/ob_dbms_ai_service.cpp) 和 [模型 DDL 服务](../../../src/rootserver/ob_ai_model_ddl_service.cpp)。

## DBMS_AI_SERVICE.DROP_AI_MODEL_ENDPOINT

**功能。** 删除指定名称的模型端点。

**使用示例。** 在清理模型前可先删除端点：

```sql
CALL DBMS_AI_SERVICE.DROP_AI_MODEL_ENDPOINT('demo_embed_endpoint');
CALL DBMS_AI_SERVICE.DROP_AI_MODEL('demo_embed');
```

**实现方式。** 过程检查 `DROP AI MODEL` 权限和端点名，然后调用端点管理接口删除端点。参见 [过程实现](../../../src/pl/sys_package/ob_dbms_ai_service.cpp) 和 [端点管理接口](../../../src/query/api/query/ai/ob_ai_endpoint_admin.h)。

## AI_EMBED

**功能。** 使用 `dense_embedding` 模型将文本转换成向量。签名为 `AI_EMBED(model_name, content[, dimension])`；可选的 `dimension` 必须是正整数。返回值类型是 `VARCHAR`，内容为 JSON 数组形式的向量字符串，例如 `[0.12,-0.34,...]`。实际维度和数值取决于外部服务。

**使用示例。** 假设 `demo_embed` 已绑定可用端点：

```sql
SELECT AI_EMBED('demo_embed', '数据库向量检索') AS embedding;
SELECT AI_EMBED('demo_embed', '数据库向量检索', 1024) AS embedding;
```

**实现方式。** [表达式实现](../../../src/sql/engine/expr/ob_expr_ai/ob_expr_ai_embed.cpp) 检查参数、解析模型和端点，并把可选维度放入请求配置的 `dimensions` 字段。共用 [AI 函数工具](../../../src/sql/engine/expr/ob_expr_ai/ob_ai_func_utils.cpp) 校验模型为 `dense_embedding`，生成服务商请求，发送 HTTP POST，解析向量数组并转成字符串；指定维度时会检查返回数组的长度。

## AI_RERANK

**功能。** 使用 `rerank` 模型按查询相关性重排候选文档。签名为 `AI_RERANK(model_name, query, documents[, doc_key])`，返回 JSON。`documents` 必须是非空 JSON 数组：不传 `doc_key` 时元素为字符串；传 `doc_key` 时元素为对象，`doc_key` 指明对象中包含文档文本的字段。

**使用示例。** 假设已经创建并配置名为 `demo_rerank` 的 `rerank` 模型：

```sql
-- 返回按分数排序的服务结果，元素包含 index 和 relevance_score。
SELECT AI_RERANK(
  'demo_rerank',
  '什么是向量索引？',
  JSON_ARRAY('向量索引用于相似度检索', '数据库支持事务')
) AS ranked_results;

-- 返回重新排序的原始对象，并为每个对象添加 _model_score。
SELECT AI_RERANK(
  'demo_rerank',
  '什么是向量索引？',
  JSON_ARRAY(
    JSON_OBJECT('id', 1, 'text', '向量索引用于相似度检索'),
    JSON_OBJECT('id', 2, 'text', '数据库支持事务')
  ),
  'text'
) AS ranked_documents;
```

**实现方式。** [表达式实现](../../../src/sql/engine/expr/ob_expr_ai/ob_expr_ai_rerank.cpp) 校验输入类型、解析模型和端点，然后通过 [AI 函数工具](../../../src/sql/engine/expr/ob_expr_ai/ob_ai_func_utils.cpp) 调用 rerank 服务。不传 `doc_key` 时最多每 20 个文档一批，按 `relevance_score` 合并批次结果；传 `doc_key` 时提取对象中的文本，按服务返回的 `index` 排序原对象，并写入 `_model_score`。

## AI_COMPLETE

**功能。** 使用 `completion` 模型根据提示词生成文本。签名为 `AI_COMPLETE(model_name, prompt[, config])`，返回 `LONGTEXT`。`prompt` 可以是普通字符串，也可以是 `AI_PROMPT` 生成的 JSON 对象；可选的 `config` 是 JSON 配置，其具体字段由服务商协议决定。

**使用示例。** 假设已经创建并配置名为 `demo_chat` 的 `completion` 模型：

```sql
SELECT AI_COMPLETE('demo_chat', '用一句话解释向量索引') AS answer;

SELECT AI_COMPLETE(
  'demo_chat',
  AI_PROMPT('用一句话解释{0}', '向量索引'),
  '{"temperature":0.2}'
) AS answer;
```

**实现方式。** [表达式实现](../../../src/sql/engine/expr/ob_expr_ai/ob_expr_ai_complete.cpp) 检查参数并解析模型和端点。若 `prompt` 是 JSON，则先确认其含 `template` 和字符串 `args`，再展开占位符。共用 [AI 函数工具](../../../src/sql/engine/expr/ob_expr_ai/ob_ai_func_utils.cpp) 校验模型为 `completion`，按服务商协议生成请求、发送 HTTP POST、解析文本响应。

## AI_PROMPT

**功能。** 构造可传给 `AI_COMPLETE` 的提示词对象。签名为 `AI_PROMPT(template[, arg1, ...])`，返回 JSON 对象，包含 `template` 和 `args` 两个字段。模板中的 `{0}`、`{1}` 等数字占位符按参数位置展开；展开在 `AI_COMPLETE` 中发生。模板和参数必须是字符串类型，JSON 类型参数不受支持。

**使用示例。**

```sql
SELECT AI_PROMPT('请用一句话解释{0}，并给出{1}个例子', '向量索引', '2') AS prompt;

SELECT AI_COMPLETE(
  'demo_chat',
  AI_PROMPT('请用一句话解释{0}', '向量索引')
) AS answer;
```

第一个查询生成的对象在结构上等同于 `{"template":"请用一句话解释{0}，并给出{1}个例子","args":["向量索引","2"]}`；JSON 字段顺序由实现决定。

**实现方式。** [表达式实现](../../../src/sql/engine/expr/ob_expr_ai/ob_expr_ai_prompt.cpp) 校验参数类型，收集字符串参数并构造 JSON 对象。[占位符处理](../../../src/sql/engine/expr/ob_expr_ai/ob_ai_func_utils.cpp) 在 `AI_COMPLETE` 消费该对象时替换 `{数字}`；索引超出参数范围会报错。

## 适用范围与源码依据

- 以上说明依据源码进行静态核对；示例中的外部服务调用需要将占位 URL、密钥和模型名替换为真实配置。
- 创建、删除模型及端点的 SQL 用法可参考 [模型 DDL 测试](../../../tools/deploy/mysql_test/test_suite/ai_function/t/ai_model_ddl.test) 和 [端点 DDL 测试](../../../tools/deploy/mysql_test/test_suite/ai_function/t/ai_model_endpoint_ddl.test)；`AI_PROMPT` 的参数边界可参考 [提示词测试](../../../tools/deploy/mysql_test/test_suite/ai_function/t/ai_prompt.test)。
- 系统包还提供 `ALTER_AI_MODEL_ENDPOINT`，但本文聚焦上述八个 API。
