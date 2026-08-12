# 07 · jdbc 模块学习要点

> 基于 `src/com/interface21/jdbc/**`（55 个 `.java`）与 `test/com/interface21/jdbc/**`（21 个）逐字核对编写。
> JDBC 抽象层是 Spring 数据访问思想的集大成者。

## ⚠️ 阅读前必读：0.9.1 与现代 Spring 的命名/接口差异

| 现代 Spring | 0.9.1 实际 |
|-------------|------------|
| `RowMapper` | **不存在**；行映射由 `RowCallbackHandler` / `ResultReader`，以及 `MappingSqlQuery.mapRow(ResultSet,int)` 承担 |
| `PreparedStatementCallback` | **不存在**；对应 `PreparedStatementCreator` + `PreparedStatementSetter` |
| `SQLExceptionTranslator` / `ResultSetExtractor` / `SQLErrorCodeSQLExceptionTranslator` | 拼写为 **`Translater`** / **`Extracter`** / **`SQLErrorCodeSQLExceptionTranslater`**（`-er` 而非 `-or`） |
| `queryForObject` / `queryForList` | **不存在** |
| `query(...)` 返回 List/Map | 0.9.1 的 `query` **返回 `void`**（结果靠回调累积） |
| `DataSourceUtils.releaseConnection` | 叫 **`closeConnectionIfNecessary`** |

这些命名差异本身就是很好的面试谈资：**说明你真读过老代码而非照搬现代 Spring**。

## 模块定位与子包结构

| 子包 | 职责 | 代表类 |
|------|------|--------|
| `jdbc/core` | 底层模板与回调、异常翻译器、SQLException 配置 | `JdbcTemplate`、`RowCallbackHandler`、`SQLExceptionTranslater` |
| `jdbc/core/support` | core 辅助：DAO 基类、主键自增器、JDBC BeanFactory | `JdbcDaoSupport`、`AbstractDataFieldMaxValueIncrementer` |
| `jdbc/datasource` | DataSource 实现、线程绑定连接、JDBC 事务管理器 | `DataSourceUtils`、`DriverManagerDataSource`、`DataSourceTransactionManager` |
| `jdbc/object` | RDBMS 操作对象（命令式 OO 包装） | `RdbmsOperation`、`SqlQuery`、`StoredProcedure` |
| `jdbc/util` | 纯工具 | `JdbcUtils` |

> 主键自增器的**接口**在 `jdbc/core/`，**实现**在 `jdbc/core/support/`——接口与实现分离的典型布局。

## 核心回调接口（逐字签名）

### `PreparedStatementCreator` — `jdbc/core/PreparedStatementCreator.java:28`
```java
public interface PreparedStatementCreator {
    PreparedStatement createPreparedStatement(Connection conn) throws SQLException;
    String getSql();
}
```

### `PreparedStatementSetter` — `jdbc/core/PreparedStatementSetter.java:33`
```java
public interface PreparedStatementSetter {
    void setValues(PreparedStatement ps) throws SQLException;
}
```

### `BatchPreparedStatementSetter` — `jdbc/core/BatchPreparedStatementSetter.java:31`
```java
public interface BatchPreparedStatementSetter {
    void setValues(PreparedStatement ps, int i) throws SQLException;
    int getBatchSize();
}
```

### `RowCallbackHandler` — `jdbc/core/RowCallbackHandler.java:23`（**参数是 `ReadOnlyResultSet`，不是 `ResultSet`**）
```java
public interface RowCallbackHandler {
    void processRow(ReadOnlyResultSet rs) throws SQLException;
}
```

### `ResultSetExtracter`（注意 `-er`）— `jdbc/core/ResultSetExtracter.java:21`
```java
public interface ResultSetExtracter {
    void extractData(ResultSet rs) throws SQLException;
}
```

### `ResultReader` — `jdbc/core/ResultReader.java:20`
```java
public interface ResultReader extends RowCallbackHandler {
    List getResults();
}
```
是 `SqlQuery` 体系的桥梁（execute 用它收集结果）。

