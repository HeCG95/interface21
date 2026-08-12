# 13 · jndi 模块学习要点

> 基于 `src/com/interface21/jndi/**`（8 个 `.java`）与 `test/com/interface21/jndi/**`（4 个）逐字核对编写。
> 对 JNDI 操作做"模板 + 回调"封装（类比 `JdbcTemplate`），并提供 Service Locator 基类与测试用纯内存 JNDI 环境。

## 模块定位

jndi 模块统一处理 `InitialContext` 的创建与关闭、`NamingException`；提供 Service Locator 基类、`FactoryBean` 桥接，以及测试用的纯内存 JNDI 环境。它是 [ejb](15-ejb.md) access 的基础（`AbstractSlsbInvokerInterceptor extends AbstractJndiLocator`）。

## 核心接口与实现类

### `JndiTemplate` — `jndi/JndiTemplate.java`（包核心类）
Javadoc 明确："analogous to the JdbcTemplate class"。
```java
public JndiTemplate();
public JndiTemplate(Properties environment);
public void setEnvironment(Properties environment);
public Object lookup(final String name) throws NamingException;
public void bind(final String name, final Object object) throws NamingException;
public void unbind(final String name) throws NamingException;
public Object execute(ContextCallback contextCallback) throws NamingException;
protected Context createInitialContext() throws NamingException;   // 可子类化（测试用）
```
`execute()` 是**模板方法**：`createInitialContext()` → 回调 → `finally` 中 `ctx.close()`。`lookup` 对返回 null 的不良实现也抛 `NamingException`。

### `ContextCallback` — `jndi/ContextCallback.java`（回调接口）
```java
Object doInContext(Context ctx) throws NamingException;
```

### `AbstractJndiLocator` — `jndi/AbstractJndiLocator.java`（Service Locator 抽象基类，`implements InitializingBean`）
```java
public static String CONTAINER_PREFIX = "java:comp/env/";
public final void setJndiTemplate(JndiTemplate template);
public final void setJndiName(String jndiName);
public final void setInContainer(boolean inContainer);
public final void afterPropertiesSet() throws NamingException, IllegalArgumentException;
protected abstract void located(Object o);
```
`afterPropertiesSet()`：校验 `jndiName`、按 `inContainer` 决定是否补 `java:comp/env/` 前缀、`lookup`、回调 `located(o)` 交子类缓存。

### `JndiObjectFactoryBean` — `jndi/JndiObjectFactoryBean.java`
`extends AbstractJndiLocator implements FactoryBean`。把 JNDI 对象暴露为 Bean 引用，使"JNDI 数据源"与"本地 `DriverManagerDataSource`"仅靠配置切换：
```java
protected void located(Object o) { this.jndiObject = o; }
public Object getObject() { return this.jndiObject; }
public boolean isSingleton() { return false; }
```

### `JndiTemplateEditor` — `jndi/JndiTemplateEditor.java`
`extends PropertyEditorSupport`，把 properties 格式字符串解析成带环境的 `JndiTemplate`。

### support 子包
| 类 | 要点 |
|----|------|
| `support/ExpectedLookupTemplate` | `extends JndiTemplate`，mock 对象：构造传期望 name+返回对象，`lookup` 仅对匹配 name 返回，否则抛 `UnsupportedOperationException` |
| `support/SimpleNamingContext` | `implements Context` 的纯内存实现，仅支持 String 名绑定 Object；含内部类 `NameClassPairEnumeration`/`BindingEnumeration`；`Name` 参数重载抛 `UnsupportedOperationException` |
| `support/SimpleNamingContextBuilder` | `implements InitialContextFactoryBuilder`，**JVM 级 JNDI 环境构建器**：`activate()` 调 `NamingManager.setInitialContextFactoryBuilder(this)`，之后 `new InitialContext()` 全走此工厂——让 EJB/JNDI 代码脱离容器做单测的关键 |

## 设计亮点

1. **模板方法 + 回调**：`JndiTemplate.execute(ContextCallback)` 统一资源回收，与 `JdbcTemplate` 同构——Spring 一以贯之的"模板范式"。
2. **`createInitialContext()` 设为 protected**：无需 mock 框架即可单测，子类化返回 mock Context。
3. **Service Locator 与 FactoryBean 融合**：`AbstractJndiLocator` 解决"在哪找"，`JndiObjectFactoryBean` 解决"如何注入"。配合 `inContainer` 开关，同一套代码容器内/外皆可运行。
4. **JVM 级 JNDI 沙箱**：`SimpleNamingContextBuilder` 通过 JNDI SPI 的 `InitialContextFactoryBuilder` 接管整个 JVM 的 `InitialContext`——这是 `XmlBeanFactoryLoaderTests` 等测试脱离容器运行的基础。

## 关键设计模式

| 模式 | 类 |
|------|----|
| 模板方法 + 回调 | `JndiTemplate.execute(ContextCallback)` |
| Service Locator | `AbstractJndiLocator` |
| FactoryBean | `JndiObjectFactoryBean` |
| 属性编辑器 | `JndiTemplateEditor` |
| 测试替身 | `ExpectedLookupTemplate`、`SimpleNamingContext`、`SimpleNamingContextBuilder` |

## 对应测试

| 测试 | 覆盖 |
|------|------|
| `JndiTemplateTests` | EasyMock mock `Context`，测 `bind`/`unbind`（子类化 `createInitialContext`） |
| `JndiObjectFactoryBeanTests` | FactoryBean 行为 |
| `JndiTemplateEditorTests` | properties 字符串解析 |
| `SimpleNamingContextTests` | 内存 Context 实现 |

## 建议学习顺序与精读片段

1. `ContextCallback.java`（1 方法，回调契约）
2. `JndiTemplate.java:execute`（模板方法）+ `createInitialContext`（为何 protected）
3. `AbstractJndiLocator.java:afterPropertiesSet`（前缀补全逻辑）
4. `JndiObjectFactoryBean.java`（FactoryBean 接入 IoC）
5. `SimpleNamingContextBuilder.java:activate`（与 JNDI SPI 的关系）
6. `JndiTemplateTests`（测试如何子类化绕过真实 JNDI）

| 精读片段 | 为什么 |
|----------|--------|
| `JndiTemplate.java:execute`（:124） | 模板方法 + 回调，与 JdbcTemplate 同构 |
| `AbstractJndiLocator.java:afterPropertiesSet`（:109） | `inContainer` 前缀补全逻辑 |
| `SimpleNamingContextBuilder.java:activate`（:117） | JVM 级 JNDI 沙箱如何接管 `InitialContext` |

---

← 上一节：[12-web.md](12-web.md) ｜ 下一节：[14-remoting.md](14-remoting.md)
