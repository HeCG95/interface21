# 02 · util 模块学习要点

> 基于 `src/com/interface21/util/**`（14 个 `.java`，扁平结构）与 `test/com/interface21/util/**`（6 个）逐字核对编写。

## 模块定位

util 是框架自用的**工具集与少量基础设施抽象**：字符串、Ant 风格路径匹配、反射常量解析、计时、性能监控、分页、日志配置、线程绑定对象。它是"core 之上的第一层具体工具"，被 beans/jdbc/web/context 等复用。

> 注意一个反向依赖：`PagedListHolder` 系列依赖 `com.interface21.beans`（util 中**唯一**向 beans 包反向依赖的点）。

## 文件结构（共 14 个）

| 文件 | 一句话 |
|------|--------|
| `StringUtils.java` | 字符串/集合工具（不依赖正则的手写实现） |
| `PathMatcher.java` | Ant 风格路径匹配（`*`/`?`/`**`） |
| `Constants.java` / `ConstantException.java` | 反射读取类的 `public static final` 常量 |
| `ClassLoaderUtils.java` | 类加载器诊断 + ContextClassLoader 优先的资源加载 |
| `StopWatch.java` | 计时器（任务耗时统计） |
| `ResponseTimeMonitor.java` / `ResponseTimeMonitorImpl.java` | 响应时间监控（接口 + 委托实现） |
| `PagedListHolder.java` / `PagedListSourceProvider.java` / `RefreshablePagedListHolder.java` | 分页家族 |
| `Log4jConfigurer.java` | Log4j 配置引导（支持热刷新） |
| `ThreadObjectManager.java` | 线程绑定对象管理（事务资源绑定的雏形） |
| `ObjectArrayUtils.java` | 原始类型→`Object[]` 装箱（为 `ParameterizableErrorCoded` 服务） |

## 核心接口

### `PagedListSourceProvider` — `src/com/interface21/util/PagedListSourceProvider.java`
```java
public interface PagedListSourceProvider {
    public List loadList(Locale locale, Object filter);
}
```
`RefreshablePagedListHolder` 的**加载策略**：按 Locale/filter 重新拉取数据。

### `ResponseTimeMonitor` — `src/com/interface21/util/ResponseTimeMonitor.java`
```java
public interface ResponseTimeMonitor {
    int getAccessCount();
    int getAverageResponseTimeMillis();
    int getBestResponseTimeMillis();
    int getWorstResponseTimeMillis();
}
```
Javadoc 强调实现者可牺牲同步换取性能，"so long as the errors are not likely to be misleading"。

## 核心工具类

### `StringUtils` — `src/com/interface21/util/StringUtils.java`（`public abstract class`）
不可实例化工具类（abstract + 全 static）。关键方法：
>abstract 只是阻止直接 new 当前类，子类可以实例化，所以它不是可靠的防实例化方案

```java
public static int countOccurrencesOf(String s, String sub);
public static String replace(String inString, String oldPattern, String newPattern); // StringBuffer + indexOf，不用正则
public static String delete(String inString, String pattern);                         // 委托 replace(..., "")
public static String[] delimitedListToStringArray(String s, String delimiter);        // 手写切分，保留空 token
public static String[] commaDelimitedListToStringArray(String s);
public static Set commaDelimitedListToSet(String s);                                  // 返回 TreeSet
public static String arrayToDelimitedString(Object[] arr, String delim);
public static String collectionToDelimitedString(Collection c, String delim);
```
> 看点：`delimitedListToStringArray` 注释里保留了被弃用的 `StringTokenizer` 实现作对比——手写切分才能保留空 token（`"a,,b"` → 长度 3），`StringTokenizer` 会丢弃空串。

### `PathMatcher` — `src/com/interface21/util/PathMatcher.java`（`public abstract class`）
```java
public static boolean match(String pattern, String str);
```
Ant 风格：`*` 任意字符、`?` 单字符、`**` 任意层目录。算法"kindly borrowed from Ant"：
- `match` 先按 `**` 分段，从前/后两端向中间**夹逼匹配**，中间段用**滑窗查找**——经典双向贪心。
- `matchStrings(pattern, str)` 单段（不含 `/`）的 `*`/`?` 匹配，基于 `char[]` 双指针。
- 这是后续 **web URL 匹配、AOP pointcut** 的算法源头。

### `Constants` — `src/com/interface21/util/Constants.java`
```java
public class Constants {
    public Constants(Class clazz);                  // 构造期反射读出所有 public static final 字段 → HashMap
    public int getSize();
    public int asInt(String code) throws ConstantException;       // 内部 toUpperCase，大小写不敏感
    public String asString(String code) throws ConstantException;
    public Object asObject(String code) throws ConstantException;
}
```
让 PropertyEditor 等组件用常量**名字**而非魔法数字。典型"元数据驱动"设计（用 `Modifier.isFinal/isStatic/isPublic` 三重过滤 + `f.get(null)` 反射读静态字段）。

