# 03 · beans 模块学习要点 ⭐

> 基于 `src/com/interface21/beans/**`（60 个 `.java`）与 `test/com/interface21/beans/**`（27 个）逐字核对编写。
> **这是整个 Spring 的灵魂，务必反复精读。**

## ⚠️ 阅读前必读：0.9.1 与现代 Spring 的关键差异（面试防坑）

经全量检索确认，以下现代概念在 0.9.1 的 beans 包**均不存在**：

| 现代 Spring 概念 | 0.9.1 beans 包的真实情况 |
|------------------|--------------------------|
| `BeanDefinition` 接口 | **不存在**。Bean 定义是抽象类 `AbstractBeanDefinition` |
| `BeanDefinitionRegistry` | **不存在**。注册方法直接长在工厂上：`ListableBeanFactoryImpl.registerBeanDefinition(String, AbstractBeanDefinition)` |
| `DefaultListableBeanFactory` | **不存在**。对应实现叫 `ListableBeanFactoryImpl` |
| `BeanPostProcessor` | **不存在**。0.9.1 的 beans 包**没有 Bean 级别的后置处理扩展点** |
| `BeanFactoryPostProcessor` | beans 包内**没有**；它定义在 `com.interface21.context.support`（属 context 层扩展） |
| 三级缓存 / ObjectFactory | **不存在**。单例缓存只有一级：`AbstractBeanFactory.singletonCache` |

> ⚠️ `src/com/interface21/beans/factory/support/CxsFactory.java` 是后人添加的**练习空壳**（注释署名"玄灭"），非 Spring 原版代码，分析时跳过。

## 模块定位

beans 是整个 Spring 的 **IoC 内核**。核心契约 `BeanFactory` 自述（since 13 April 2001）：
> "The BeanFactory is a central registry of application components, and centralizes the configuring of application components."

它把"组件装配"从应用代码中剥离：Bean 定义可来自 XML / Properties / Map / LDAP 等任意源，容器负责实例化、属性注入、生命周期回调。context、aop、jdbc、web、transaction 等上层模块都建立在 BeanFactory 之上。

## 子包结构

| 包 | 职责 | 代表类 |
|----|------|--------|
| `com.interface21.beans`（顶层） | JavaBean 操纵基础设施：属性访问、类型转换、内省缓存、属性值模型、异常 | `BeanWrapper(Impl)`、`CachedIntrospectionResults`、`PropertyValue(s)`、`BeanUtils`、`BeansException` |
| `beans.propertyeditors` | 自定义 `PropertyEditor` | `ClassEditor`、`CustomBooleanEditor`、`CustomNumberEditor`、`StringArrayPropertyEditor` |
| `beans.factory` | IoC 容器**契约层**：核心接口 + 生命周期扩展接口 + 异常 | `BeanFactory`、`ListableBeanFactory`、`HierarchicalBeanFactory`、`FactoryBean`、`InitializingBean`、`DisposableBean`、`BeanFactoryAware` |
| `beans.factory.support` | IoC 容器**实现层**：BeanDefinition、AbstractBeanFactory、ListableBeanFactoryImpl、工具、bootstrap | `AbstractBeanFactory`、`ListableBeanFactoryImpl`、`AbstractBeanDefinition`/`Root`/`Child`、`BeanFactoryUtils`、`BeanFactoryBootstrap` |
| `beans.factory.xml` | XML 配置解析 | `XmlBeanFactory`、`BeansDtdResolver`、`spring-beans.dtd` |

## 核心接口（逐字签名）

### `BeanFactory` — IoC 根接口 — `factory/BeanFactory.java`
```java
Object getBean(String name) throws BeansException;
Object getBean(String name, Class requiredType) throws BeansException;
boolean isSingleton(String name) throws NoSuchBeanDefinitionException;
String[] getAliases(String name) throws NoSuchBeanDefinitionException;
```
`getBean(name, requiredType)` 用 `requiredType.isAssignableFrom(bean.getClass())` 校验，不匹配抛 `BeanNotOfRequiredTypeException`。返回实例可能共享（singleton）也可能独立（prototype），"API is the same"。