### `SQLExceptionTranslater` — `jdbc/core/SQLExceptionTranslater.java:29`（异常翻译策略接口）
```java
public interface SQLExceptionTranslater {
    DataAccessException translate(String task, String sql, SQLException sqlex);
}
```

## 核心实现类

### `JdbcTemplate` — `jdbc/core/JdbcTemplate.java:71`（本包灵魂）
实现 `InitializingBean`，本身即模板。关键字段：`DataSource`、`boolean ignoreWarnings = true`、`SQLExceptionTranslater exceptionTranslater`（懒加载）。

**公开方法签名（逐字，注意 query 返回 void）**：
```java
public void query(String sql, RowCallbackHandler callbackHandler) throws DataAccessException;                       // :206
public void query(PreparedStatementCreator psc, RowCallbackHandler callbackHandler) throws DataAccessException;     // :261
public void query(final String sql, final PreparedStatementSetter pss, RowCallbackHandler callbackHandler) ...;     // :310
public void doWithResultSetFromStaticQuery(String sql, ResultSetExtracter rse) throws DataAccessException;          // :220
public void doWithResultSetFromPreparedQuery(PreparedStatementCreator psc, ResultSetExtracter rse) ...;             // :273
public int update(final String sql) throws DataAccessException;              // :349
public int update(PreparedStatementCreator psc) throws DataAccessException;  // :362
public int[] update(PreparedStatementCreator[] pscs) throws DataAccessException;  // :373
public int update(final String sql, final PreparedStatementSetter pss) ...;  // :414
public int[] batchUpdate(String sql, BatchPreparedStatementSetter setter) throws DataAccessException;  // :442
public void afterPropertiesSet();   // :178
```

### 异常翻译器（策略模式多实现）
| 类 | 翻译依据 |
|----|----------|
| `SQLStateSQLExceptionTranslater`（:31） | 用 SQLState 前两位（类码）做**可移植**翻译。静态两组 Set：`BAD_SQL_CODES={"07","42","65","S0"}`、`INTEGRITY_VIOLATION_CODES={"22","23","27","44"}` |
| `SQLErrorCodeSQLExceptionTranslater`（:29） | 用厂商专有 `errorCode` 做**精确但厂商相关**翻译；匹配不上时**降级委托** `SQLStateSQLExceptionTranslater`（:69-70） |
| `OracleSQLExceptionTranslater`（`support/:33`） | 硬编码 Oracle 错误码 switch |
| `SQLExceptionTranslaterFactory`（:43） | **单例工厂**：读 `sql-error-codes.xml`，按 `DatabaseMetaData.getDatabaseProductName()` 选实现 |

`SQLErrorCodes`（:17）是 JavaBean，含 `badSqlGrammarCodes`、`dataIntegrityViolationCodes` 两个 `String[]`。配置文件 `jdbc/core/sql-error-codes.xml` 预置 DB2/HSQL/MS-SQL/MySQL/Oracle 五库错误码。

### `ReadOnlyResultSet` — `jdbc/core/ReadOnlyResultSet.java:40`（0.9.1 独有）
**不是 `ResultSet` 子类**，而是**代理包装器**：`getXxx()` 委托内部 `ResultSet`；导航/修改类方法（`next()`/`close()`/`first()`/`updateXxx()`…）抛 `InvalidResultSetMethodInvocationException`。目的：防止回调用户破坏 JdbcTemplate 的遍历驱动。

### `RowCountCallbackHandler`（:28）
`RowCallbackHandler` 便利基类。`processRow` 为 **final**（:50），首行读 `ResultSetMetaData` 缓存列信息，再回调子类 `protected void processRow(ResultSet rs, int rowNum)`（:72）。

## datasource 包实现

