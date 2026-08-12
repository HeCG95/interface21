# 04 · context 模块学习要点

> 基于 `src/com/interface21/context/**`（29 个 `.java`）与 `test/com/interface21/context/**`（6 个）逐字核对编写。

## ⚠️ 阅读前必读：0.9.1 与现代 Spring 的关键差异

| 现代 Spring 的概念 | 0.9.1 实际情况 |
|--------------------|----------------|
| `ConfigurableApplicationContext` 接口 | **不存在**。只有 `ApplicationContext` 一个顶层接口，`refresh()`/`close()` 直接声明在它上面 |
| `ApplicationContextEvent` 基类 | **不存在**。`ContextRefreshedEvent`/`ContextClosedEvent` 直接继承 `ApplicationEvent` |
| 统一 `Resource` 抽象（`core.io.Resource`） | **不存在**。资源 API 只有 `getResourceAsStream(String)` + `getResourceBasePath()` |
| `Environment` / `${placeholder}` | **不存在**。只有 `PropertyResourceConfigurer`，做的是**属性覆盖**（`beanName.property=value`），不是占位符替换 |

## 模块定位：ApplicationContext 相比 BeanFactory 增强了什么

`ApplicationContext` 在 BeanFactory 的"对象创建/装配"之上叠加**五项能力**：

1. **国际化（i18n）**：继承 `MessageSource`，提供 `getMessage(...)` 按 Locale 解析消息。
2. **事件机制（Observer 模式）**：`publishEvent(ApplicationEvent)` + 监听器自动注册。
3. **资源访问一致 API**：`getResourceAsStream(String)` 统一处理 URL / 文件路径 / 相对路径。
4. **共享对象**：`shareObject` / `sharedObject` / `removeSharedObject`，类似 `ServletContext` 属性。
5. **分层上下文（hierarchy）**：`getParent()` 形成父子上下文树，子定义优先。

### 关键继承关系（面试高频）
```
ApplicationContext extends MessageSource, ListableBeanFactory, HierarchicalBeanFactory
                     ↑                ↑                       ↑
                 i18n 能力      可枚举 bean 能力        分层能力
                                     ↑
                       ListableBeanFactory extends BeanFactory
                       HierarchicalBeanFactory extends BeanFactory
```
即 `ApplicationContext` 通过**接口多继承**把"消息源 + 可枚举工厂 + 可分层工厂"聚合为一个类型。注意实现上不是 class 继承 BeanFactory，而是**内部组合一个 `ListableBeanFactoryImpl`** 并委托——这是"接口聚合 + 实现组合"。

## 子包结构

```
context/                  顶层接口与异常（ApplicationContext, MessageSource, ApplicationEvent...）
context/support/          实现（AbstractApplicationContext, *XmlApplicationContext, 事件/消息源实现...）
```

## 核心接口（路径 + 逐字签名）

### `ApplicationContext` — `src/com/interface21/context/ApplicationContext.java:51`
```java
public interface ApplicationContext extends MessageSource, ListableBeanFactory, HierarchicalBeanFactory {
    ApplicationContext getParent();
    String getDisplayName();
    long getStartupDate();
    ContextOptions getOptions();
    void refresh() throws ApplicationContextException;
    void close() throws ApplicationContextException;
    void publishEvent(ApplicationEvent event);
    InputStream getResourceAsStream(String location) throws IOException;
    String getResourceBasePath();
    void shareObject(String key, Object o);
    Object sharedObject(String key);
    Object removeSharedObject(String key);
}
```

### `MessageSource` — `src/com/interface21/context/MessageSource.java:22`
```java
public interface MessageSource {
    String getMessage(String code, Object args[], String defaultMessage, Locale locale);
    String getMessage(String code, Object args[], Locale locale) throws NoSuchMessageException;
    String getMessage(MessageSourceResolvable resolvable, Locale locale) throws NoSuchMessageException;
}
```
参数化基于 `java.text.MessageFormat`（`{0}`、`{1,date}`）。三参重载带默认值不抛异常；两参找不到抛 `NoSuchMessageException`；`MessageSourceResolvable` 重载支持多 code 兜底。