### `ClassLoaderUtils` — `src/com/interface21/util/ClassLoaderUtils.java`（`public abstract class`）
```java
public static InputStream getResourceAsStream(Class clazz, String name);   // 先 ContextClassLoader 再回退 clazz
public static String showClassLoaderHierarchy(Object obj, String role, String delim, String tabText);
public static String showClassLoaderHierarchy(ClassLoader cl, String delim, String tabText, int indent);
```
**ContextClassLoader 优先回退策略**——Spring 在 Tomcat/JBoss 类加载体系下避免资源丢失的标准手法。`showClassLoaderHierarchy` 递归打印类加载器父子链，诊断 J2EE 部署问题。

### `StopWatch` — `src/com/interface21/util/StopWatch.java`
```java
public StopWatch();  public StopWatch(String id);
public void start(String task) throws IllegalStateException;
public void stop() throws IllegalStateException;       // 用 running 布尔防误用
public long getTotalTime();  public long getLastInterval() throws IllegalStateException;
public double getTotalTimeSecs();  public int getTaskCount();
public TaskInfo[] getTaskInfo();                      // 内部 public static class TaskInfo
public String shortSummary();  public String prettyPrint();  // 带百分比的表格
```
Javadoc 明确 **"not designed to be threadsafe… safe to invoke from EJBs"**——有意不同步以符合 EJB 规范。

### `ResponseTimeMonitorImpl` — `src/com/interface21/util/ResponseTimeMonitorImpl.java`
```java
public class ResponseTimeMonitorImpl implements ResponseTimeMonitor {
    public final void recordResponseTime(long responseTime);   // final 保证统计逻辑不被子类破坏
    public final long getUptime();  public final Date getLoadDate();
}
```
Javadoc 写明 **"for use via delegation"**——业务对象实现接口、内部委托给本类。典型的"接口 + 委托实现"组合。

### `PagedListHolder` / `RefreshablePagedListHolder` — 分页家族
```java
// PagedListHolder
public static final int DEFAULT_PAGE_SIZE = 10;
public static final int DEFAULT_MAX_LINKED_PAGES = 10;
public List getPageList();            // source.subList(...)
public int getNrOfPages();
public void resort();                 // 通过 beans.PropertyComparator 排序；用 sortUsed 快照做脏检查

// RefreshablePagedListHolder extends PagedListHolder
public void refresh(boolean force);   // locale/filter 变化时经 sourceProvider.loadList(...) 重新加载
```
面向 Web UI 的分页状态豆（pageSize/page/sort），支持数据绑定；`RefreshablePagedListHolder` 是模板方法式的"变化检测 + 重新加载"扩展。

### `Log4jConfigurer` — `src/com/interface21/util/Log4jConfigurer.java`（abstract）
```java
public static final long DEFAULT_REFRESH_INTERVAL = FileWatchdog.DEFAULT_DELAY;
public static final String XML_FILE_EXTENSION = ".xml";
public static void initLogging(String location) throws FileNotFoundException;
public static void initLogging(String location, long refreshInterval) throws FileNotFoundException; // 按扩展名分流 DOM/PropertyConfigurator，均 configureAndWatch
public static void shutdownLogging();
```

### `ThreadObjectManager` — `src/com/interface21/util/ThreadObjectManager.java`
```java
public class ThreadObjectManager {
    public boolean hasThreadObject(Object key);
    public Object getThreadObject(Object key);
    public void bindThreadObject(Object key, Object value);   // 重复绑定抛 IllegalStateException
    public void removeThreadObject(Object key);
}
```
内部用一个 **`ThreadLocal`（覆写 `initialValue()` 返回 `HashMap`）** 持有"每线程一个 Map"。被 `jdbc.datasource.DataSourceTransactionManager` / `DataSourceUtils` 用来按 DataSource+线程绑定事务——**是后来 `TransactionSynchronizationManager` 的雏形**。面试讲"Spring 事务资源与线程绑定"可溯源于此。

### `ObjectArrayUtils` — `src/com/interface21/util/ObjectArrayUtils.java`（abstract）
大量 `public static Object[] toArray(...)` 重载，把原始类型参数装箱为 `Object[]`，专为 `ParameterizableErrorCoded.getErrorArgs()` 服务（作者 Javadoc 自承 "CURRENT LIMITATIONS"，未覆盖全部组合）。

## 关键设计模式与亮点

1. **策略（Strategy）**：`PagedListSourceProvider.loadList(Locale, Object)`。
2. **委托（Delegation）**：`ResponseTimeMonitorImpl`（Javadoc 原话 "for use via delegation"）。
3. **Comparator**：见 core 的 `OrderComparator`。
4. **元数据驱动 / 反射**：`Constants` 构造期扫描静态字段。
5. **模板方法雏形**：`RefreshablePagedListHolder.refresh()` 编排"变化检测→加载→重排"。
6. **不可实例化工具类**：`StringUtils`/`PathMatcher`/`ClassLoaderUtils`/`Log4jConfigurer`/`ObjectArrayUtils` 均 abstract + 仅 static 方法。
7. **线程上下文优先的类加载策略**：`ClassLoaderUtils.getResourceAsStream`。