### `ListableBeanFactory` — 可枚举 — `extends BeanFactory`
```java
int getBeanDefinitionCount();
String[] getBeanDefinitionNames();
String[] getBeanDefinitionNames(Class type);
```
**不考虑 hierarchy**（跨层用 `BeanFactoryUtils`）。`getBeanDefinitionNames(Class)` 对 FactoryBean **按工厂自身类型计，不按 `getObject()` 产物类型计**。

### `HierarchicalBeanFactory` — 分层 — `extends BeanFactory`（since 07-Jul-2003）
```java
BeanFactory getParentBeanFactory();
```

### `FactoryBean` — 工厂 Bean — `factory/FactoryBean.java`（since 08-Mar-2003）
```java
Object getObject() throws BeansException;
boolean isSingleton();
PropertyValues getPropertyValues();
```
实现此接口的 bean "cannot be used as a normal bean"。`getBean("x")` 返回 `getObject()` 产物；`getBean("&x")`（`FACTORY_BEAN_PREFIX="&"`）返回工厂本身。

### `BeanFactoryAware` — 注入容器引用 — `factory/BeanFactoryAware.java`
```java
void setBeanFactory(BeanFactory beanFactory) throws Exception;
```
> ⚠️ javadoc 明确"If the bean also implements InitializingBean, this method will be invoked **after** InitializingBean's `afterPropertiesSet`"——顺序是 `afterPropertiesSet` → init-method → **setBeanFactory**。**这与现代 Spring 完全相反**（现代版 Aware 回调在最前）。

### `InitializingBean` / `DisposableBean`
```java
void afterPropertiesSet() throws Exception;   // InitializingBean
void destroy() throws Exception;              // DisposableBean（since 12.08.2003）
```

### `BeanWrapper` — JavaBean 操纵核心接口 — `beans/BeanWrapper.java`（since 13 April 2001）
常量 `String NESTED_PROPERTY_SEPARATOR = "."`。关键签名：
```java
void setPropertyValue(String propertyName, Object value) throws PropertyVetoException, BeansException;
void setPropertyValue(PropertyValue pv) throws PropertyVetoException, BeansException;
Object getPropertyValue(String propertyName) throws BeansException;
Object getIndexedPropertyValue(String propertyName, int index) throws BeansException;
void setPropertyValues(PropertyValues pvs) throws BeansException;
void setPropertyValues(PropertyValues pvs, boolean ignoreUnknown, PropertyValuesValidator pvsValidator) throws BeansException;
PropertyDescriptor[] getPropertyDescriptors() throws BeansException;
PropertyDescriptor getPropertyDescriptor(String propertyName) throws BeansException;
boolean isReadableProperty(String propertyName);
boolean isWritableProperty(String propertyName);
Object getWrappedInstance();  void setWrappedInstance(Object obj) throws BeansException;
void newWrappedInstance() throws BeansException;
Class getWrappedClass();
void registerCustomEditor(Class requiredType, String propertyPath, PropertyEditor propertyEditor);
PropertyEditor findCustomEditor(Class requiredType, String propertyPath);
boolean isEventPropagationEnabled();  void setEventPropagationEnabled(boolean flag);
Object invoke(String methodName, Object[] args) throws BeansException;
```
支持**嵌套属性**到无限深度（`"foo.bar"` → `getFoo().getBar()`）；批量更新遇可恢复错误会继续，最后抛 `PropertyVetoExceptionsException` 汇总。

### 属性值模型
```java
// PropertyValues（接口）
PropertyValue[] getPropertyValues();
boolean contains(String propertyName);
PropertyValue getPropertyValue(String propertyName);
PropertyValues changesSince(PropertyValues old);
// PropertyValuesValidator（接口）
void validatePropertyValues(PropertyValues pvs) throws InvalidPropertyValuesException;
```
`PropertyValue` 是简单 POJO：`PropertyValue(String name, Object value)`。value 不必是最终类型，转换由 BeanWrapper 负责。

## 核心实现类

### `AbstractBeanFactory` — 模板方法 + 单例缓存 — `support/AbstractBeanFactory.java`
`implements HierarchicalBeanFactory`。javadoc 原文："This class uses the **Template Method** design pattern. Subclasses must implement only the `getBeanDefinition(name)` method."

