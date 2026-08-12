# 08 · transaction 模块学习要点

> 基于 `src/com/interface21/transaction/**`（32 个 `.java`）与 `test/com/interface21/transaction/**`（10 个）逐字核对编写。
> 此版与现代 Spring 差异极大，多处"常识"在此版**不成立**。

## ⚠️ 面试要点速记（针对 0.9.1，务必区别现代版）

- **传播行为只有 3 种**：`PROPAGATION_REQUIRED` / `PROPAGATION_SUPPORTS` / `PROPAGATION_MANDATORY`。**没有** REQUIRES_NEW / NOT_SUPPORTED / NEVER / NESTED（不是 7 种！）。
- **`TransactionStatus` 是类，不是接口**；没有 `SavepointManager`、没有 `flush()`。
- **`TransactionSynchronizationManager` 不绑资源**——它只管同步回调；真正的资源绑定在 `util.ThreadObjectManager`，且各 Utils 类各持一个实例（非全局唯一）。
- **没有 `@Transactional` 注解**；声明式事务靠 AOP Alliance 拦截器 + `TransactionAttributeSource` 策略 + 属性编辑器配置字符串。
- `AbstractPlatformTransactionManager` 的 `getTransaction/commit/rollback` 三大入口都是 **final**，子类只能实现 `doXxx`——教科书级模板方法。

## 模块定位

抽象出一套**与具体事务策略（JTA / JDBC / Hibernate / JDO）无关**的统一编程模型。上层业务代码只面向 `PlatformTransactionManager`，切换底层事务实现仅需改配置。与 [orm](09-orm.md) 的衔接点：各 `*TransactionManager` 继承 `AbstractPlatformTransactionManager`；ORM 资源（Session/PersistenceManager）通过 `ThreadObjectManager` 与线程绑定，让事务管理器与 Template/Interceptor 共享同一份资源。

## 子包结构

```
transaction/
├── PlatformTransactionManager.java       核心策略接口
├── TransactionDefinition.java            传播/隔离/超时/只读 常量与接口
├── TransactionStatus.java                事务状态（⚠ 类，非接口）
├── TransactionException.java             异常体系根（abstract）
├── （8 个具体异常类）
├── interceptor/                          声明式事务（AOP）
│   ├── TransactionInterceptor.java       AOP Alliance 方法拦截器
│   ├── TransactionAttribute / TransactionAttributeSource
│   ├── MapTransactionAttributeSource / AttributeRegistryTransactionAttributeSource
│   ├── DefaultTransactionAttribute / RuleBasedTransactionAttribute
│   └── RollbackRuleAttribute / NoRollbackRuleAttribute / *Editor
├── jta/    JtaTransactionManager.java
└── support/                              编程式事务 + 同步管理
    ├── AbstractPlatformTransactionManager.java   模板方法基类
    ├── DefaultTransactionDefinition.java
    ├── TransactionTemplate.java          编程式模板
    ├── TransactionCallback / TransactionCallbackWithoutResult
    ├── TransactionSynchronization.java   回调接口
    └── TransactionSynchronizationManager.java   线程同步（仅回调，不绑资源！）
```

## 核心接口（逐字签名）

### `PlatformTransactionManager` — 事务抽象的"策略接口"
```java
TransactionStatus getTransaction(TransactionDefinition definition) throws TransactionException;
void commit(TransactionStatus status) throws TransactionException;
void rollback(TransactionStatus status) throws TransactionException;
```

### `TransactionDefinition` — 常量逐字核对（⚠ 仅 3 种传播行为）
```java
String PROPAGATION_CONSTANT_PREFIX = "PROPAGATION";
String ISOLATION_CONSTANT_PREFIX  = "ISOLATION";

int PROPAGATION_REQUIRED  = 0;   // 没有就新建
int PROPAGATION_SUPPORTS  = 1;   // 没有就非事务执行
int PROPAGATION_MANDATORY = 2;   // 没有就抛异常
// ↑ 仅此 3 种

int ISOLATION_DEFAULT          = -1;
int ISOLATION_READ_UNCOMMITTED = Connection.TRANSACTION_READ_UNCOMMITTED;
int ISOLATION_READ_COMMITTED   = Connection.TRANSACTION_READ_COMMITTED;
int ISOLATION_REPEATABLE_READ  = Connection.TRANSACTION_REPEATABLE_READ;
int ISOLATION_SERIALIZABLE     = Connection.TRANSACTION_SERIALIZABLE;

int TIMEOUT_DEFAULT = -1;

int  getPropagationBehavior();
int  getIsolationLevel();
int  getTimeout();
boolean isReadOnly();
```

