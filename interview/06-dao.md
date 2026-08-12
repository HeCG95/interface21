# 06 · dao 模块学习要点

> 基于 `src/com/interface21/dao/**`（11 个 `.java`，无子包）逐字核对编写。dao 模块无专属测试。
> 这是最"小而美"的模块——一套与具体数据访问技术无关的**异常层次**。

## 模块定位

dao 不依赖 JDBC/Hibernate/iBatis 任何一种技术，只定义**统一的异常语义**。其价值有三：

1. **检查异常 → 运行时异常**：根类 `DataAccessException` 继承 `com.interface21.core.NestedRuntimeException`，业务代码不再被迫 `throws SQLException` 或写空 catch。
2. **语义化分类**：把"连不上库""SQL 语法错""完整性约束违反""乐观锁失败"等抽象成统一类型，调用方"不知道底层是 JDBC"也能按类型 catch。
3. **跨技术栈一致**：同一套异常既能被 JDBC 翻译器抛出，也能被 Hibernate/JDO 翻译器抛出。

Rod Johnson 在源码 Javadoc 中明确指向其著作《Expert One-On-One J2EE Design and Development》第 9 章。

## 异常层次（全部 11 个类）

根：`DataAccessException`（**abstract**）— `src/com/interface21/dao/DataAccessException.java:31`
```java
public abstract class DataAccessException extends NestedRuntimeException {
    public DataAccessException(String msg);
    public DataAccessException(String msg, Throwable ex);
}
```

子类（均直接或间接继承 `DataAccessException`，全部 unchecked）：

| 类名 | 父类 | 语义 | 构造签名 |
|------|------|------|----------|
| `DataAccessResourceFailureException` | `DataAccessException` | 资源彻底失败（连不上库） | `(String, Throwable)` |
| `DataIntegrityViolationException` | `DataAccessException` | 违反完整性约束（唯一键等） | `(String, Throwable)` |
| `DeadlockLoserDataAccessException` | `DataAccessException` | 死锁失败者 | `(String, Throwable)` |
| `OptimisticLockingFailureException` | `DataAccessException` | 乐观锁失败 | `(String)` / `(String, Throwable)` |
| `TypeMismatchDataAccessException` | `DataAccessException` | Java 类型与 DB 类型不匹配 | `(String, Throwable)` |
| `CleanupFailureDataAccessException` | `DataAccessException` | 操作成功但收尾（关连接）失败 | `(String, Throwable)` |
| `InvalidDataAccessApiUsageException` | `DataAccessException` | 框架 API 被误用（未编译就执行） | `(String)` / `(String, Throwable)` |
| `InvalidDataAccessResourceUsageException` | `DataAccessException` | 错误使用资源的根（如 SQL 语法错） | `(String)` / `(String, Throwable)` |
| `IncorrectUpdateSemanticsDataAccessException`（**abstract**） | `InvalidDataAccessResourceUsageException` | 更新语义不符（该更新 1 行却 3 行） | 含抽象方法 `public abstract boolean getDataWasUpdated()` |
| `UncategorizedDataAccessException`（**abstract**） | `DataAccessException` | 无法精分类别的兜底层 | `(String, Throwable)` |

### 设计细节
- `DataAccessException`、`UncategorizedDataAccessException`、`IncorrectUpdateSemanticsDataAccessException` 是 **abstract**，强迫用更具体的子类。
- `IncorrectUpdateSemanticsDataAccessException` 自带抽象方法 `getDataWasUpdated()`——表达"语义不符但数据是否已被改动（要不要回滚）"，非常细的语义。
- 所有异常通过 `NestedRuntimeException` 保留 `Throwable` 根因（`getRootCause()`）。

## JDBC 对 dao 体系的扩展

JDBC 包在 dao 根之下追加更具体的异常（路径在 `jdbc/core` 与 `jdbc/datasource`）：

| 类 | 继承自 | 路径 |
|----|--------|------|
| `BadSqlGrammarException` | `InvalidDataAccessResourceUsageException` | `jdbc/core/BadSqlGrammarException.java:24` |
| `JdbcUpdateAffectedIncorrectNumberOfRowsException` | `IncorrectUpdateSemanticsDataAccessException` | `jdbc/core/...:12` |
| `UncategorizedSQLException` | `UncategorizedDataAccessException` | `jdbc/core/...:23` |
| `SQLWarningException` | `UncategorizedDataAccessException` | `jdbc/core/...:16` |
| `InvalidResultSetMethodInvocationException` | `InvalidDataAccessApiUsageException` | `jdbc/core/...:12` |
| `CannotGetJdbcConnectionException` | `DataAccessResourceFailureException` | `jdbc/datasource/...:19` |
| `CannotCloseJdbcConnectionException` | `CleanupFailureDataAccessException` | `jdbc/datasource/...:22` |

`BadSqlGrammarException`（:24-60）额外保存 `SQLException ex` 与 `String sql`，提供 `getSQLException()` / `getSql()`——这是翻译器产出的"有上下文"异常典范。

> 异常翻译的完整机制见 [07-jdbc.md](07-jdbc.md)。

## 关键设计模式与亮点

1. **统一异常层次（异常分类树）**：用继承表达"错误语义的泛化-特化"。
2. **检查→运行时（Uncheck）**：继承 `NestedRuntimeException`，把 JDBC 的 checked `SQLException` 转成 unchecked，消除模板 catch。
3. **模板方法的根因保留**：所有异常通过 `HasRootCause.getRootCause()` 暴露原始异常，不丢上下文。
4. **抽象类强制具体化**：根类与兜底类设为 abstract，引导使用者用语义更准确的子类。

## 对应测试

dao 模块**无专属测试**。其异常类型在 jdbc 包测试（`JdbcTemplateTestSuite.testSQLErrorCodeTranslation` 等）中被验证翻译结果。

## 建议学习顺序

dao 文件少且短，建议一口气通读，按"根 → 资源失败类 → 资源误用类 → 完整性/锁类 → 兜底类"顺序：
1. `DataAccessException`（根，abstract）
2. `DataAccessResourceFailureException` / `CleanupFailureDataAccessException`
3. `InvalidDataAccessResourceUsageException` / `InvalidDataAccessApiUsageException`
4. `DataIntegrityViolationException` / `OptimisticLockingFailureException` / `DeadlockLoserDataAccessException`
5. `UncategorizedDataAccessException`（兜底，abstract）

读完立刻接 [07-jdbc.md](07-jdbc.md) 看这些异常如何被翻译器生产出来。

## 值得精读的源码片段

- **`DataAccessException.java:31`** —— 整个异常体系的根，体会"为什么继承 NestedRuntimeException 而非 Exception"。
- **`IncorrectUpdateSemanticsDataAccessException.java`** —— 罕见的"异常里带抽象方法 `getDataWasUpdated()`"，体现对事务回滚决策的细致考虑。
- **`jdbc/core/BadSqlGrammarException.java:24-60`** —— 翻译器产出的"携带 SQL + 原始 SQLException"的异常典范。

---

← 上一节：[05-aop.md](05-aop.md) ｜ 下一节：[07-jdbc.md](07-jdbc.md)
