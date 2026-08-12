# 01 · core 模块学习要点

> 基于 `src/com/interface21/core/**`（9 个 `.java`，无子包，扁平结构）逐字核对编写。
> core 是整个 Spring 的"原子层"——它不依赖任何其它 interface21 包，只定义最通用的契约。

## 模块定位

core 不提供具体功能，只定义**横切所有上层模块的基础契约**：异常嵌套包装、错误码、排序语义、缓存时间戳。`beans`/`jdbc`/`web`/`validation`/`remoting` 等上层包都 import 它。

> ⚠️ 纠偏：网上常说 Spring core 里有 `Resource` 资源抽象、`Visitor` 访问者模式——**本 0.9.1 版本的 core 包里都没有**（`Resource` 此版尚未出现；`Visitor` 只存在于 `sandbox/`，非正式代码）。core 里实际只有下面这些类。

## 文件结构（共 9 个）

| 文件 | 类型 | 一句话 |
|------|------|--------|
| `ErrorCoded.java` | 接口 | 暴露 String 错误码 |
| `ParameterizableErrorCoded.java` | 接口 | 带 `MessageFormat` 占位符参数的错误码 |
| `HasRootCause.java` | 接口 | 暴露根因异常 |
| `NestedCheckedException.java` | 抽象类 | 受检异常的嵌套包装基类 |
| `NestedRuntimeException.java` | 抽象类 | 非受检异常的嵌套包装基类 |
| `InternalErrorException.java` | 类 | 框架自身内部错误 |
| `Ordered.java` | 接口 | 可排序语义 |
| `OrderComparator.java` | 类 | 按 `Ordered` 排序的比较器 |
| `TimeStamped.java` | 接口 | 暴露缓存时间戳 |

## 核心接口（方法签名逐字抄录）

### `ErrorCoded` — `src/com/interface21/core/ErrorCoded.java`
```java
public interface ErrorCoded {
    String getErrorCode();
}
```
让异常暴露一个 String 错误码（如 `"object.failureDescription"`），由 `context.MessageSource` 解析供 i18n。
**设计动机**（Javadoc 原话）：受检异常和非受检异常都有用，但它们无法共享一个框架特定的父类——所以用**接口**而非抽象类横跨两者。

### `ParameterizableErrorCoded` — `src/com/interface21/core/ParameterizableErrorCoded.java`
```java
public interface ParameterizableErrorCoded extends ErrorCoded {
    Object[] getErrorArgs();
}
```
配合 `java.text.MessageFormat` 的 `{0}`、`{1,date}` 占位符语法提供消息参数。

### `HasRootCause` — `src/com/interface21/core/HasRootCause.java`
```java
public interface HasRootCause {
    Throwable getRootCause();
}
```
统一暴露根因异常。Javadoc 明说 **"This will no longer be necessary in Java 1.4"**——这是 Java 1.4 异常链（`Throwable.initCause`）出现前的过渡方案。是面试点"Spring 为什么自建异常体系"的直接出处。

### `Ordered` — `src/com/interface21/core/Ordered.java`
```java
public interface Ordered {
    public int getOrder();
}
```
order 值**小者优先级高**（类比 Servlet `load-on-startup`），`Integer.MAX_VALUE` 表示最后。

### `TimeStamped` — `src/com/interface21/core/TimeStamped.java`
```java
public interface TimeStamped {
    long getTimeStamp();
}
```
标记可缓存对象的新鲜度，返回值等同 `System.currentTimeMillis()`。

## 核心类

### `NestedCheckedException` — `src/com/interface21/core/NestedCheckedException.java`
```java
public abstract class NestedCheckedException extends Exception implements HasRootCause {
    public NestedCheckedException(String msg);
    public NestedCheckedException(String msg, Throwable ex);
    public Throwable getRootCause();
    public String getMessage();                 // 拼接 "; nested exception is: \n\t" + rootCause
    public void printStackTrace(PrintStream ps);
    public void printStackTrace(PrintWriter pw); // 把栈追踪转发给 rootCause
}
```
- abstract 强制子类化；持有 `Throwable rootCause` 并实现 `HasRootCause`。

