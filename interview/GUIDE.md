# Spring 源码学习指南（interface21 / 0.9.1）

> 本指南基于仓库内**真实源码**编写，覆盖 `src/com/interface21/**` 与 `test/com/interface21/**`。
> 版本：Spring Framework **0.9.1**（2003-08），根包 `com.interface21`（1.0 之后才改名 `org.springframework`）。
> 这是 Spring **最精简、最贴近设计本质**的早期内核——无注解、无泛型、无 SpEL，IoC/AOP/事务/JDBC/MVC 的骨架已完整成型，非常适合作为源码研读的起点。

配套阅读：根目录 [`CLAUDE.md`](../CLAUDE.md)（项目速览、构建命令、代码规范）、[`readme.md`](../readme.md)、[`changelog.txt`](../changelog.txt)。

---

## 0. 为什么要读 0.9.1 这个版本

| 维度 | 说明 |
|------|------|
| 代码规模 | 主源码 404 个 `.java`，测试 141 个，规模可控，可在有限时间读完核心 |
| 纯度 | 没有后期为兼容、性能、注解引入的复杂分支，**一个类往往只表达一个设计思想** |
| 完整性 | IoC 容器、AOP、事务、JDBC、ORM 集成、MVC、校验、远程调用——现代 Spring 的各大子系统在此已全部就位 |
| 历史价值 | 即《Expert One-on-One J2EE Design and Development》一书随书框架，Rod Johnson 原始设计意图最直接 |

读懂这一版，再去看现代 Spring（5.x/6.x）会势如破竹——因为所有"新"特性都是在同一套骨架上做加法。

---

## 1. 分层架构与模块依赖

Spring 是**严格分层**的：所有功能都构筑在更低层之上。你可以只用 Bean 容器而不用 MVC/AOP；但用了 MVC/AOP，它们一定建在 Bean 容器之上。

```
                         ┌─────────────────────────────────────────────┐
   表现层                 │  web (MVC/视图/标签/i18n) │ ui │ validation  │
                         ├─────────────────────────────────────────────┤
   集成层                 │  remoting │ ejb │ jndi                       │
                         ├─────────────────────────────────────────────┤
   数据访问层             │  orm (Hibernate/JDO) │ transaction │ jdbc │ dao │
                         ├─────────────────────────────────────────────┤
   AOP 切面层             │  aop (pointcut / advice / proxy)            │
                         ├─────────────────────────────────────────────┤
   容器/上下文层          │  context (ApplicationContext)               │
                         │  beans (BeanFactory / IoC)    ← 一切的地基   │
                         ├─────────────────────────────────────────────┤
   基础设施层             │  core │ util                                │
                         └─────────────────────────────────────────────┘
```

依赖方向**自上而下**：上层依赖下层，下层绝不反向依赖上层（例如 `beans` 不知道 `web` 的存在）。这一点也体现在 `build.xml` 的三个发布 jar 上：

- **`spring-beans`**：`beans` + `core` + `util/*` —— 仅 IoC 容器
- **`spring-jdbc`**：再加 `aop` + `dao` + `jdbc` + `jndi` + `orm` + `transaction`
- **`spring-full`**：全部 `com/interface21/**`（含 `web`/`ui`/`validation`/`ejb`/`remoting`）

> 改动或研读某类时，先确认它在哪个 jar、属于哪一层，能迅速理清依赖边界。

---

## 2. 模块全景表（15 个模块）

> 「序号」即推荐阅读顺序；「文件数」为主源码/测试的实际 `.java` 数量。
> ⭐ 表示 Spring 的核心中的核心。

| 序号 | 模块 | 主源码 | 测试 | 所属层 | 一句话定位 | 详细文档 |
|----|------|------:|----:|------|-----------|---------|
| 01 | core | 9 | 0 | 基础 | 框架底层抽象（错误码、访问者等） | [01-core.md](01-core.md) |
| 02 | util | 14 | 6 | 基础 | 通用工具类 | [02-util.md](02-util.md) |
| 03 | beans ⭐ | 60 | 27 | 容器 | **IoC 容器**：BeanFactory / BeanWrapper / XML 配置 | [03-beans.md](03-beans.md) |
| 04 | context | 29 | 6 | 容器 | ApplicationContext：国际化、事件、资源 | [04-context.md](04-context.md) |
| 05 | aop | 26 | 12 | AOP | AOP 框架（切点/通知/代理，对接 AOP Alliance） | [05-aop.md](05-aop.md) |
| 06 | dao | 11 | 0 | 数据访问 | 数据访问**异常体系**（DataAccessException） | [06-dao.md](06-dao.md) |
| 07 | jdbc | 55 | 21 | 数据访问 | JDBC 抽象：JdbcTemplate、异常翻译、RDBMS 对象 | [07-jdbc.md](07-jdbc.md) |
| 08 | transaction | 32 | 10 | 数据访问 | 事务抽象：策略/拦截器/JTA/同步管理器 | [08-transaction.md](08-transaction.md) |
| 09 | orm | 24 | 10 | 数据访问 | Hibernate 2.0 / JDO 1.0 集成 | [09-orm.md](09-orm.md) |
| 10 | validation | 7 | 1 | 表现 | 数据校验框架（Validator / Errors） | [10-validation.md](10-validation.md) |
| 11 | ui | 8 | 0 | 表现 | 视图/应用上下文抽象（支撑 web 视图层） | [11-ui.md](11-ui.md) |
| 12 | web | 96 | 37 | 表现 | **MVC Web 框架**：DispatcherServlet 全链路 | [12-web.md](12-web.md) |
| 13 | jndi | 8 | 4 | 集成 | JNDI 抽象与简化 | [13-jndi.md](13-jndi.md) |
| 14 | remoting | 12 | 1 | 集成 | RMI / Hessian / Burlap 远程调用 | [14-remoting.md](14-remoting.md) |
| 15 | ejb | 13 | 6 | 集成 | EJB 访问与支持类 | [15-ejb.md](15-ejb.md) |