```java
public static final String FACTORY_BEAN_PREFIX = "&";
private BeanFactory parentBeanFactory;
private Map singletonCache = new HashMap();   // 单例缓存（唯一一级）
private Map aliasMap = new HashMap();

public final Object getBean(String name);
private Object getBeanInternal(String name, Map newlyCreatedBeans);    // newlyCreatedBeans 是方法局部 Map，解析单例循环依赖
private final synchronized Object getSharedInstance(String pname, Map newlyCreatedBeans) throws BeansException;
private Object createBean(String name, Map newlyCreatedBeans) throws BeansException;
private void applyPropertyValues(BeanWrapper bw, PropertyValues pvs, String name, Map newlyCreatedBeans);
private Object resolveValueIfNecessary(BeanWrapper bw, Map newlyCreatedBeans, PropertyValue pv) throws BeansException;
private void callLifecycleMethodsIfNecessary(Object bean, String name, RootBeanDefinition rbd, BeanWrapper bw) throws BeansException;
protected final RootBeanDefinition getMergedBeanDefinition(String name) throws NoSuchBeanDefinitionException;
protected abstract AbstractBeanDefinition getBeanDefinition(String beanName) throws NoSuchBeanDefinitionException;
```

**设计要点（以代码为准，勿套现代术语）**：
- **单例缓存只有一级** `singletonCache`。`getSharedInstance` 整方法 `synchronized`。
- **循环依赖解析机制**：`getBeanInternal` 接受方法局部的 `newlyCreatedBeans` Map。`createBean` 在实例化得到原始对象后、属性注入前，先把半成品 `put` 进 `newlyCreatedBeans`（`FactoryBean` 实例被显式跳过）。递归解析引用时先查 `newlyCreatedBeans`，命中则返回半成品。**这是 0.9.1 的循环依赖方案，没有"三级缓存"**。
- `getMergedBeanDefinition`：`RootBeanDefinition` 直接深拷；`ChildBeanDefinition` 递归取父定义再 `merge(parent, overrides)`。
- `resolveValueIfNecessary` 区分 `RuntimeBeanReference` / `ManagedList` / `ManagedMap` / 普通值。

### `ListableBeanFactoryImpl` — "DefaultListableBeanFactory 的前身" — `support/`
`extends AbstractBeanFactory implements ListableBeanFactory`。
```java
private Map beanDefinitionMap = new HashMap();   // bean 定义注册表（无独立 Registry 接口）
public final void registerBeanDefinition(String prototypeName, AbstractBeanDefinition beanDefinition);
public void preInstantiateSingletons();
public final int registerBeanDefinitions(Map m, String prefix) throws BeansException;          // Properties/Map 驱动装配
public final int registerBeanDefinitions(ResourceBundle rb, String prefix) throws BeansException;
public void registerAdditionalPropertyValue(String beanName, PropertyValue pv) throws BeansException;
```
- **编程式装配**：除 XML 外还支持 `java.util.Map`/`ResourceBundle`。约定 key 形如 `beans.<name>.class=...`、`<name>.<prop>=...`、`<name>.<prop>(ref)=target`、`<name>.(singleton)=false`、值以 `*` 开头表示引用（`**` 转义）。这套 API 后来被废弃。
- `preInstantiateSingletons` 遍历所有定义对 singleton 调 `getBean` 触发实例化。

### `XmlBeanFactory` — XML 解析入口 — `xml/XmlBeanFactory.java`
`extends ListableBeanFactoryImpl`。
```java
public XmlBeanFactory(String filename) throws BeansException;
public XmlBeanFactory(InputStream is, BeanFactory parentBeanFactory) throws BeansException;
public void loadBeanDefinitions(String filename) throws BeansException;
public void loadBeanDefinitions(Document doc) throws BeansException;
```
继承链：`XmlBeanFactory → ListableBeanFactoryImpl → AbstractBeanFactory`。JAXP/DOM 解析，`setValidating(true)`，默认 `BeansDtdResolver` 解析 DTD 实体。