### `DataSourceUtils` — `jdbc/datasource/DataSourceUtils.java:45`（abstract，全静态）
**资源管理与线程绑定的中枢**：
```java
public static ThreadObjectManager getThreadObjectManager();                                                       // :60
public static boolean isConnectionBoundToThread(Connection con, DataSource ds);                                    // :71
public static DataSource getDataSourceFromJndi(String jndiName) throws CannotGetJdbcConnectionException;           // :86
public static Connection getConnection(DataSource ds) throws CannotGetJdbcConnectionException;                     // :127
public static void closeConnectionIfNecessary(Connection con, DataSource ds) throws CannotCloseJdbcConnectionException;  // :150
static Connection getCloseSuppressingConnectionProxy(Connection source);   // :174 包级
```
- `getConnection`（:127-139）：先从 `ThreadObjectManager` 取该 DataSource 的 `ConnectionHolder`，命中→返回其连接（当前事务的连接）；未命中→`ds.getConnection()`。
- `closeConnectionIfNecessary`（:150-164）：已绑定线程→不关；`SmartDataSource.shouldClose()` 返回 false→不关；否则才 `close()`。
- 内部类 `CloseSuppressingInvocationHandler`（:185）用 JDK 动态代理拦截 `close()`。

### DataSource 实现与事务管理器
| 类 | 要点 |
|----|------|
| `SmartDataSource`（:18） | `interface extends DataSource { boolean shouldClose(Connection conn); }` |
| `AbstractDataSource`（:24） | "无趣"方法的默认实现基类 |
| `DriverManagerDataSource`（:41） | `extends AbstractDataSource implements SmartDataSource`；每次 `getConnection()` 都返回新连接（`shouldClose` 恒 true）；`Class.forName` 加载驱动 |
| `SingleConnectionDataSource`（:33） | `extends DriverManagerDataSource implements DisposableBean`；整个生命周期只用一个物理连接（非线程安全，测试用）；`shouldClose` 恒 false；`suppressClose=true` 时用代理屏蔽 `close()` |
| `ConnectionHolder`（:20） | 包 `Connection` + `rollbackOnly`，SPI 类 |
| `DataSourceTransactionObject`（:14） | 事务对象，持 `ConnectionHolder` + `previousIsolationLevel` |
| `DataSourceTransactionManager`（:43） | `extends AbstractPlatformTransactionManager implements InitializingBean`——单数据源本地事务管理器（详见 [08-transaction.md](08-transaction.md)） |

## object 包：RDBMS 操作对象

继承层级：
```
RdbmsOperation (abstract, implements InitializingBean)
├── SqlOperation (abstract)
│   ├── SqlQuery (abstract)
│   │   ├── MappingSqlQueryWithParameters (abstract)
│   │   │   └── MappingSqlQuery (abstract)        ← 用户最常继承
│   │   │       └── SqlFunction                   ← 具体类，SQL 单值函数
│   │   └── ReflectionExtractionSqlQuery (abstract)  ← DEMO，未完整实现
│   └── SqlUpdate                                  ← 具体类，更新
└── StoredProcedure (abstract)         ← 存储过程（不经 SqlOperation）
```

