# 15 · ejb 模块学习要点

> 基于 `src/com/interface21/ejb/**`（13 个 `.java`）与 `test/com/interface21/ejb/**`（6 个）逐字核对编写。
> 分两条线：`support` 为"用 Spring 写 EJB"提供基类；`access` 为"在 Spring 中调用 EJB"提供 AOP 代理。

> ⚠️ 纠偏：网上或任务描述里的 `BaseStatelessEjbObject`、`SessionBeanFactory` 在 0.9.1 中**不存在**，真实基类如下。

## 模块定位

- **`support`**（用 Spring 写 EJB）：让 EJB 成为**薄壳**，业务逻辑落在 Spring `BeanFactory` 管理的 POJO。EJB 方法体只需从 `getBeanFactory()` 取 POJO 委托。
- **`access`**（在 Spring 中调用 EJB）：提供 AOP 代理，使调用者只看到业务接口，EJB Home/创建/异常处理全部被拦截器吞掉——与 [remoting](14-remoting.md) 的透明化哲学一脉相承。

## ejb/support —— EJB 基类

| 类 | 路径可见性 | 要点 |
|----|-----------|------|
| `AbstractEnterpriseBean` | 包级 abstract，`implements EnterpriseBean` | 所有 EJB 基类根：`loadBeanFactory()`（默认 `XmlBeanFactoryLoader`）、`setBeanFactoryLoader`、`getBeanFactory`、`ejbRemove` |
| `AbstractSessionBean` | 包级，`extends AbstractEnterpriseBean implements SessionBean` | 保存 `SessionContext` |
| `AbstractStatelessSessionBean` | public，`extends AbstractSessionBean` | SLSB 最常用基类：`ejbCreate()`（已实现：`loadBeanFactory()` + 抽象 `onEjbCreate()`）；`ejbActivate`/`ejbPassivate` 抛 `IllegalStateException`（SLSB 不应被调用，EJB 规范禁 final 只能抛异常兜底） |
| `AbstractStatefulSessionBean` | public，`extends AbstractSessionBean` | 把 `loadBeanFactory` 暴露为 protected，交子类在自己的 `ejbCreate` 调 |
| `AbstractMessageDrivenBean` | public，`extends AbstractEnterpriseBean implements MessageDrivenBean` | MDB 基类：`setMessageDrivenContext`/`ejbCreate`/抽象 `onEjbCreate` |
| `AbstractJmsMessageDrivenBean` | public，`extends AbstractMessageDrivenBean implements MessageListener` | 空类，仅用 `implements MessageListener` 强制子类实现 JMS 回调（EJB 2.1 前 MDB 即 JMS-MDB） |
| `XmlBeanFactoryLoader` | public，`implements BeanFactoryLoader` | **EJB 与 Spring IoC 桥接的关键** |

### `XmlBeanFactoryLoader` — `ejb/support/XmlBeanFactoryLoader.java`
```java
public static final String BEAN_FACTORY_PATH_ENVIRONMENT_KEY = "java:comp/env/ejb/BeanFactoryPath";
public BeanFactory loadBeanFactory() throws BootstrapException;
```
`loadBeanFactory()`：用 `JndiTemplate.lookup` 取 classpath 路径 → `getClass().getResourceAsStream` → `new XmlBeanFactory(is)`。JNDI 缺失、资源不存在、XML 非法均转成带可读消息的 `BootstrapException`。**部署时改 JNDI env-entry 即可切换容器配置，无需重新打包 EJB**。

## ejb/access —— AOP 方式访问 EJB

| 类 | 要点 |
|----|------|
| `AbstractSlsbInvokerInterceptor` | **同时** `extends AbstractJndiLocator` 且 `implements MethodInterceptor, InitializingBean`——把"JNDI 抽象"与"AOP 拦截器"两条线汇合。常量 `CREATE_METHOD = "create"`；`located(o)` 缓存 home 的 BeanWrapper |
| `AbstractRemoteSlsbInvokerInterceptor` | `extends AbstractSlsbInvokerInterceptor`；`newSessionBeanInstance()` 调 `homeBeanWrapper.invoke("create", null)` |
| `SimpleRemoteSlsbInvokerInterceptor` | 最末端拦截器（见下） |
| `LocalSlsbInvokerInterceptor` | 本地 EJB 版，`newSessionBeanInstance()` 返回 `EJBLocalObject`，结构对称 |
| `SimpleRemoteStatelessSessionProxyFactoryBean` / `LocalStatelessSessionProxyFactoryBean` | 便捷工厂：`setBusinessInterface` + `afterLocated()` 建 AOP 代理 |

### `SimpleRemoteSlsbInvokerInterceptor.invoke`
```java
public Object invoke(MethodInvocation invocation) throws Throwable {
    EJBObject ejb = newSessionBeanInstance();
    try { return invocation.getMethod().invoke(ejb, invocation.getArguments()); }
    catch (InvocationTargetException ex) { throw ex.getTargetException(); }
    catch (Throwable t) { throw new AspectException("Failed to invoke remote EJB", t); }
}
```