### BeanDefinition 体系（`support/`）
| 类 | 要点 |
|----|------|
| `AbstractBeanDefinition`（abstract） | 字段 `boolean singleton`、`PropertyValues pvs`。⚠️ **真实 bug**：`equals(Object)` 第 78 行写的是 `this.singleton = obd.singleton && ...`（赋值 `=`，不是比较 `==`）。 |
| `RootBeanDefinition extends AbstractBeanDefinition` | 含 `Class clazz`、`String initMethodName`、`String destroyMethodName`；有**深拷贝构造器** `RootBeanDefinition(RootBeanDefinition other)` |
| `ChildBeanDefinition extends AbstractBeanDefinition` | 含 `String parentName`；表示"类由父定义决定 + 属性可覆盖父定义"（配置继承，非 Java 继承） |

### `BeanWrapperImpl` — IoC 属性引擎内核 — `beans/BeanWrapperImpl.java`
```java
public BeanWrapperImpl(Object object) throws BeansException;
public BeanWrapperImpl(Class clazz) throws BeansException;   // 实例化并包装
public Object doTypeConversionIfNecessary(Object target, String propertyName, Object oldValue, Object newValue, Class requiredType) throws BeansException;
```
- **静态默认 editor 注册表** + **editor 搜索路径**：
  ```java
  PropertyEditorManager.setEditorSearchPath(new String[] { "sun.beans.editors", "com.interface21.beans.propertyeditors" });
  defaultEditors.put(String[].class, new StringArrayPropertyEditor());
  defaultEditors.put(PropertyValues.class, new PropertyValuesEditor());
  defaultEditors.put(Properties.class, new PropertiesEditor());
  defaultEditors.put(Class.class, new ClassEditor());
  defaultEditors.put(Locale.class, new LocaleEditor());
  ```
  类型转换链：自定义 editor → `defaultEditors` → `PropertyEditorManager.findEditor`。
- **内省缓存** `CachedIntrospectionResults`，`setObject` 时仅在新对象类不同时刷新。
- **嵌套属性** `getBeanWrapperForNestedProperty` 递归导航，`nestedBeanWrappers` 缓存子 BeanWrapper。
- **事件传播**默认关闭（`DEFAULT_EVENT_PROPAGATION_ENABLED = false`）。
- 批量设置遇 veto / 类型不匹配时**继续**，最后汇总成 `PropertyVetoExceptionsException`，合法属性仍生效。

### `CachedIntrospectionResults` — 内省缓存（工厂模式）— `beans/`
`final class`（包级可见）。javadoc 原文："implements the factory design pattern, using a private constructor and a public static `forClass()` method"。
```java
private static Map $cache = new HashMap();   // Class → 结果或异常
protected static CachedIntrospectionResults forClass(Class clazz) throws BeansException;
private CachedIntrospectionResults(Class clazz) throws BeansException;
```
动机：JDK 1.3 的 `Introspector.getBeanInfo()` 每次返回深拷贝代价高，故按 Class **静态缓存** `BeanInfo`，并额外按名字缓存 `PropertyDescriptor`/`MethodDescriptor`。失败结果也缓存，下次直接抛同一异常。

### 其余 support 类
| 类 | 要点 |
|----|------|
| `StaticListableBeanFactory implements ListableBeanFactory` | singleton-only 编程式工厂，`addBean(name, bean)` 直接注册现成实例 |
| `BeanFactoryUtils`（abstract） | `beansOfType`、`beansOfTypeIncludingAncestors`、`countBeansIncludingAncestors`、`beanNamesIncludingAncestors`——跨祖先查询，`HashSet` 去重 |
| `AbstractFactoryBean implements FactoryBean` | `FactoryBean` 便捷基类，默认 singleton=true |
| `BeanFactoryBootstrap` | "One singleton to rule them all"，从 JVM **系统属性**装配 `bootstrapBeanFactory` |
| `BeanFactoryLoader`（接口） | `BeanFactory loadBeanFactory() throws BootstrapException;`，供 EJB 加载工厂 |
| `RuntimeBeanReference` | 不可变占位符，持有 `String beanName`，表示 PropertyValue 的值是运行时 bean 引用 |
| `ManagedList extends LinkedList` / `ManagedMap extends HashMap` | 标记子类，让 `resolveValueIfNecessary` 用 `instanceof` 识别需引用解析的集合 |

## Bean 生命周期（据代码逐环节，必背）

顺序严格来自 `AbstractBeanFactory.callLifecycleMethodsIfNecessary` 与 `createBean`：