---

## 3. 推荐学习路线（五阶段）

### 🟢 阶段一：打地基（01 core → 02 util）
目标：理解 Spring 最底层的基础抽象——异常包装链（`HasRootCause` / `Nested*Exception`，Java 1.4 异常链的过渡方案）、错误码契约（`ErrorCoded`）、排序语义（`Ordered` / `OrderComparator`）、时间戳缓存（`TimeStamped`）；以及 `util` 工具集的设计品味（`StringUtils`、`PathMatcher`、`Constants`、`StopWatch`、`ThreadObjectManager`）。文件少，**1~2 小时**通读。

### 🔵 阶段二：吃透 IoC 容器（03 beans → 04 context）⭐ 最关键
目标：这是整个 Spring 的灵魂。必须做到：
- 能讲清 **Bean 的完整生命周期**（实例化 → 属性注入 → `afterPropertiesSet` → init-method → **`setBeanFactory`(Aware 回调，0.9.1 中在最后！)** → 使用 → 销毁回调），每一步对应哪个类/方法；
- 看懂 `XmlBeanFactory` 如何根据 `spring-beans.dtd` 把 XML 解析成 `BeanDefinition`；
- 理解 `BeanWrapperImpl` 如何用 **JavaBeans 内省 + PropertyEditor** 做属性访问与类型转换；
- 明白 `ApplicationContext` 比 `BeanFactory` 多了什么（事件、国际化、资源加载）。
**预计 1~2 周**，反复读，配合 `docs/model/BeanFactory.svg`、`ApplicationContext.svg`。

### 🟣 阶段三：AOP 切面（05 aop）
目标：理解 0.9.1 版 Spring AOP 的真实形态——它紧贴 **AOP Alliance 的 `MethodInterceptor`/`MethodInvocation`**；**`MethodPointcut` 把"切点条件 + 拦截器"绑在一起**（此版尚无独立的 `Pointcut` 接口、无 `Advisor`、无 Before/After/Throws 分离的通知类型，通知统一是环绕通知）；区分**静态切点（只看 `Method`）vs 动态切点（再看入参）**；搞清 `AopProxy` 按"有无接口"选择 **JDK 动态代理 vs CGLIB**；看懂 `MethodInvocationImpl.proceed()` 如何用责任链递归驱动调用、链尾 `InvokerInterceptor` 反射调目标。AOP 是下一阶段"声明式事务"的基础。

### 🟠 阶段四：数据访问全家桶（06 dao → 07 jdbc → 08 transaction → 09 orm）
目标：
- `dao`：异常体系如何把各家 `SQLException` 翻译成统一的 `DataAccessException`；
- `jdbc`：`JdbcTemplate` 如何用**模板方法 + 回调**消灭 `try/catch/finally` 样板；`RowCallbackHandler` / `ResultReader` / `ResultSetExtracter` 三类回调（**0.9.1 尚无独立 `RowMapper` 接口**，行映射由 `MappingSqlQuery.mapRow` 承担）；`query` 方法**返回 `void`**（结果靠回调累积）；`SqlQuery`/`SqlUpdate`/`StoredProcedure` 等 RDBMS 对象；
- `transaction`：`PlatformTransactionManager` 抽象、**事务传播行为/隔离级别常量**、`TransactionInterceptor` 如何借 AOP 实现**声明式事务**、`TransactionSynchronizationManager` 如何把资源绑定到线程；
- `orm`：`HibernateTemplate`/`JdoTemplate` + `Interceptor` + `FactoryBean` 三件套如何复用上面的事务基础设施。
这一阶段是面试重灾区，务必精读。