### `TransactionStatus` — ⚠ 是具体类不是接口
```java
public class TransactionStatus {
    public TransactionStatus(Object transaction, boolean newTransaction);
    public Object getTransaction();          // 底层事务对象（如 JTA UserTransaction）
    public boolean isNewTransaction();       // 新事务 vs 参与已有事务
    public void setRollbackOnly();           // 编程式标记只回滚
    public boolean isRollbackOnly();
}
```
`isNewTransaction()` 实现：`return (transaction != null && newTransaction);`

### `TransactionAttribute` — 在 Definition 之上加"按异常回滚"
```java
public interface TransactionAttribute extends TransactionDefinition {
    boolean rollbackOn(Throwable ex);
}
```

### `TransactionAttributeSource` — 策略：如何为某方法找到事务属性
```java
TransactionAttribute getTransactionAttribute(MethodInvocation invocation);
```

### `TransactionCallback` / `TransactionSynchronization`
```java
// support/TransactionCallback.java
Object doInTransaction(TransactionStatus status);
// support/TransactionSynchronization.java（常量 STATUS_COMMITTED=0 / STATUS_ROLLED_BACK=1 / STATUS_UNKNOWN=2）
void afterCompletion(int status);
```

## 核心实现类

### 事务管理器（策略模式的具体策略）
| 类 | 路径 | 要点 |
|----|------|------|
| `AbstractPlatformTransactionManager` | `support/` | **模板方法基类**。`getTransaction/commit/rollback` 均 **final**，统一处理传播判断、非事务回退、rollbackOnly、同步回调触发；留 5 个 `doXxx` 抽象方法给子类 |
| `JtaTransactionManager` | `jta/` | 通过 JNDI 查 `java:comp/UserTransaction`；**不支持自定义隔离级别**（抛 `InvalidIsolationException`）、忽略 readOnly、超时用 `ut.setTransactionTimeout()` |
| `HibernateTransactionManager` | `orm/hibernate/` | 绑 `SessionHolder`；可选把 Session 的 JDBC `Connection` 也绑给 DataSource（**允许 Hibernate 与裸 JDBC 在同一事务混用**）；支持隔离级别（`session.connection()`）、**不支持超时** |
| `JdoTransactionManager` | `orm/jdo/` | 绑 `PersistenceManagerHolder`；**既不支持隔离级别也不支持超时** |
| `DataSourceTransactionManager` | `jdbc/datasource/` | 单 JDBC DataSource 本地事务管理器（见 [07-jdbc.md](07-jdbc.md)） |

### 编程式事务：`TransactionTemplate`
`extends DefaultTransactionDefinition`。`execute` 完整逻辑（模板方法）：
```java
public Object execute(TransactionCallback action) throws TransactionException {
    TransactionStatus status = this.transactionManager.getTransaction(this);  // this 即 Definition
    Object result = null;
    try { result = action.doInTransaction(status); }
    catch (RuntimeException ex) { performRollback(status, ex); throw ex; }
    catch (Error err)           { performRollback(status, err); throw err; }
    this.transactionManager.commit(status);
    return result;
}
```
回调抛 `RuntimeException` 或 `Error` 触发回滚并上抛；正常则提交。`TransactionCallbackWithoutResult` 是无返回值便捷子类。支持按常量名配置（`setPropagationBehaviorName`）。

## 声明式事务机制（TransactionInterceptor 与 AOP 的协作）

`TransactionInterceptor`（`interceptor/TransactionInterceptor.java`）实现 **AOP Alliance 的 `MethodInterceptor`**，本身不做事务控制，只做编排。

**`invoke` 调用链（逐行逻辑）**：
1. `TransactionAttribute transAtt = this.transactionAttributeSource.getTransactionAttribute(invocation);`——找属性，`null` 表示非事务方法
2. 若 `transAtt != null`：`status = this.transactionManager.getTransaction(transAtt);` 并通过 `invocation.addAttachment(TRANSACTION_STATUS_ATTACHMENT_NAME, status)` 把 status 挂到 AOP 调用上下文（供目标方法内 `TransactionInterceptor.currentTransactionStatus()` 取用，实现编程式 setRollbackOnly）
3. `retVal = invocation.proceed();`——调调用链下一环（通常目标方法）
4. 抛异常→`onThrowable()`：若 `txAtt.rollbackOn(ex)` 为真则 `rollback`，否则仍 `commit`（除非 rollbackOnly）
5. finally 撤销 attachment；正常返回则 `commit(status)`

**`rollbackOn` 默认行为**（`DefaultTransactionAttribute`）：`return (t instanceof RuntimeException);`——EJB 语义，仅运行时异常回滚。