```
getBean(name)
 └─ getBeanInternal(name, newlyCreatedBeans=null)
     ├─ getBeanDefinition(transformedBeanName)
     ├─ 若 singleton:
     │    ├─ 查 newlyCreatedBeans（解析单例循环依赖）
     │    └─ getSharedInstance → 未命中则 createBean:
     │         1. getMergedBeanDefinition(name)              // 合并父子定义
     │         2. new BeanWrapperImpl(beanClass)             // 反射实例化（clazz.newInstance()）
     │         3. newlyCreatedBeans.put(name, bean)          // 提前暴露半成品（FactoryBean 跳过）
     │         4. applyPropertyValues(...)                   // 解析引用 + 深拷贝 + bw.setPropertyValues
     │         5. callLifecycleMethodsIfNecessary:
     │              a. (InitializingBean) afterPropertiesSet()
     │              b. (initMethodName!=null) bw.invoke(initMethod, null)
     │              c. (BeanFactoryAware) setBeanFactory(this)   ← Aware 在最后！
     │         └─ singletonCache.put(name, beanInstance)
     │         ├─ 若 & 前缀但非 FactoryBean → BeanIsNotAFactoryException
     │         └─ 若 FactoryBean 且非 & 前缀 → factory.getObject() + 透传 factory.getPropertyValues()
     └─ 若 prototype: createBean(...)（不入缓存，每次新建）

destroySingletons():
 └─ 遍历 singletonCache，对每个 bean:
      ├─ (DisposableBean) destroy()                         // 先于 destroy-method
      └─ (destroyMethodName!=null) bw.invoke(destroyMethod)
 然后 singletonCache.clear()
```

**关键面试点**：
1. **实例化**：`BeanUtils.instantiateClass(clazz)` 即 `clazz.newInstance()`，要求无参 public 构造器。
2. **生命周期回调顺序**：`afterPropertiesSet` → init-method → **setBeanFactory**（与现代 Spring 相反！）。测试 `LifecycleBean` 用 `if (!inited) throw` 强制锁定此顺序。
3. **销毁只对 singleton**：DTD 注释明确"destroy-method: Only invoked on singleton beans!"。销毁顺序：`DisposableBean.destroy()` → destroy-method。
4. **prototype 不缓存、不销毁**。
5. **单例循环依赖能解**（`newlyCreatedBeans` 局部 Map），**FactoryBean 循环依赖不能解**（抛 `StackOverflowError`，已知限制）。

## XML 解析机制

### DTD（`xml/spring-beans.dtd`）核心
```xml
<!ELEMENT beans ( bean+ )>
<!ELEMENT bean ( property* )>
<!ATTLIST bean id ID #REQUIRED
               class CDATA #IMPLIED
               parent CDATA #IMPLIED
               init-method CDATA #IMPLIED
               destroy-method CDATA #IMPLIED
               singleton CDATA #IMPLIED
               name CDATA #IMPLIED>
<!ELEMENT property ( value | ref | list | map | props )>
<!ELEMENT ref EMPTY>
<!ATTLIST ref bean IDREF #IMPLIED        <!-- 同工厂内部引用，DTD 可校验 -->
              external CDATA #IMPLIED>    <!-- 外部（父）工厂引用，仅运行时校验 -->
```
要点：`id` 是 `ID` 类型（必填），非法字符名用 `name`（自动建别名）；默认 singleton；`class` 与 `parent` 二选一；ref 分 `bean`（同工厂、`IDREF` 校验）与 `external`（跨工厂、运行时解析）。

### 解析流程（`XmlBeanFactory` + `BeansDtdResolver`）
1. 构造器调 `loadBeanDefinitions`。
2. JAXP `DocumentBuilderFactory`，`setValidating(true)`，挂 `BeansErrorHandler`（error/fatalError 抛 SAXException，warning 仅 log）。
3. `BeansDtdResolver.resolveEntity`：只要 publicId/systemId 含 `spring-beans`，就从 classpath 读 DTD，**避免联网下载**。
4. `loadBeanDefinitions(Document)`：`getElementsByTagName("bean")` 取全部 bean 元素。
5. `parseBeanDefinition` 依有无 `class`/`parent` 创建 `RootBeanDefinition`（`Class.forName` 用线程上下文 ClassLoader）或 `ChildBeanDefinition`；`registerBeanDefinition(id, bd)`；有 `name` 则 `registerAlias`。
6. `parsePropertySubelement` 按 tag 分派：`ref`→`RuntimeBeanReference`、`value`→文本、`list`→`ManagedList`、`map`→`ManagedMap`、`props`→`Properties`；特殊属性 `distinguishedValue="null"` 表示显式 null。