### 关键类
| 类 | 要点 |
|----|------|
| `RdbmsOperation`（:49） | 持 `DataSource`/`sql`/`declaredParameters`/`compiled`。**编译后禁止再 declareParameter/setTypes**；`compile()`（:182）是 **final** 模板方法，调子类 `compileInternal()` |
| `SqlOperation`（:30） | `compileInternal`（:68，final）：用 `JdbcUtils.countParameterPlaceholders` 数 `?`（有限状态机，跳过引号字面量），与声明参数数不符则抛异常 |
| `SqlQuery`（:38） | 所有执行汇到 `public final List execute(final Object[] parameters, Map context)`（:116）；子类实现 `protected abstract ResultReader newResultReader(int rowsExpected, Object[] parameters, Map context)`（:103）；提供 `findObject(...)`（结果非唯一即抛异常） |
| `MappingSqlQueryWithParameters`（:50） | `newResultReader` 返回内部 `ResultReaderImpl`（:97）；`processRow` 调子类 `protected abstract Object mapRow(ResultSet rs, int rowNum, Object[] parameters, Map context)`（:90）。类注释讲"手动映射优于自动反射（帕累托法则）" |
| `MappingSqlQuery`（:34） | 简化版：`protected abstract Object mapRow(ResultSet rs, int rowNum)`（:72）。**用户最常继承**——`mapRow` 的角色即后世 `RowMapper`（但接口本身当时不存在） |
| `SqlUpdate`（:34） | `public int update(Object[] args)`（:127）调 JdbcTemplate.update，再按 `maxRowsAffected`/`requiredRowsAffected` 校验 |
| `StoredProcedure`（:42） | 不经 SqlOperation，直接 extends RdbmsOperation。`compileInternal`（:144）拼 `{call NAME(?,?,?)}` 或 `{? = call NAME(?)}`；**强制参数带名字**；`execute(Map inParams)`（:182）直接用 JDBC `CallableStatement`，不经 JdbcTemplate |
| `SqlFunction`（:44） | extends MappingSqlQuery，封装"单行单列"SQL 函数，多于一行抛异常 |

## support 与 util 包

**主键自增器**（模板方法模式，Javadoc 自述）：
```java
// jdbc/core/DataFieldMaxValueIncrementer.java:19
public interface DataFieldMaxValueIncrementer {
    int nextIntValue() throws DataAccessException;
    long nextLongValue() throws DataAccessException;
    double nextDoubleValue() throws DataAccessException;
    String nextStringValue() throws DataAccessException;
    Object nextValue(Class keyClass) throws DataAccessException;
}
```
抽象实现 `AbstractDataFieldMaxValueIncrementer` 的 `nextIntValue/...` 都 **final**，调子类 `protected abstract int incrementIntValue()`。三个具体实现：`OracleSequenceMaxValueIncrementer`（`seq.NEXTVAL`）、`MySQLMaxValueIncrementer`（`last_insert_id`）、`HsqlMaxValueIncrementer`（`identity()`）。

| 类 | 要点 |
|----|------|
| `JdbcDaoSupport`（support/:26） | DAO 推荐基类：`setDataSource` 里 `new JdbcTemplate`，`afterPropertiesSet` 调 `initDao()` 钩子 |
| `JdbcBeanFactory`（support/:26） | `implements ListableBeanFactory`，从 DB 表读 `(beanName,property,value)` 构造 Bean 定义——**JdbcTemplate 的第一个非平凡用户** |
| `JdbcUtils`（util/:12） | `countParameterPlaceholders`（有限状态机数 SQL 占位符）、`isNumeric`、`translateType` |

## 模板方法模式：JdbcTemplate 如何消除样板

以 `query(String sql, RowCallbackHandler)` 为例（`JdbcTemplate.java:206-251`）完整流程：
1. **获取连接（事务感知）**：`con = DataSourceUtils.getConnection(this.dataSource);`（:230）
2. **创建语句**：`s = con.createStatement();`（:231）
3. **执行**：`rs = s.executeQuery(sql);`（:232）
4. **回调用户代码（唯一可变部分）**：`rse.extractData(rs);`（:237）——内部 `RowCallbackHandlerResultSetExtracter`（:491）把 `RowCallbackHandler` 适配成 `ResultSetExtracter`，**用 `ReadOnlyResultSet` 包装**（:510），`while(rs.next()) callbackHandler.processRow(rors);`
5. **处理警告**：取 `SQLWarning`，`throwExceptionOnWarningIfNotIgnoringWarnings`（:239-243）
6. **异常翻译**：`catch (SQLException ex) { throw getExceptionTranslater().translate(...); }`（:245-247）
7. **释放资源**：`finally { DataSourceUtils.closeConnectionIfNecessary(con, this.dataSource); }`（:248-250）