### `ApplicationEvent` / `ApplicationListener` / `ApplicationEventMulticaster`
```java
// context/ApplicationEvent.java:23
public abstract class ApplicationEvent extends EventObject {
    private long timestamp;
    public ApplicationEvent(Object source);
    public long getTimestamp();
}

// context/ApplicationListener.java:24
public interface ApplicationListener extends EventListener {
    void onApplicationEvent(ApplicationEvent e);
}

// context/ApplicationEventMulticaster.java:11  —— 注意它本身 extends ApplicationListener
public interface ApplicationEventMulticaster extends ApplicationListener {
    void addApplicationListener(ApplicationListener listener);
    void removeApplicationListener(ApplicationListener listener);
    void removeAllListeners();
}
```
> 巧妙之处：`ApplicationEventMulticaster` **既是监听器又是广播器**——上下文把事件交给它（以 `onApplicationEvent` 形式），它再 fan-out 给所有注册监听器。

### 其他接口
```java
// context/NestingMessageSource.java:12  —— 可分层消息源
public interface NestingMessageSource extends MessageSource { void setParent(MessageSource parent); }

// context/MessageSourceResolvable.java:12  —— 消息值对象（多 code + 参数 + 默认消息）
public interface MessageSourceResolvable {
    public String[] getCodes();
    public Object[] getArgs();
    public String getDefaultMessage();
}

// context/ApplicationContextAware.java:28
public interface ApplicationContextAware {
    void setApplicationContext(ApplicationContext context) throws ApplicationContextException;
}

// context/support/BeanFactoryPostProcessor.java:22  —— 注意继承 ApplicationContextAware
public interface BeanFactoryPostProcessor extends ApplicationContextAware {
    void postProcessBeanFactory(ListableBeanFactoryImpl beanFactory) throws ApplicationContextException;
}
```
异常：`NoSuchMessageException extends Exception`（受检）；`ApplicationContextException extends NestedRuntimeException`。

## 核心实现类（support 子包）

### `AbstractApplicationContext`（模板方法核心）— `support/AbstractApplicationContext.java:81`
实现 ApplicationContext 通用逻辑，强制子类实现两个模板方法：
```java
protected abstract void refreshBeanFactory() throws ApplicationContextException;   // :577
protected abstract ListableBeanFactoryImpl getBeanFactory();                       // :585
```
设计要点：
- **持有内部 `ListableBeanFactoryImpl`**，`getBean` 委托给它再做 `ApplicationContextAware` 回调（组合 + 委托）：
  ```java
  public Object getBean(String name) throws BeansException {       // :508
      Object bean = getBeanFactory().getBean(name);
      configureManagedObject(name, bean);
      return bean;
  }
  ```
- **`refresh()` 是 final 模板方法**（:221-268），规定启动 8 步顺序（见下文"启动流程"）。
- **事件广播器直接 new**（:125）：`private ApplicationEventMulticaster eventMulticaster = new ApplicationEventMulticasterImpl();`（0.9.1 尚不支持配成 bean，有 TODO）。
- **`publishEvent` 是 final**（:403-411）：先本地 multicaster，**再向上冒泡到 parent**：
  ```java
  public final void publishEvent(ApplicationEvent event) {
      this.eventMulticaster.onApplicationEvent(event);
      if (this.parent != null) { parent.publishEvent(event); }
  }
  ```
- **`close()` 不是 final**（:384）：`destroySingletons()` 后发 `ContextClosedEvent`，**不会**调 parent 的 close。
- 两个特殊 bean 名常量：`OPTIONS_BEAN_NAME = "contextOptions"`、`MESSAGE_SOURCE_BEAN_NAME = "messageSource"`。

### XML 上下文实现链
| 类 | 要点 |
|----|------|
| `AbstractXmlApplicationContext`（:25） | `refreshBeanFactory()` 创建 `XmlBeanFactory(getParent())`、注入 `ResourceBaseEntityResolver`、`loadBeanDefinitions(is)`；子类实现 `getInputStreamForBeanFactory()` |
| `FileSystemXmlApplicationContext`（:20） | 构造器收 `String[] locations`，**最后一个 location 是本上下文，前面全是 parent**（:47-61），递归建父子链；构造即 `refresh()` |
| `ClassPathXmlApplicationContext`（:30） | 继承 `FileSystemXmlApplicationContext` 复用父子链逻辑，只重写 `getResourceByPath`（走 classpath，自动补 `/`）、`getResourceBasePath`（返回 null）、`createParentContext` |
| `StaticApplicationContext`（:22） | 纯编程式注册 bean（测试用）；构造时自动注册 `StaticMessageSource` 为 `messageSource` bean |

