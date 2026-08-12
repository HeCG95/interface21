# 09 · orm 模块学习要点

> 基于 `src/com/interface21/orm/**`（24 个 `.java`）与 `test/com/interface21/orm/**`（10 个）逐字核对编写。
> 将 Hibernate 2.0 / JDO 1.0 接入 Spring 的统一资源管理与事务基础设施。

## 模块定位

orm 把"获取/关闭 Session、异常转换、参与线程事务"全部封装，DAO 只面向 `com.interface21.dao` 异常体系。通过 **Template + Interceptor + FactoryBean 三件套**，复用 [transaction](08-transaction.md) 的基础设施，让切换底层事务策略对 DAO 代码**零侵入**。

Hibernate 与 JDO 的集成代码几乎完全对称（同作者 Juergen Hoeller），都遵循同一套模式——读懂 Hibernate 分支，JDO 对照即可。

## 子包结构

```
orm/
├── hibernate/
│   ├── HibernateAccessor.java            Template/Interceptor 共同基类
│   ├── HibernateCallback.java            回调接口
│   ├── HibernateTemplate.java            编程式模板
│   ├── HibernateInterceptor.java         声明式拦截器
│   ├── HibernateTransactionManager.java  Hibernate 事务管理器
│   ├── HibernateTransactionObject.java   事务对象
│   ├── SessionHolder.java                资源持有者
│   ├── SessionFactoryUtils.java          资源管理工具（含 ThreadObjectManager）
│   ├── LocalSessionFactoryBean.java      FactoryBean
│   ├── LocalDataSourceConnectionProvider.java  Hibernate 连接提供者
│   ├── HibernateJdbcException / HibernateSystemException
│   └── support/HibernateDaoSupport.java  DAO 基类
└── jdo/
    ├── JdoCallback / JdoTemplate / JdoInterceptor
    ├── JdoTransactionManager / JdoTransactionObject
    ├── PersistenceManagerHolder
    ├── PersistenceManagerFactoryUtils     含 ThreadObjectManager
    ├── LocalPersistenceManagerFactoryBean
    ├── JdoSystemException / JdoUsageException
    └── support/JdoDaoSupport.java
```

## 三件套模式（Hibernate 为例）

### 1. FactoryBean — `LocalSessionFactoryBean`
`implements FactoryBean, InitializingBean, DisposableBean`。从 `mappingResources`/`hibernateProperties`/`dataSource` 构建 Hibernate `SessionFactory` 单例；`destroy()` 关闭。若注入 `dataSource`，则强制使用 `LocalDataSourceConnectionProvider`（实现 Hibernate 的 `ConnectionProvider`）并通过静态 `ThreadLocal configTimeDataSourceHolder` 传递 DataSource——**让 Hibernate 复用 Spring 管理的 DataSource，避免重复配置**。

JDO 对应 `LocalPersistenceManagerFactoryBean`：从 `configLocation`/`jdoProperties` 经 `JDOHelper.getPersistenceManagerFactory(prop)` 构建 PMF 单例。

### 2. Template — `HibernateTemplate`
`extends HibernateAccessor`。核心 `execute(HibernateCallback)`：每次从 `SessionFactoryUtils` 取线程绑定 Session（有则复用、无则按 `allowCreate` 决定新建或抛异常），执行回调，统一异常转换（`HibernateException` → `DataAccessException`），最后按需关闭。还提供 `find`/`load`/`save`/`update`/`saveOrUpdate`/`delete` 等便捷方法。

### 3. Interceptor — `HibernateInterceptor`
`extends HibernateAccessor implements MethodInterceptor`。声明式等价物：方法前若无线程绑定资源则新建并 `bindThreadObject`，方法后 `removeThreadObject` + 关闭；若已有预绑定（来自事务管理器或外层拦截器）则直接参与、不关闭。

### 资源持有者（SPI）
- `SessionHolder`：包 `Session` + Hibernate `Transaction` + `rollbackOnly`（支持嵌套 Hibernate 事务的 rollback-only）。
- `HibernateTransactionObject`：事务管理器返回的事务对象，内嵌 `SessionHolder` + `isNewSessionHolder` + `previousIsolationLevel`。

## 事务管理器与资源绑定

`HibernateTransactionManager`（`extends AbstractPlatformTransactionManager implements InitializingBean`）：
- 绑 `SessionHolder` 到线程；可选地把 Session 的 JDBC `Connection` 也绑给一个 DataSource（`setDataSource`），从而**允许 Hibernate 与裸 JDBC 在同一事务内混用**。
- 支持自定义隔离级别（通过 `session.connection()`），**不支持超时**。