## 关键设计模式（对应类）

| 模式 | 类 | 证据 |
|------|----|------|
| 工厂方法 | `BeanFactory`、`FactoryBean`、`BeanFactoryLoader` | 接口即工厂；`FactoryBean.getObject()` 是工厂方法 |
| 模板方法 | `AbstractBeanFactory` | javadoc 原文 "uses the Template Method design pattern"；抽象钩子 `getBeanDefinition(String)` |
| 静态工厂 + 私有构造 | `CachedIntrospectionResults.forClass(Class)` | javadoc 原文 "implements the factory design pattern" |
| 注册表 | `ListableBeanFactoryImpl.beanDefinitionMap` + `registerBeanDefinition` | 注意：**无独立 Registry 接口**，注册能力长在工厂上 |
| 观察者/事件 | `BeanWrapperImpl` 的 `PropertyChangeSupport`/`VetoableChangeSupport` | 标准 JavaBean 事件；`PropertyVetoExceptionsException` 汇总 veto |
| 标记类型 | `ManagedList`/`ManagedMap`、`RuntimeBeanReference` | 用 `instanceof` 在 `resolveValueIfNecessary` 区分处理 |
| 原型/单例 | `singletonCache` vs `createBean` 每次 new | BeanFactory javadoc 原文即点出 Prototype/Singleton 模式 |
| 策略 | `EntityResolver`(`BeansDtdResolver`)、`PropertyValuesValidator` | 可替换的解析/校验策略 |

## 对应测试（`test/com/interface21/beans/**`，27 个）

测试组织用**模板方法**：抽象基类定义与实现无关的 IoC 契约，`XmlBeanFactory` 与 `ListableBeanFactoryImpl` 跑同一套行为用例。

| 测试类 | 覆盖场景 |
|--------|----------|
| `AbstractBeanFactoryTests`（abstract） | 钩子 `getBeanFactory()`；继承、`InitializingBean` 回调、生命周期、类型不匹配（`PropertyVetoExceptionsException.getExceptionCount()` + 合法属性仍生效）、FactoryBean 单例/原型、`&` 前缀取工厂本身、别名 |
| `AbstractListableBeanFactoryTests` | `assertCount(15)`、按类型查定义（FactoryBean 按工厂类型计） |
| `XmlBeanFactoryTestSuite` | 父子工厂、`testSingletonInheritsFromParentFactoryPrototype`（singleton/prototype 标记不参与继承）、**`testCircularReferences`**（jenny↔david、ego 自引用——证实已支持单例循环依赖）、`testFactoryReferenceCircleDoesNotWork`（FactoryBean 自循环抛 StackOverflow）、集合注入、初始化顺序（`afterPropertiesSet→customInit→destroy→customDestroy`） |
| `ListableBeanFactoryImplTestSuite` | **Properties 驱动装配**：`(ref)` 语法、`(singleton)=false`、`*r` 引用语法、`**` 转义、`preInstantiateSingletons` |
| `BeanFactoryBootstrapTests` | 系统属性 bootstrap 失败路径 |
| `BeanFactoryUtilsTests` | 三层 context（root/middle/leaf.xml）；同名覆盖去重 |
| `BeanWrapperTestSuite` | 嵌套属性（`spouse.spouse.age`、`NullValueInNestedPathException`）、批量错误收集、事件 veto、方法调用、类型转换 |
| `CustomEditorTestSuite` | editor 注册粒度、`CustomBooleanEditor(true/false)`、`CustomNumberEditor`+GERMAN NumberFormat、`StringTrimmerEditor` |

夹具：`TestBean`、`DummyFactory`、`LifecycleBean`（`implements InitializingBean, BeanFactoryAware, DisposableBean`）等。