### `SimpleRemoteStatelessSessionProxyFactoryBean.afterLocated`
```java
public void afterLocated() {     // 在 JNDI lookup 完成后建 AOP 代理
    ProxyFactory pf = new ProxyFactory(new Class[] { this.businessInterface });
    pf.addInterceptor(this);      // 复用 this（自身就是末端拦截器）
    this.proxy = pf.getProxy();
}
```
注意：它要求设的是 `businessInterface`（业务方法父接口），代理只暴露这个接口——**客户端不含 `RemoteException`、不含 `EJBObject` 依赖**，把"EJB 调用"伪装成"本地 bean 引用"。

## 设计亮点

1. **EJB 即薄壳，业务在 BeanFactory**（support）：`ejbCreate` 加载 Spring 容器，EJB 方法体从 `getBeanFactory()` 取 POJO 委托。`XmlBeanFactoryLoader` 通过 JNDI env-entry 找 XML——部署时改配置即可，无需重打包。

2. **多重继承的组合威力**（access）：`AbstractSlsbInvokerInterceptor extends AbstractJndiLocator implements MethodInterceptor`——一个类同时是 JNDI Service Locator 和 AOP `MethodInterceptor`。JNDI 层负责"找到 home"，AOP 层负责"每次 create + 反射执行"。这是 Spring 把不同模块抽象拼装的经典案例。

3. **业务接口而非组件接口**：代理只暴露 `businessInterface`，客户端零 EJB 依赖。

4. **错误处理策略**：`invoke` 对 `InvocationTargetException` 解包抛原异常（应用异常原样传播），其他 `Throwable` 包成 `AspectException`。

5. **生命周期防御**：`AbstractStatelessSessionBean.ejbActivate/Passivate` 抛 `IllegalStateException`——EJB 规范禁 final，只能用运行时异常阻止误调用。

## 关键设计模式

| 模式 | 类 |
|------|----|
| 模板方法 | `AbstractEnterpriseBean.loadBeanFactory` + `AbstractStatelessSessionBean.ejbCreate` + `onEjbCreate` 钩子 |
| 拦截器/代理（AOP） | `*SlsbInvokerInterceptor`（AOP Alliance `MethodInterceptor`） |
| 工厂 + 多重继承组合 | `SimpleRemoteStatelessSessionProxyFactoryBean`（extends 拦截器 implements FactoryBean） |
| Service Locator | 复用 jndi 模块的 `AbstractJndiLocator` |

## 对应测试

| 测试 | 覆盖 |
|------|------|
| `access/SimpleRemoteSlsbInvokerInterceptorTests` | EasyMock mock `Context`/`SlsbHome`/`RemoteInterface`：lookup 成功/失败、方法正常调用、应用异常与 `RemoteException` 透传（`createInitialContext` 子类化注入 mock） |
| `access/LocalSlsbInvokerInterceptorTests` | 本地 EJB 版 |
| `access/SimpleRemoteStatelessSessionProxyFactoryBeanTests` | 便捷工厂代理 |
| `access/LocalStatelessSessionProxyFactoryBeanTests` | 本地便捷工厂 |
| `support/EjbSupportTests` | EJB 基类 |
| `support/XmlBeanFactoryLoaderTests` | 用 `SimpleNamingContextBuilder` 搭内存 JNDI，测三种失败（JNDI 无 key、path 找不到资源、path 非法 XML），断言 `BootstrapException` 消息含可读提示 |

## 建议学习顺序与精读片段

1. `support/AbstractEnterpriseBean.java:loadBeanFactory`（:53）——EJB 与 IoC 桥接点
2. `support/AbstractStatelessSessionBean.java:ejbCreate`（:44）+ `onEjbCreate` 钩子
3. `support/XmlBeanFactoryLoader.java:loadBeanFactory`（:46）——JNDI → classpath → XmlBeanFactory
4. `access/AbstractSlsbInvokerInterceptor.java`（:20）——看 `extends AbstractJndiLocator implements MethodInterceptor` 的多重身份
5. `access/SimpleRemoteSlsbInvokerInterceptor.java:invoke`（:46）——create + 反射 + 异常解包
6. `access/SimpleRemoteStatelessSessionProxyFactoryBean.java:afterLocated`（:57）——AOP 代理如何把拦截器变成业务接口 bean
7. `test/ejb/support/XmlBeanFactoryLoaderTests`——看 support 测试如何用 jndi.support 搭沙箱

| 精读片段 | 为什么 |
|----------|--------|
| `support/XmlBeanFactoryLoader.java:loadBeanFactory`（:46） | EJB↔IoC 桥接，JNDI env-entry 驱动容器加载 |
| `access/AbstractSlsbInvokerInterceptor.java`（:20） | `extends AbstractJndiLocator implements MethodInterceptor` 的多重身份 |
| `access/SimpleRemoteSlsbInvokerInterceptor.java:invoke`（:46） | create + 反射 + 异常解包策略 |
| `access/SimpleRemoteStatelessSessionProxyFactoryBean.java:afterLocated`（:57） | AOP 代理把拦截器变成业务接口 bean |

---

← 上一节：[14-remoting.md](14-remoting.md) ｜ 返回：[GUIDE.md](GUIDE.md)