**协作流程（`doBegin` 内逐行）**：
1. `doGetTransaction`：先查 `SessionFactoryUtils.getThreadObjectManager().hasThreadObject(sessionFactory)`，有则复用 `SessionHolder`；无则新开 Session。
2. `doBegin`：`session.beginTransaction()`，若新 Holder 则 `bindThreadObject(sessionFactory, sessionHolder)`；若设了 dataSource，还会把 `session.connection()` 包成 `ConnectionHolder` 绑给 DataSource。
3. 此后 `HibernateTemplate.execute` / `HibernateInterceptor.invoke` 通过 `SessionFactoryUtils.getSession` 取到**同一个线程绑定 Session**，实现"一个事务一个 Session"。
4. `doCommit/doRollback` 后 `closeSession`：仅当新 Holder 才 `removeThreadObject` + 关闭，预绑定的 Session 不关闭。
5. **JTA 场景**：`SessionFactoryUtils.closeSessionIfNecessary` 检测到 `TransactionSynchronizationManager.isActive()` 时不立即关，而是 `register(new SessionSynchronization(...))` 并绑线程，等 JTA 事务完成的 `afterCompletion` 回调再关——这就是"Hibernate + JTA 也能正确管理 JVM 级缓存"的实现。

> 这是 `util.ThreadObjectManager`（资源绑定）与 `TransactionSynchronizationManager`（仅回调）协作的范例，详见 [08-transaction.md](08-transaction.md)。

## DAO 基类

`HibernateDaoSupport`（`support/`）、`JdoDaoSupport`（`support/`）：注入工厂后自动构造对应 Template，提供 `initDao()` 钩子。

## 关键设计模式

| 模式 | 类 |
|------|----|
| 模板方法 | `HibernateTemplate.execute`、`JdoTemplate.execute` |
| 拦截器/代理（AOP） | `HibernateInterceptor`、`JdoInterceptor`（均 AOP Alliance `MethodInterceptor`） |
| 回调 | `HibernateCallback`、`JdoCallback` |
| FactoryBean | `LocalSessionFactoryBean`、`LocalPersistenceManagerFactoryBean`（把复杂工厂创建纳入 IoC） |
| 持有者 | `SessionHolder`、`PersistenceManagerHolder`、`ConnectionHolder`（+ rollbackOnly）配合 `ThreadObjectManager` |

## 对应测试（`test/com/interface21/orm/`，均用 EasyMock mock 接口，单元测试非集成）

- `hibernate/HibernateTemplateTests`、`HibernateInterceptorTests`、`HibernateTransactionManagerTests`、`LocalSessionFactoryBeanTests`、`support/HibernateDaoSupportTests`
- `jdo/JdoTemplateTests`、`JdoInterceptorTests`、`JdoTransactionManagerTests`、`LocalPersistenceManagerFactoryTests`、`support/JdoDaoSupportTests`

## 建议学习顺序

1. `LocalSessionFactoryBean.afterPropertiesSet`（FactoryBean 如何把 DataSource 注入 Hibernate，经 `LocalDataSourceConnectionProvider.configTimeDataSourceHolder`）
2. `HibernateAccessor`（公共属性）
3. `HibernateTemplate.execute`（编程式模板方法：取 Session → FlushMode → 回调 → flush → closeSessionIfNecessary）
4. `HibernateInterceptor.invoke`（与 Template 对偶的声明式版本，绑/解绑 Session）
5. `HibernateDaoSupport`（DAO 基类）
6. `SessionFactoryUtils.getSession` / `closeSessionIfNecessary`（线程绑定 Session 复用 + JTA 场景注册 `SessionSynchronization` 延迟关闭）
7. `HibernateTransactionManager.doBegin` / `closeSession`（同时绑定 SessionHolder 与 ConnectionHolder）
8. JDO 分支对照阅读

## 值得精读的源码片段

| 片段 | 为什么 |
|------|--------|
| `hibernate/LocalSessionFactoryBean.java:afterPropertiesSet` | FactoryBean 如何把 DataSource 注入 Hibernate |
| `hibernate/HibernateTemplate.java:execute` | 模板方法：取 Session → 回调 → flush → 按需关闭 |
| `hibernate/HibernateInterceptor.java:invoke` | 与 Template 对偶的声明式版本 |
| `hibernate/SessionFactoryUtils.java:getSession` / `closeSessionIfNecessary` | 线程绑定 Session 复用 + JTA 场景延迟关闭 |
| `hibernate/HibernateTransactionManager.java:doBegin` | 同时绑定 SessionHolder 与 ConnectionHolder（Hibernate+JDBC 混用） |
| `hibernate/HibernateTransactionManager.java:closeSession` | 重置隔离级别/只读、仅对新建 Holder 解绑关闭 |

---

**一句话总结**：orm 用 **FactoryBean（建工厂）+ Template（编程式）+ Interceptor（声明式）** 三件套，把 Hibernate/JDO 接入 Spring 的事务基础设施。所有 ORM 事务管理器都 `extends AbstractPlatformTransactionManager` 只实现 `doXxx`；Template/Interceptor 只认线程绑定的资源不关心谁绑的——因此切换事务策略（如 `HibernateTransactionManager` ↔ `JtaTransactionManager`）对 DAO 代码**零侵入**。

← 上一节：[08-transaction.md](08-transaction.md) ｜ 下一节：[10-validation.md](10-validation.md)