### 消息源体系
```
MessageSource → NestingMessageSource → AbstractNestingMessageSource（abstract）
                                        ├── StaticMessageSource（内存 Map）
                                        └── ResourceBundleMessageSource（JDK ResourceBundle）
```
- `AbstractNestingMessageSource`（:26）：`getMessage(code,args,locale)` 是 **final 模板方法**（:126）——先调子类 `resolve()`，null 则委派 parent，再 null 抛 `NoSuchMessageException`，最后 `MessageFormat` 格式化；**缓存 `MessageFormat` 实例**（按 `localeKey.code` 索引）。
- `ResourceBundleMessageSource.resolve()`（:41-58）：**找不到 bundle/key 都返回 null 不抛异常**，以便 parent 接手。

### 其他
| 类 | 要点 |
|----|------|
| `ApplicationEventMulticasterImpl`（:29） | `HashSet` 持监听器（不允许重复注册）；**同步**调用、调用方线程；Javadoc 坦言线程安全非其职责、单监听器阻塞会卡全应用 |
| `MessageSourceResourceBundle`（:15） | **反向适配**：把 `MessageSource` 包装成 JDK `ResourceBundle` |
| `ApplicationObjectSupport`（:39） | `ApplicationContextAware` 便利基类：`setApplicationContext` 设为 final、存 context、调 `initApplicationContext()` 钩子；忽略重入 |
| `PropertyResourceConfigurer`（:27） | 读 `.properties` 的 `beanName.property=value`，调 `registerAdditionalPropertyValue` **覆盖** bean 定义属性（属性覆盖语义，非占位符） |
| `ResourceBaseEntityResolver`（:34） | SAX `EntityResolver`，让 XML 实体引用相对上下文资源基路径解析 |

## 启动流程：refresh() 的 8 步（必背）

`AbstractApplicationContext.refresh()`（:221-268）严格顺序：
1. 检查 reloadable → 设 startupTime → `refreshBeanFactory()`（子类实现）
2. `invokeContextConfigurer()`——查找所有 `BeanFactoryPostProcessor` bean 并调 `postProcessBeanFactory`
3. `loadOptions()`——取名为 `contextOptions` 的 bean，否则默认 `ContextOptions`
4. `initMessageSource()`——取名为 `messageSource` 的 bean；若是 `NestingMessageSource` 且定义在本上下文则 `setParent(parent)`
5. `onRefresh()`——钩子，默认空实现
6. `refreshListeners()`——用 `BeanFactoryUtils.beansOfType(ApplicationListener.class, this)` 自动发现监听器 bean 注册
7. `preInstantiateSingletons()`——预实例化所有单例（触发 `ApplicationContextAware` 回调）
8. `publishEvent(new ContextRefreshedEvent(this))`

## 事件机制完整链路

```
publishEvent(event)
   ├─→ eventMulticaster.onApplicationEvent(event)        // 本地广播
   │       └─→ ApplicationEventMulticasterImpl 遍历 HashSet
   │              └─→ 每个 listener.onApplicationEvent(e)  // 同步、调用方线程
   └─→ if (parent != null) parent.publishEvent(event)    // 向上冒泡
```
- 注册有两条路径：编程式 `addListener`（protected）/ 声明式（bean 实现 `ApplicationListener` 接口被自动发现，测试 `testBeanAutomaticallyHearsEvents` 验证）。
- **0.9.1 监听器不能按事件类型过滤**——所有事件都调到所有监听器，由监听器自行 `instanceof` 判断。
- 事件**冒泡到父上下文**；`close()` **不冒泡**。

## 关键设计模式

| 模式 | 对应类 |
|------|--------|
| 观察者（Observer） | `ApplicationEvent`/`ApplicationListener`/`ApplicationEventMulticaster`/`publishEvent` |
| 模板方法 | `refresh()`（final）+ `refreshBeanFactory()`；`AbstractNestingMessageSource.getMessage()`（final）+ `resolve()` |
| 组合 + 委托 | `AbstractApplicationContext` 持有 `ListableBeanFactoryImpl`，`getBean` 委托 |
| 接口聚合（Mixin） | `ApplicationContext extends MessageSource, ListableBeanFactory, HierarchicalBeanFactory` |
| 分层/职责链 | 父子上下文；`NestingMessageSource.setParent`；`publishEvent` 冒泡 |
| 适配器 | `MessageSourceResourceBundle`（MessageSource→ResourceBundle） |
| 策略 | `MessageSource` 可换实现，按 bean 名 `messageSource` 注入 |

## 对应测试（`test/com/interface21/context/`，5 个文件，无 XML 资源——用 `StaticApplicationContext` 编程式构建）