用户代码因此缩减为"提供 SQL + 一个 `processRow` 回调"。`RowCallbackHandlerResultSetExtracter`（:491）是**适配器模式**范例。

## 异常翻译机制（面试高频）

**两层翻译 + 工厂选择**：
1. **工厂选翻译器**：`SQLExceptionTranslaterFactory.getDefaultTranslater(DataSource)` 取 `DatabaseMetaData.getDatabaseProductName()`，在 `sql-error-codes.xml` 查（DB2 名前缀特殊处理），命中→`SQLErrorCodeSQLExceptionTranslater`；未命中→`SQLStateSQLExceptionTranslater`。
2. **错误码翻译**（`SQLErrorCodeSQLExceptionTranslater.translate`，:54-71）：`Arrays.binarySearch` 在 `badSqlGrammarCodes`→`BadSqlGrammarException`；否则在 `dataIntegrityViolationCodes`→`DataIntegrityViolationException`；**都不命中→委托 `SQLStateSQLExceptionTranslater` 兜底**（降级策略）。
3. **SQLState 翻译**（:57-73）：取前两位类码，在 `BAD_SQL_CODES`→`BadSqlGrammarException`；在 `INTEGRITY_VIOLATION_CODES`→`DataIntegrityViolationException`；否则→`UncategorizedSQLException`。

设计精髓：**可移植性（SQLState）与精确性（errorCode）的二选一与降级**——先精确，不中再可移植。

## 关键设计模式

| 模式 | 类 |
|------|----|
| 模板方法 | `JdbcTemplate.query/update/...`；`RdbmsOperation.compile()`→`compileInternal()`；`AbstractDataFieldMaxValueIncrementer`（Javadoc 自述） |
| 策略 | `SQLExceptionTranslater` + 三实现 |
| 工厂/单例 | `SQLExceptionTranslaterFactory`、`PreparedStatementCreatorFactory` |
| 适配器 | `RowCallbackHandlerResultSetExtracter`（:491） |
| 代理（JDK 动态代理） | `CloseSuppressingInvocationHandler`（屏蔽 `close()`） |
| 包装器 | `ReadOnlyResultSet` |
| 回调 | 全部回调接口 |
| OO 命令对象 | `RdbmsOperation`/`SqlQuery`/`SqlUpdate`/`StoredProcedure` |

## 对应测试（`test/com/interface21/jdbc/`）

| 测试 | 覆盖 |
|------|------|
| `core/JdbcTemplateTestSuite` | 最核心（~1000 行，EasyMock + MockObjects）：query/update/batchUpdate、异常翻译、SQLWarning、连接失败/关闭失败、**`testSQLErrorCodeTranslation`（:956）验证 MySQL errorCode 1054→`BadSqlGrammarException`** |
| `core/SqlStateExceptionTranslaterTestSuite` | SQLState 翻译 + null/空/单字符 SQLState 健壮性 |
| `core/ReadOnlyResultSetTestSuite` | 禁用方法测试 |
| `object/RdbmsOperationTestSuite` | 编译校验：空 dataSource/sql、编译后改参数、参数数量不符 |
| `object/StoredProcedureTestSuite` | 存储过程：OutputParameter、不存在的 SP→`BadSqlGrammarException`、事务内执行、自定义翻译器 |
| `object/SqlQueryTestSuite` / `SqlUpdateTestSuite` / `SqlFunctionTestSuite` | 对象包装用法 |
| `datasource/DataSourceUtilsTests` | close 抑制代理 |
| `datasource/DataSourceTransactionManagerTests` | 事务管理器：commit/rollback/嵌套/隔离级别/超时/参与已有事务 |
| `core/support/JdbcBeanFactoryTests`、`util/JdbcUtilsTests` 等 | 辅助 |

> `livetest/com/interface21/jdbc/`（2 个文件）是连真实 MySQL 的集成测试，不进单测套件，仅作参考。

## 建议学习顺序