### `NestedRuntimeException` — `src/com/interface21/core/NestedRuntimeException.java`
```java
public abstract class NestedRuntimeException extends RuntimeException implements HasRootCause { /* 签名与上对称 */ }
```
Javadoc 直言两类重复 **"unavoidable, as Java forces these two classes to have different superclasses"**。这是整个 Spring 异常体系的根（`BeansException`、`DataAccessException` 等都继承自这两个之一）。

### `InternalErrorException` — `src/com/interface21/core/InternalErrorException.java`
```java
public class InternalErrorException extends NestedRuntimeException {
    public InternalErrorException();
    public InternalErrorException(String msg);
    public InternalErrorException(String msg, Throwable ex);
}
```
表示框架自身内部错误（作者 Isabelle Muszynski，2003-04-05）。

### `OrderComparator` — `src/com/interface21/core/OrderComparator.java`
```java
public class OrderComparator implements Comparator {
    public int compare(Object o1, Object o2);
}
```
按 `Ordered.getOrder()` 升序排；非 `Ordered` 对象按 `Integer.MAX_VALUE` 处理（排末尾）。
注释特意说明用**直接 int 比较而非 `Integer.compareTo`** 来避免创建对象（早期 JDK 无自动装箱优化的写法）。

## 关键设计模式与亮点

1. **标记/契约接口（Marker Interface）**：`ErrorCoded`、`HasRootCause`、`Ordered`、`TimeStamped` 都是无方法或单方法的契约，跨继承树暴露统一能力。
2. **异常包装（Exception Wrapping）**：`Nested*Exception` 持有 `rootCause`，`printStackTrace` 委托根因——Java 1.4 异常链的过渡方案。
3. **Comparator 抽象**：`OrderComparator` 把 `Ordered` 语义抽成可复用的 `Comparator`，原始 int 比较避免装箱。

## 类间协作

- `NestedCheckedException` / `NestedRuntimeException` 都 `implements HasRootCause`；`InternalErrorException extends NestedRuntimeException`。
- `ParameterizableErrorCoded extends ErrorCoded`；`util.ObjectArrayUtils.toArray(...)` 产出的 `Object[]` 正是给 `ParameterizableErrorCoded.getErrorArgs()` 用。
- `OrderComparator` 消费 `Ordered`。
- **`ErrorCoded` 是跨模块的错误码契约**：被 `beans.ErrorCodedPropertyVetoException`、`jdbc` 的 `SQLExceptionTranslater` 系列、`validation.DataBinder`、`web.BindTag` 等广泛实现/消费。

## 对应测试

core 模块**没有专属测试**。相关行为在 util 测试中顺带覆盖（如异常消息拼接）。建议读 `test/com/interface21/util/*` 时关注 `ErrorCoded`/`HasRootCause` 的使用样例。

## 建议学习顺序

1. `HasRootCause` + `NestedCheckedException` / `NestedRuntimeException`（异常体系根基，为何要为 Java 1.3 自建异常链）
2. `ErrorCoded` + `ParameterizableErrorCoded`（错误码契约，顺带看 `util/ObjectArrayUtils` 如何为其产参）
3. `Ordered` + `OrderComparator`（最小的排序语义闭环）
4. `TimeStamped` + `InternalErrorException`（标记接口与异常子类的两种用法）

## 值得精读的源码片段

- **`NestedCheckedException.java:74-109`** — `getMessage()` / `printStackTrace(...)`。看 Spring 在 Java 1.3 无原生异常链时如何手工拼接消息并把栈追踪委托给根因。**面试高频点"Spring 为什么有 NestedException 层次"直接对应这里。**
- **`OrderComparator.java:19-30`** — `compare`。注释里"avoid unnecessary object creation"体现早期 JDK 写法。

---

← 上一节：[GUIDE.md](GUIDE.md) ｜ 下一节：[02-util.md](02-util.md)