### 🔴 阶段五：表现层与集成层（10 validation → 11 ui → 12 web → 13~15 集成）
目标：
- `web`：画出 **DispatcherServlet 一次请求的完整处理流程**（前端控制器 → HandlerMapping → Controller → ModelAndView → ViewResolver → View 渲染）；理清表单控制器继承体系、数据绑定、主题/国际化解析；
- `validation`：`Validator` 如何与 web 表单绑定协作；
- `jndi`/`remoting`/`ejb`：了解 Spring 如何用一致的抽象屏蔽 J2EE 各类底层服务，这部分可**选读**。

> 💡 节奏建议：主线是 **beans → context → aop → jdbc → transaction → web**。其余模块按需穿插。如果时间紧，优先保证主线六块。

---

## 4. 如何读这份源码（方法论）

1. **接口先行**：每个模块先读它的核心接口（如 `BeanFactory`、`JdbcTemplate` 的回调接口、`PlatformTransactionManager`），把握"契约"，再看实现。
2. **测试驱动阅读**：`test/` 下的 `*TestSuite` / `*Tests` 是最好的活文档——先看测试怎么用，再去看实现怎么写。测试用的 `TestBean` 等桩对象尤其直观。
3. **画类图对照**：`docs/model/*.svg` 已有维护者绘制的 UML（`BeanFactory.svg`、`JdbcTemplate.svg`、`ViewResolver.svg` 等），读源码时对照看。
4. **跑起来验证**：`ant build` 编译，`ant tests` 跑测试（见 [CLAUDE.md](../CLAUDE.md)）。改一处断点调试胜过读十遍。
5. **看示例**：`samples/petclinic`、`samples/countries` 是真实应用，能让你看到各模块如何组合。
6. **读 DTD/配置**：`src/com/interface21/beans/factory/xml/spring-beans.dtd` 和 `samples/*/war/WEB-INF/*.xml` 揭示了配置视角的 Spring。
7. **关注 `@author` 与 `@since`**：Javadoc 标注了每个类的起源与作者，能体会设计演进。

### 阅读前的知识储备
| 主题 | 为什么需要 |
|------|-----------|
| Java 反射 (`java.lang.reflect`) | beans 的属性注入、AOP 的方法拦截都基于反射 |
| JDK 动态代理 (`Proxy`/`InvocationHandler`) | AOP 默认代理机制 |
| JavaBeans 内省 (`Introspector`/`PropertyDescriptor`)、`PropertyEditor` | `BeanWrapperImpl` 的核心 |
| Servlet 2.3 生命周期 | web/MVC 的基础 |
| 事务基本概念（传播、隔离） | transaction 模块 |
| 设计模式：模板方法、策略、工厂、代理、责任链、观察者 | 几乎每个模块都在用 |

---

## 5. 面试高频考点 × 模块映射

| 面试题 | 答案所在模块 / 文档 |
|--------|-------------------|
| Bean 生命周期？BeanFactory 与 ApplicationContext 区别？ | beans / context → [03](03-beans.md) [04](04-context.md) |
| 循环依赖怎么解决？*(注：0.9.1 尚无现代三级缓存，需据代码实事求是)* | beans → [03](03-beans.md) |
| Spring AOP vs AspectJ？JDK 代理 vs CGLIB？ | aop → [05](05-aop.md) |
| JdbcTemplate 为什么不用手写 finally？模板方法怎么用？ | jdbc → [07](07-jdbc.md) |
| SQLException 如何转成 DataAccessException？ | dao/jdbc → [06](06-dao.md) [07](07-jdbc.md) |
| 声明式事务原理？事务传播行为有哪些？ | transaction → [08](08-transaction.md) |
| @Transactional 失效场景？*(本版用 TransactionInterceptor + AOP，原理一致)* | transaction/aop → [08](08-transaction.md) |
| DispatcherServlet 请求处理流程？ | web → [12](12-web.md) |
| Spring MVC 的 Controller 继承体系？数据绑定？ | web → [12](12-web.md) |
| Spring 如何集成 Hibernate/JDO 并统一事务？ | orm/transaction → [09](09-orm.md) [08](08-transaction.md) |

---

## 6. 文档约定

- 每个模块文档（`01-core.md` ~ `15-ejb.md`）包含：**模块定位 → 子包结构 → 核心接口（含逐字方法签名）→ 核心实现类 → 设计模式 → 对应测试 → 学习顺序 → 精读片段**。
- 所有类名、方法签名均**逐字抄录自源码**，并标注文件路径（如 `src/com/interface21/beans/factory/BeanFactory.java`），便于点击/跳转核对。
- 中文讲解为主，关键术语保留英文。

> 本仓库为**学习/研读型 fork**，源码中夹杂维护者添加的中文 Javadoc 翻译（双语对照风格）。读源码遇到中文注释即为此类学习产物。

祝阅读愉快。建议从 [01-core.md](01-core.md) 开始，但若想直奔主题，可从 [03-beans.md](03-beans.md) 起步。