**与 AOP 的关系**：`TransactionInterceptor` 是普通 AOP 拦截器，需配合 `ProxyFactory`（见测试）创建代理。属性获取策略化：`AttributeRegistryTransactionAttributeSource`（元数据属性）、`MapTransactionAttributeSource`（Method→Attribute 映射）。

**回滚规则**（`RuleBasedTransactionAttribute`）：按异常类名深度匹配（`RollbackRuleAttribute.getDepth`），最近匹配（depth 最小）胜出；可用 `TransactionAttributeEditor` 解析配置字符串 `PROPAGATION_*,ISOLATION_*,readOnly,+Ex,-Ex`。

## 资源与事务同步（⚠ 0.9.1 与现代版最大架构差异）

**0.9.1 的 `TransactionSynchronizationManager` 只管"同步回调"，不管"资源绑定"。** 资源绑定由另一个类承担。

### `TransactionSynchronizationManager`（仅同步回调）
`support/TransactionSynchronizationManager.java`——abstract，全静态，内部仅一个 `ThreadLocal`（值为 `List`）：
- `init()`：开事务时由 `AbstractPlatformTransactionManager` 调，`set(new ArrayList())`
- `isActive()`：判断是否激活
- `register(TransactionSynchronization)`：资源管理代码注册回调
- `triggerAfterCompletion(int status)`：事务完成后逐个回调
- `clear()`：`set(null)`

### `ThreadObjectManager`（真正的资源绑定）
`util/ThreadObjectManager.java`——每个工具类各持一个静态实例，`ThreadLocal<Map>` 保存 key→value：
```java
public boolean hasThreadObject(Object key);
public Object  getThreadObject(Object key);
public void    bindThreadObject(Object key, Object value);   // 重复绑定抛 IllegalStateException
public void    removeThreadObject(Object key);
```
- `SessionFactoryUtils.getThreadObjectManager()`（key=SessionFactory → SessionHolder）
- `PersistenceManagerFactoryUtils.getThreadObjectManager()`（key=PMF → PersistenceManagerHolder）
- `DataSourceUtils.getThreadObjectManager()`（key=DataSource → ConnectionHolder）

**协作流程（以 HibernateTransactionManager.doBegin 为例）**：
1. `doGetTransaction`：先查 `SessionFactoryUtils` 的 ThreadObjectManager，有则复用 `SessionHolder`；无则新开 Session
2. `doBegin`：`session.beginTransaction()`，若新 Holder 则 `bindThreadObject(sessionFactory, sessionHolder)`；若设了 dataSource 还会把 `session.connection()` 绑给 DataSource
3. 此后 `HibernateTemplate.execute` / `HibernateInterceptor.invoke` 通过 `SessionFactoryUtils.getSession` 取到**同一个线程绑定 Session**，实现"一个事务一个 Session"
4. `doCommit/doRollback` 后 `closeSession`：仅当新 Holder 才 `removeThreadObject` + 关闭
5. **JTA 场景**：`SessionFactoryUtils.closeSessionIfNecessary` 检测到 `TransactionSynchronizationManager.isActive()` 时不立即关，而是 `register(new SessionSynchronization(...))` 并绑线程，等 JTA 事务 `afterCompletion` 回调再关——这是"Hibernate + JTA 也能正确管理 JVM 级缓存"的实现

## 异常体系（已逐文件核对，全部 unchecked）
```
NestedRuntimeException (com.interface21.core)
└── TransactionException (abstract)
    ├── CannotCreateTransactionException
    │   └── NestedTransactionNotPermittedException
    ├── HeuristicCompletionException
    ├── TransactionSystemException
    ├── UnexpectedRollbackException
    └── TransactionUsageException
        ├── InvalidTimeoutException
        ├── InvalidIsolationException
        └── NoTransactionException
```

## 关键设计模式

| 模式 | 类 |
|------|----|
| 策略 | `PlatformTransactionManager` + 各实现。Javadoc 原话 "Uses the Strategy design pattern" |
| 模板方法 | `AbstractPlatformTransactionManager`（final 入口 + 抽象 `doXxx`）；`TransactionTemplate.execute`、`HibernateTemplate.execute` |
| 拦截器/代理（AOP） | `TransactionInterceptor`、`HibernateInterceptor`、`JdoInterceptor` |
| 回调 | `TransactionCallback`、`HibernateCallback`、`JdoCallback` |
| FactoryBean | `LocalSessionFactoryBean`、`LocalPersistenceManagerFactoryBean` |
| 属性编辑器 | `TransactionAttributeEditor`、`TransactionAttributeSourceEditor` |
| 规则链 | `RuleBasedTransactionAttribute` + `RollbackRuleAttribute`/`NoRollbackRuleAttribute` |
| 持有者 | `SessionHolder`、`PersistenceManagerHolder`、`ConnectionHolder`（+ rollbackOnly）配合 `ThreadObjectManager` |

