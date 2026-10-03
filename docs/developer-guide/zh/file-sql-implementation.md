# File SQL实现

## 1. 目标与范围

File SQL 为 SQL 查询提供直接读取外部文件的能力。查询通过关系表达式引用文件，由数据库负责解析 SQL、检查访问权限、读取文件并把记录转换为行。

本文记录当前的 CSV 单文件查询实现，并为后续文件格式和查询能力保留章节。当前阶段不包含自动 schema 推断、类型转换、文件通配符、多文件扫描、JSON、Parquet 或 `COPY`。

### 1.1 当前实现

- 输入：一个 CSV 文件的绝对路径。
- SQL 入口：`READ_CSV(path) AS relation_name (column, ...)`。
- 列定义：调用者显式提供列名；列值当前统一作为 `VARCHAR` 返回。
- CSV 行为：逗号分隔、双引号字段、双引号转义、换行记录；支持跳过首行标题。
- 执行方式：按需流式读取，不把整个文件载入内存。
- 安全：解析并规范化路径，遵守 `secure_file_priv`；查询 CSV 文件表需要全局 `FILE` 权限。

## 2. CSV 单文件查询

### 2.1 SQL 语法与示例

基本语法：

```sql
SELECT select_list
FROM READ_CSV('/absolute/path/data.csv') AS csv_data (id, name)
[WHERE predicate];
```

示例：

假设 `/tmp/users.csv` 内容如下：

```csv
id,name
1,Alice
2,Bob
```

```sql
SELECT id, name
FROM READ_CSV('/tmp/users.csv') AS users (id, name)
WHERE id = '1';
```

查询结果：

```text
+----+-------+
| id | name  |
+----+-------+
| 1  | Alice |
+----+-------+
1 row in set
```

列名列表是必需的。列值按 CSV 字段原样输出为字符串；需要数值或日期语义时，查询方可显式使用 SQL 转换函数。路径当前要求为字符串字面量，并且必须是可访问的文件路径。

### 2.2 SQL 到结果行的执行链路

1. **词法与语法分析**：MySQL 模式语法识别 `READ_CSV(path) AS alias (columns)`，构造关系表达式节点。当前复用 `T_TABLE_COLLECTION_EXPRESSION` 节点，并以三个子节点分别承载路径、关系别名和列名列表。
2. **语义解析与安全检查**：DML resolver 校验路径为字符串常量，解析并规范化路径，检查 `secure_file_priv`，将规范化路径保存到常量节点；随后构造带 CSV 标记的 function table 和 VARCHAR 输出列。
3. **权限检查**：查询引用 CSV 文件表时检查全局 `FILE` 权限。
4. **逻辑计划与代码生成**：`TableItem` 保存 CSV 标记和列数；function table 逻辑算子向下传递 CSV 属性；代码生成阶段将 CSV 列映射写入物理算子 spec。
5. **物理执行**：function table 算子打开文件，逐行解析 CSV，并根据投影列映射输出所需字段。扫描是流式的，过滤条件继续由常规 SQL 执行计划处理。

### 2.3 主要实现位置

| 层次 | 文件 | 职责 |
| --- | --- | --- |
| 语法 | `src/sql/parser/sql_parser_mysql_mode.y`、`src/sql/parser/non_reserved_keywords_mysql_mode.c` | 声明 `READ_CSV` 语法及关键字 |
| 解析与权限边界 | `src/sql/resolver/dml/ob_dml_resolver.cpp` | 校验字面量路径和列清单、规范路径、应用 `secure_file_priv`、创建 CSV function table |
| 表元数据 | `src/sql/resolver/dml/ob_dml_stmt.h`、`src/sql/resolver/dml/ob_dml_stmt.cpp` | 保存 CSV 标记和 CSV 列数 |
| 逻辑计划 | `src/sql/optimizer/ob_log_function_table.h`、`src/sql/optimizer/ob_log_plan.cpp` | 传递 CSV 扫描属性 |
| 代码生成 | `src/sql/code_generator/ob_static_engine_cg.cpp` | 生成物理算子所需的列索引映射和配置 |
| 物理执行 | `src/sql/engine/basic/ob_function_table_op.h`、`src/sql/engine/basic/ob_function_table_op.cpp` | 持有扫描 spec、打开文件、生成结果行和重扫 |
| 文件读取 | `src/sql/engine/cmd/ob_load_data_file_reader.h`、`src/sql/engine/cmd/ob_load_data_file_reader.cpp` | CSV 流式读取器及文件随机读取器的 seek 状态处理 |
| 权限 | `src/sql/privilege_check/ob_privilege_check.cpp` | 为 CSV 文件查询检查全局 `FILE` 权限 |

### 2.4 读取与行映射