## 类间协作

- `RefreshablePagedListHolder extends PagedListHolder`，持有 `PagedListSourceProvider`；`PagedListHolder` 依赖 `beans.{BeanUtils, MutableSortDefinition, PropertyComparator, SortDefinition}`（util→beans 唯一反向依赖）。
- `ThreadObjectManager` 被 `jdbc.datasource.DataSourceTransactionManager` / `DataSourceUtils` 使用。
- `Constants` + `ConstantException` 自成一组。
- `ObjectArrayUtils.toArray(...)` → `ParameterizableErrorCoded.getErrorArgs()`。

## 对应测试（6 个，全为 JUnit 3 `TestCase`）

| 测试类 | 覆盖内容 |
|--------|---------|
| `ConstantsTests` | 内嵌含 `DOG=0`/`CAT=66`/String 字段及应被忽略的 protected/包级字段的类；验证 `getSize()`、大小写不敏感、`asInt("bogus")` 抛异常、对 String 字段调 `asInt` 抛异常 |
| `ObjectArrayUtilsTests` | **用反射批量测试**全部 public static 方法：`getMethods()` → 每个方法用准随机参数 + null 参数两次调用并断言。是"用元数据测元数据工具"的典范 |
| `PathMatcherTestSuite` | 单一 `testPathMatcher()` 覆盖精确匹配、`?`/`*`/`**`、跨目录、`/**`、`/*bla*/**/bla/**` 等大量断言 |
| `StopWatchTests` | `testValidUsage`、`testFailureToStartBeforeGettingTimings`、`testFailureToStartBeforeStop`、`testRejectsStartTwice`——重点测状态机异常 |
| `StringUtilsTestSuite` | 计数、replace、delete、CSV↔数组/集合互转、空串保留、往返一致性 |
| `PagedListHolderTests` | 分页/边界、按 name/age 多次 `resort` 验证升降序 toggle、`RefreshablePagedListHolder` 用 `MockSourceProvider`+`MockFilter`+`BeanWrapperImpl` 验证 locale/filter 触发重载 |

> 💡 `ObjectArrayUtilsTests` 和 `PagedListHolderTests` 的测试写法本身就很值得学：前者用反射批量测同构方法，后者综合 Mock + BeanWrapper。

## 建议学习顺序

1. `StringUtils`（最常用，手写字符串算法）
2. `Constants` + `ConstantException`（反射元数据驱动）
3. `StopWatch` + `ResponseTimeMonitor(Impl)`（计时/监控基础设施）
4. `PathMatcher`（Ant 匹配，web/AOP pointcut 的基础）
5. `ClassLoaderUtils`（类加载器诊断 + ContextClassLoader 优先）
6. `ThreadObjectManager`（线程绑定，通向事务管理）
7. `Log4jConfigurer`（日志引导）
8. `PagedListHolder` → `PagedListSourceProvider` → `RefreshablePagedListHolder`（分页家族，依赖 beans，放最后）

测试建议与源码同步读，尤其 `StringUtilsTestSuite`、`PathMatcherTestSuite`、`ObjectArrayUtilsTests`、`PagedListHolderTests`。

## 值得精读的源码片段

- **`PathMatcher.java:40-152`（`match`）与 `:167-298`（`matchStrings`）** — 双向贪心路径匹配核心算法。Spring AOP / web URL 匹配的算法源头，值得逐行读。
- **`Constants.java:45-63`** — 构造器。`Modifier` 三重过滤 + `f.get(null)` 反射读静态字段，元数据驱动极简范例。
- **`StringUtils.java:55-74`** — `replace`。不依赖正则的手写替换。
- **`StringUtils.java:107-135`** — `delimitedListToStringArray`。新旧实现对比，理解为何不用 `StringTokenizer`（保留空 token）。
- **`ThreadObjectManager.java:26-34`** — `ThreadLocal` 覆写 `initialValue()` 返回 `HashMap`，实现"每线程一个 Map"。**事务资源线程绑定的雏形。**
- **`ClassLoaderUtils.java:22-28`** — ContextClassLoader 优先回退策略。
- **`StopWatch.java:87-107` / `:165-178`** — 状态机防误用 API + `prettyPrint` 可读性诊断输出。
- **`test/.../ObjectArrayUtilsTests.java:37-86`** — 反射遍历全部 public static 方法自动造参测试，教学级样板。

---

← 上一节：[01-core.md](01-core.md) ｜ 下一节：[03-beans.md](03-beans.md)