## 对应测试

| 测试 | 覆盖 |
|------|------|
| `TransactionTestSuite` | `AbstractPlatformTransactionManager` 传播行为、commit/rollback/rollbackOnly、非事务回退、`TransactionTemplate`（异常/Error 触发回滚）。配合 mock 桩 `TestTransactionManager` |
| `interceptor/TransactionInterceptorTests` | EasyMock mock `PlatformTransactionManager` + `ProxyFactory` 造代理：无事务方法、正常提交、**编程式回滚**（`currentTransactionStatus().setRollbackOnly()`）、checked/unchecked 异常回滚与否、无法创建事务、提交失败 |
| `interceptor/BeanFactoryTransactionTests` | 从 `transactionalBeanFactory.xml` 加载，验证 AOP 代理 + `MapTransactionAttributeSource` 容器集成 |
| `interceptor/RollbackRuleTests` / `RuleBasedTransactionAttributeTests` / `TransactionAttributeEditorTests` | 回滚规则深度匹配、字符串解析 |
| `JtaTransactionTestSuite` | JTA 集成测试 |

## 建议学习顺序

1. **事务核心抽象**：`TransactionDefinition`（背常量）→ `TransactionStatus`（注意是类）→ `PlatformTransactionManager`（三方法策略）
2. **模板方法骨架**：`AbstractPlatformTransactionManager.getTransaction/commit/rollback`——看它如何处理"已有事务 / PROPAGATION_REQUIRED / MANDATORY / 非事务回退"及 5 个 `doXxx`
3. **第一个具体实现**：`JtaTransactionManager`（最薄最易懂），再对比 `HibernateTransactionManager`（带资源绑定、隔离级别支持）
4. **资源同步机制**：先 `ThreadObjectManager`（资源绑定），再 `TransactionSynchronizationManager`（仅回调），最后 `SessionFactoryUtils.closeSessionIfNecessary` 如何把二者粘起来
5. **编程式事务**：`TransactionTemplate.execute` + `TransactionCallback`
6. **声明式事务**：`TransactionInterceptor.invoke` + `TransactionAttributeSource`/`TransactionAttribute`/`RuleBasedTransactionAttribute`（回滚规则）+ `TransactionAttributeEditor`（字符串解析）
7. **测试印证**：读 `TransactionTestSuite` 与 `TransactionInterceptorTests` 对照行为

## 值得精读的源码片段

| 片段 | 看什么 |
|------|--------|
| `AbstractPlatformTransactionManager.java:getTransaction` | 传播判断、MANDATORY 抛异常、`TransactionSynchronizationManager.init()`、`CannotCreateTransactionException` 触发非事务回退 |
| `AbstractPlatformTransactionManager.java:commit` | rollbackOnly 优先于 commit、仅 `isNewTransaction` 才真提交、`triggerAfterCompletion`+`clear` |
| `AbstractPlatformTransactionManager.java:rollback` | 新事务真回滚；参与型调 `doSetRollbackOnly`；无事务支持仅记日志 |
| `TransactionInterceptor.java:invoke` | 声明式事务完整调用链 |
| `TransactionInterceptor.java:onThrowable` | `rollbackOn(ex)` 分支：回滚 vs 仍提交 |
| `RuleBasedTransactionAttribute.java:rollbackOn` | 多回滚规则按继承深度择优 |
| `TransactionAttributeEditor.java:setAsText` | 字符串 `PROPAGATION_*,ISOLATION_*,readOnly,+Ex,-Ex` 解析 |
| `TransactionTemplate.java:execute` | 编程式事务模板方法最简形态 |
| `TransactionSynchronizationManager.java`（全文 86 行） | 仅同步回调无资源绑定——理解 0.9.1 架构的关键 |
| `util/ThreadObjectManager.java`（全文 90 行） | 真正的资源绑定机制（`ThreadLocal<Map>`，不允许重复绑定） |

---

**一句话总结**：0.9.1 的事务抽象以 `PlatformTransactionManager` 策略接口 + `AbstractPlatformTransactionManager` final 模板方法为骨架（子类只实现 `doXxx`），声明式事务靠 `TransactionInterceptor`（AOP Alliance 拦截器）编排"按需开事务→proceed→按异常回滚或提交"。但此版**只有 3 种传播行为**、`TransactionStatus` 是类、**资源绑定在 `util.ThreadObjectManager` 而非 `TransactionSynchronizationManager`**、无 `@Transactional`——这些都是后续版本演进的结果，对照阅读能清晰看到 Spring 事务的演化路径。

← 上一节：[07-jdbc.md](07-jdbc.md) ｜ 下一节：[09-orm.md](09-orm.md)