1. **dao 异常体系**（见 [06-dao.md](06-dao.md)）
2. **jdbc/core 回调接口**：`PreparedStatementCreator`/`Setter`/`RowCallbackHandler`/`ResultSetExtracter`/`ResultReader`（注意 `ReadOnlyResultSet` 签名）
3. **`JdbcTemplate`**：从 `doWithResultSetFromStaticQuery`（:220）和 `RowCallbackHandlerResultSetExtracter`（:491）入手
4. **异常翻译器**：`SQLExceptionTranslater` → `SQLState...` → `SQLErrorCode...` → `Factory` + `sql-error-codes.xml` + `SQLErrorCodes`
5. **`DataSourceUtils` + datasource 包**：线程绑定连接、`SmartDataSource`、`SingleConnectionDataSource` 的 close 抑制代理、`DataSourceTransactionManager`
6. **object 包**：`RdbmsOperation`→`SqlOperation`→`SqlQuery`→`MappingSqlQueryWithParameters`→`MappingSqlQuery`（重点 `mapRow`）→`SqlUpdate`→`StoredProcedure`
7. **support/util**：主键自增器、`JdbcDaoSupport`、`JdbcBeanFactory`、`JdbcUtils.countParameterPlaceholders`

## 值得精读的源码片段

| 位置 | 为什么 |
|------|--------|
| `JdbcTemplate.java:doWithResultSetFromStaticQuery`（:220） | 模板方法完整骨架，最简洁 |
| `JdbcTemplate.java:RowCallbackHandlerResultSetExtracter`（:491） | 适配器 + ReadOnlyResultSet 包装 + 行遍历，一处看懂三个设计 |
| `JdbcTemplate.java:batchUpdate`（:442） | JDBC 2.0 批量更新模板封装 |
| `SQLErrorCodeSQLExceptionTranslater.java:translate`（:54） | 错误码翻译 + 降级委托 |
| `SQLExceptionTranslaterFactory.java:getDefaultTranslater`（:119） | 基于元数据选策略、单例工厂 |
| `ReadOnlyResultSet.java:next/close/first`（:75,84,497） | 包装器禁止危险方法 |
| `DataSourceUtils.java:getConnection`（:127）与 `closeConnectionIfNecessary`（:150） | 线程绑定连接取/还核心 |
| `DataSourceUtils.java:CloseSuppressingInvocationHandler`（:185） | JDK 动态代理屏蔽 close |
| `DataSourceTransactionManager.java:doBegin/doCommit/closeConnection`（:103,130,170） | 声明式 JDBC 事务完整实现 |
| `RdbmsOperation.java:compile`（:182）与 `validateParameters`（:211） | "编译"机制 + 参数校验 |
| `SqlOperation.java:compileInternal`（:68） | 占位符数与声明参数数一致性检查 |
| `MappingSqlQueryWithParameters.java:ResultReaderImpl`（:97） | ResultReader 典型实现，List 累积 |
| `StoredProcedure.java:execute(ParameterMapper)`（:203） | 存储过程直接用 CallableStatement + 翻译器 |
| `JdbcUtils.java:countParameterPlaceholders`（:47） | 有限状态机解析 SQL 占位符 |

---

**一句话总结**：0.9.1 的 jdbc 已把 Spring 数据访问**全部核心思想**立起来——模板方法 + 回调消除样板、运行时异常层次、可插拔异常翻译策略（含降级）、线程绑定连接的事务集成、OO 命令对象。但**命名有差异**（`Translater`/`Extracter`/`-er`）、**接口集合更小**（无 `RowMapper`/`PreparedStatementCallback`/`queryForObject`）、**query 返回 void**、引入了 `ReadOnlyResultSet` 这种后来被放弃的防误用包装器。这些差异本身就是最有价值的面试谈资。

← 上一节：[06-dao.md](06-dao.md) ｜ 下一节：[08-transaction.md](08-transaction.md)