## 建议学习顺序

1. `BeanFactory`（契约心智）
2. `BeanWrapper` + `BeanWrapperImpl`（属性访问/类型转换/嵌套属性——IoC 的"引擎"）
3. `CachedIntrospectionResults`（对 JDK 内省的缓存）
4. `PropertyValue(s)` / `MutablePropertyValues`（属性值模型）
5. `AbstractBeanDefinition` / `Root` / `Child`（Bean 定义模型）
6. **`AbstractBeanFactory`（重中之重）**——跟读 `getBeanInternal` → `getSharedInstance` → `createBean` → `applyPropertyValues` → `callLifecycleMethodsIfNecessary` → `destroySingletons`，把生命周期与循环依赖走通
7. `ListableBeanFactoryImpl`（注册表 + 批量查询 + preInstantiateSingletons）
8. `spring-beans.dtd` → `XmlBeanFactory` → `BeansDtdResolver`（XML 装配链）
9. `propertyeditors/*` + `BeanWrapperImpl` 静态块（类型转换）
10. 测试：先 `AbstractBeanFactoryTests`（契约），再 `XmlBeanFactoryTestSuite`（XML 如何兑现契约）

## 值得精读的源码片段

| 片段 | 为什么 |
|------|--------|
| `AbstractBeanFactory.java:getBeanInternal(String, Map)` | 单例/原型分叉 + 父工厂 fallback + **循环依赖通过 `newlyCreatedBeans` 局部 Map 解析**的全貌 |
| `AbstractBeanFactory.java:createBean(String, Map)` | 实例化→提前暴露半成品→注入→生命周期回调完整 5 步 |
| `AbstractBeanFactory.java:callLifecycleMethodsIfNecessary(...)` | **0.9.1 独有的生命周期顺序**（`afterPropertiesSet` → init-method → setBeanFactory），浓缩一个面试考点 |
| `AbstractBeanFactory.java:getMergedBeanDefinition(String)` | bean 定义继承的递归合并 |
| `AbstractBeanFactory.java:getSharedInstance(String, Map)` | 一级单例缓存 + FactoryBean 解引用 + 透传 propertyValues |
| `BeanWrapperImpl.java:doTypeConversionIfNecessary(...)` | String→目标类型的 editor 查找三级链 |
| `BeanWrapperImpl.java:setPropertyValue(PropertyValue)` | 反射 writeMethod + 事件 veto + 异常翻译 |
| `BeanWrapperImpl.java:getBeanWrapperForNestedProperty` | 嵌套属性导航与子 wrapper 缓存 |
| `CachedIntrospectionResults.java:forClass(Class)` | 静态内省缓存 + 工厂模式 + 失败结果也缓存 |
| `XmlBeanFactory.java:parsePropertySubelement(Element)` | XML 子元素到 `RuntimeBeanReference`/`ManagedList`/`ManagedMap`/`Properties` 的分派 |
| `ListableBeanFactoryImpl.java:registerBeanDefinitions(Map, String)` | 早期 Properties 驱动装配的解析约定 |
| `XmlBeanFactoryTestSuite:testCircularReferences` vs `testFactoryReferenceCircleDoesNotWork` | 一组对照：单例循环依赖能解、FactoryBean 循环依赖不能解 |
| `AbstractBeanDefinition.java:equals`（:78） | **真实 bug**：`=` 误作 `==`，体会早期代码也难免笔误 |

---

**一句话总结（面试口诀）**：Spring 0.9.1 的 beans 已确立 `BeanFactory` / `BeanWrapper` / `BeanDefinition` 三件套与"实例化→注入→`afterPropertiesSet`→init-method→`setBeanFactory`→销毁"的生命周期骨架，并具备单例循环依赖解析（`newlyCreatedBeans` 局部 Map）、FactoryBean、分层工厂、bean 定义继承、XML 装配等核心能力；但它**没有** `BeanPostProcessor`、`BeanDefinitionRegistry`、`DefaultListableBeanFactory`、三级缓存，后置处理放在 `context.support` 层，Aware 回调在生命周期最后——这些"缺失与差异"本身就是最好的面试考点。

← 上一节：[02-util.md](02-util.md) ｜ 下一节：[04-context.md](04-context.md)