- **`AbstractApplicationContextTests`**（抽象基类，继承 `AbstractListableBeanFactoryTests`）：`testContextAwareSingletonWasCalledBack`/`testContextAwarePrototypeWasCalledBack`、`testParentNonNull`/`testGrandparentNull`、`testCloseTriggersDestroy`、`testMessageSource`、`testRetrievesSharedObject`、**`testEvents`**（验证子+父监听器都收到事件，即冒泡）、`testBeanAutomaticallyHearsEvents`
- **`ACATest`**：`ApplicationContextAware` 测试 bean，验证重入保护
- **`BeanThatListens`** / **`TestListener`**：计数监听器
- **`support/StaticApplicationContextTestSuite`**：编程式构造父子两层上下文
- **`support/StaticMessageSourceTestSuite`**：`MessageFormat` 缓存验证、`MessageSourceResolvable` 多 code 兜底、Locale.US/UK 国际化

## 建议学习顺序

1. `ApplicationContext.java`（接口 Javadoc + 方法签名）——建立"Context = BeanFactory + i18n + 事件 + 资源 + 分层"心智模型
2. `AbstractApplicationContext.refresh()`（:221-268）——**容器启动编舞的活化石**，背下 8 步
3. `getBean` + `configureManagedObject`（:508-518, 356-364）——组合委托 + Aware 回调
4. 事件三件套 + `publishEvent`（:403-411）——理解冒泡
5. 消息源：`MessageSource` → `NestingMessageSource` → `AbstractNestingMessageSource.getMessage`（:126-155）→ `ResourceBundleMessageSource`
6. XML 实现：`AbstractXmlApplicationContext.refreshBeanFactory` → `FileSystemXmlApplicationContext`（多 location 父子链）→ `ClassPathXmlApplicationContext`
7. 跑 `StaticApplicationContextTestSuite` / `StaticMessageSourceTestSuite` 对照源码
8. 进阶：`BeanFactoryPostProcessor` + `PropertyResourceConfigurer`、`ApplicationObjectSupport`

## 值得精读的源码片段

| 片段 | 行 | 为什么 |
|------|----|--------|
| `ApplicationContext.java` 整接口 | 51-162 | 一眼看清 Context 全部契约 |
| `AbstractApplicationContext#refresh()` | 221-268 | **现代 refresh() 的祖先**；含被注释掉的 TODO，能看到设计演进 |
| `AbstractApplicationContext#publishEvent()` | 403-411 | 事件发布 + 父子冒泡，10 行讲清传播 |
| `AbstractApplicationContext#getBean()` + `configureManagedObject()` | 508-518, 356-364 | 组合委托 + Aware 回调典型写法 |
| `AbstractApplicationContext#initMessageSource()` | 311-326 | 按 bean 名查找 + `setParent` 实现消息分层回退 |
| `AbstractApplicationContext#refreshListeners()` | 370-379 | 自动发现 `ApplicationListener` bean |
| `ApplicationEventMulticasterImpl#onApplicationEvent()` | 42-48 | 最朴素的观察者广播，诚实标注线程安全局限 |
| `AbstractNestingMessageSource#getMessage(String,Object[],Locale)` | 126-155 | 模板方法 + parent 委派 + `MessageFormat` 缓存 + 单引号 escape |
| `AbstractXmlApplicationContext#refreshBeanFactory()` | 44-75 | XmlBeanFactory 创建、EntityResolver 注入、异常转换 |
| `FileSystemXmlApplicationContext` 构造器 | 30-65 | 多 location 递归建父子上下文链 |
| `ResourceBundleMessageSource#resolve()` | 41-58 | "找不到不抛异常、返回 null 让 parent 接手"的关键设计 |
| `PropertyResourceConfigurer#postProcessBeanFactory()` | 42-58 | BeanFactoryPostProcessor 早期实现（属性覆盖，非占位符） |

---

**一句话总结（面试版）**：0.9.1 的 `ApplicationContext` 通过接口多继承 `MessageSource + ListableBeanFactory + HierarchicalBeanFactory` 聚合三种身份，内部组合一个 `ListableBeanFactoryImpl` 做真正的 `getBean`；在 `refresh()` 这个 final 模板方法里规定"加载 BeanFactory → 后置处理 → options → 消息源 → 钩子 → 注册监听器 → 预实例化单例 → 发 ContextRefreshedEvent"的标准启动；通过 `ApplicationEventMulticaster`（自己又是个 Listener）实现同步、会向父上下文冒泡的观察者。**此版还没有** ConfigurableApplicationContext、Environment、`${}` 占位符、统一 Resource 抽象、按类型的事件泛型——都是后续版本逐步加上的。

← 上一节：[03-beans.md](03-beans.md) ｜ 下一节：[05-aop.md](05-aop.md)