`CsvScanReader` 复用现有 CSV 解析能力，以记录为单位读取数据，并支持跳过标题行及重新扫描。单条记录限制为 64 MiB，避免无界缓冲单行输入。物理 function table 根据代码生成阶段提供的列索引选择字段；输出列当前全部为 `VARCHAR`。

文件随机读取器的 `seek()` 会校验初始化状态和偏移，并清除 EOF 状态，使重扫可以从目标位置继续读取。扫描器和文件句柄由物理算子生命周期持有，算子关闭时释放。

### 2.5 错误与约束

- 非字面量路径、缺少列名列表或无效列定义在解析/语义解析阶段报错。
- 路径无法规范化、违反 `secure_file_priv` 或文件打不开时，查询不能开始扫描。
- 缺少全局 `FILE` 权限时，查询在权限检查阶段失败。
- CSV 列数与声明列数不匹配时，按现有解析和列访问错误路径处理；具体错误码与兼容性应由 sqltest 固化。
- 当前格式选项固定，不支持用户指定分隔符、转义符、编码或空值规则。

### 2.6 SQL 测试

端到端用例位于：

- `tools/deploy/mysql_test/test_suite/file_sql/t/read_csv_single_file.test`
- `tools/deploy/mysql_test/test_suite/file_sql/r/mysql/read_csv_single_file.result`

用例准备单个 CSV 文件，并覆盖完整扫描、只投影部分列和过滤条件。该用例已添加，但在当前环境中尚未运行；构建与 sqltest 验证待具备相应工具链后完成。

### 2.7 当前限制

- 仅支持单个 CSV 文件，路径为绝对路径字符串字面量。
- 必须由 SQL 显式声明列名，不做 header 推断或 schema 推断。
- 所有字段均按字符串返回，不做类型推断或自动转换。
- CSV 格式选项固定；复杂、多行引号字段和异常输入的行为取决于现有 CSV parser 能力。
- 不支持 `READ_CSV` 与写入、事务快照或外部文件变更检测之间的协调。
- 尚未完成可执行构建、sqltest 运行及性能验证。

## 3. JSON 文件查询（预留）

本章用于后续设计和实现 JSON 文件查询，当前尚未实现。

建议在落地前明确：JSON Lines 与普通 JSON 文档的支持范围、嵌套对象展开规则、数组处理、列路径表达式、类型推断和错误行策略。实现后补充 SQL 语法、resolver 校验、物理扫描器、权限策略和 sqltest 覆盖。

## 4. Parquet 文件查询（预留）

本章用于后续设计和实现 Parquet 查询，当前尚未实现。

需要补充所采用的 Parquet 读取依赖及构建集成、schema 到 SQL 类型的映射、列裁剪和谓词下推、压缩编码支持、文件元数据读取、资源生命周期及错误处理，并通过包含多种类型和编码的测试文件验证。

## 5. 多文件与路径模式查询（预留）

本章用于后续实现 glob、目录和多文件扫描，当前尚未实现。

实现前需要定义路径展开规则、文件排序、空目录行为、文件间 schema 一致性、单文件失败策略、并发扫描和取消行为。权限检查必须覆盖路径展开后实际打开的每个文件。

## 6. 文件写入与 COPY（预留）

本章用于后续实现文件导入/导出语句，例如 `COPY`，当前尚未实现。

需要分别定义导入与导出的语法、目标覆盖策略、原子性、错误行处理、格式选项、权限要求和失败清理规则。文件写入属于外部副作用，需明确事务失败时文件是否保留及如何避免覆盖非目标文件。

## 7. 文件访问安全（预留扩展）

CSV 查询当前已在 resolver 路径上应用 `secure_file_priv` 检查，并在权限检查阶段要求全局 `FILE` 权限。本章后续用于统一各种文件格式和读写操作的安全策略。

后续需覆盖规范化路径与符号链接、目录边界、路径模式展开、服务端身份下的文件访问、读写权限区分、错误信息中的路径泄露，以及多租户场景下的访问隔离。各文件格式和操作都应复用统一的路径校验入口。

## 8. Schema、类型与格式选项（预留）

本章用于后续记录 schema 推断、显式 schema、类型转换、默认值、NULL/空字符串区别、编码和格式参数设计。当前 CSV 查询要求显式列名，字段统一以 `VARCHAR` 返回，且不开放格式选项。

## 9. 测试、验证与性能（预留）

当前 CSV 测试覆盖基本端到端查询路径。后续应在项目既有 SQL 测试目录中补齐权限拒绝、路径边界、空文件、标题行、引号与转义、CRLF、字段数不匹配、超大记录、重扫、并发连接及文件错误等场景。

构建和验证记录应注明构建配置、测试命令、实际运行结果及未覆盖项。性能验证应关注大文件下的内存上界、吞吐、投影列裁剪收益和过滤下推能力。
