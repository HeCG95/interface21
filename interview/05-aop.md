# 05 · aop 模块学习要点

> 基于 `src/com/interface21/aop/**`（26 个 `.java`）与 `test/com/interface21/aop/**`（12 个）逐字核对编写。
> Java 1.3，无泛型。依赖 `lib/aop-alliance/aopalliance.jar` 与 `lib/cglib/cglib-1.0.jar`。

## ⚠️ 阅读前必读：0.9.1 与现代 Spring AOP 的关键差异（面试加分点）

不要拿现代 Spring（2.0+）的概念往 0.9.1 上套。以下在 **0.9.1 中均不存在**（源码 grep `Advice|Before|After|Throws|Around` 零命中）：

- ❌ 没有独立的 `Pointcut` 接口——切点的根接口叫 **`MethodPointcut`**（方法级）。
- ❌ 没有 `Advisor` / `AdvisorChainFactory`——**`MethodPointcut` 本身就把"切点 + 拦截器"绑在一起**（`getInterceptor()` 直接返回 `MethodInterceptor`）。Pointcut 与 Advice 拆分的 `Advisor` 是后来才有的。
- ❌ 没有 `BeforeAdvice` / `AfterReturningAdvice` / `ThrowsAdvice` / `AroundAdvice`——通知模型**完全对接 AOP Alliance 的 `MethodInterceptor`（环绕通知）**。
- ❌ 没有静态切点缓存——每次方法调用都重建调用链并重新判定（代码里有 TODO "Could cache static pointcut decisions"）。

记住这几点，再读现代 Spring 时你会清楚哪些是演进出来的。

## 模块定位

`com.interface21.aop` 是 Spring 早期 AOP 实现，核心子包 `framework` 的 `package.html` 自述：
> "Package containing Spring's basic AOP infrastructure, **compliant with the AOP Alliance interfaces**."

设计目标：与 AOP Alliance 兼容、同时支持 JDK 动态代理与 CGLIB、支持静态/动态方法切点与正则表达式切点与 Introduction（mixin），并可通过 `BeanFactory` 以 `FactoryBean` 形式声明式使用。

### AOP Alliance 契约（理解一切的基线）
| 接口 | 关键方法 |
|------|---------|
| `Interceptor` | （空标记接口） |
| `MethodInterceptor extends Interceptor` | `Object invoke(MethodInvocation) throws Throwable` |
| `MethodInvocation extends Invocation` | `Method getMethod()` |
| `Invocation extends Joinpoint` | `getArgument/getArguments/getArgumentCount/setArgument/addAttachment/getAttachment/...` |
| `Joinpoint` | `proceed() throws Throwable` / `getThis()` / `getStaticPart()` |

## 子包结构

| 子包 | 职责（据 `package.html`） |
|------|------|
| `framework` | AOP 核心基础设施（代理工厂、切点体系、调用链、Introduction），对接 AOP Alliance |
| `interceptor` | 杂项拦截器实现；更专门的拦截器在 transaction、orm 等功能包 |
| `attributes` | AOP 元数据属性支持——`package.html` 注明 **"Not fully implemented yet."** |

## 核心接口（路径 + 逐字签名）

### 切点体系（均位于 `src/com/interface21/aop/framework/`）

**`MethodPointcut`**（`MethodPointcut.java:21`）—— 0.9.1 切点根接口，**切点与拦截器绑定**：
```java
public interface MethodPointcut {
    MethodInterceptor getInterceptor();   // 返回"有条件运行"的拦截器
    //int getPrecedence();  // 已注释掉
}
```

**`StaticMethodPointcut extends MethodPointcut`**（`StaticMethodPointcut.java:26`）—— 静态切点，**只看方法不看参数**，理论上可在代理构造期一次性判定：
```java
boolean applies(Method m, AttributeRegistry attributeRegistry);
```

**`DynamicMethodPointcut extends StaticMethodPointcut`**（`DynamicMethodPointcut.java:26`）—— 动态切点，**还能看入参**：
```java
boolean applies(Method m, Object[] arguments, AttributeRegistry attributeRegistry);
```
> 注意继承关系：`Dynamic extends Static`（而非并列）。即动态切点必须同时实现静态判定——这是"静态先过筛、动态再细判"两段式判定的接口契约。

### 通知/拦截器接口

- **`MethodInterceptor`**（AOP Alliance）—— 唯一的通知载体（环绕）。
- **`IntroductionInterceptor extends MethodInterceptor`**（`IntroductionInterceptor.java:22`）—— Introduction（mixin）能力：
  ```java
  Class[] getIntroducedInterfaces();
  ```
- **`ProxyInterceptor extends Interceptor`**（`ProxyInterceptor.java:17`）—— 标记"拥有目标对象"的拦截器（**只能出现在链尾**）：
  ```java
  Object getTarget();
  ```

### 代理配置接口 `ProxyConfig`（`ProxyConfig.java:21`）
```java
boolean getExposeInvocation();
AttributeRegistry getAttributeRegistry();
List getMethodPointcuts();
Class[] getProxiedInterfaces();
void addInterceptor(Interceptor interceptor);
void addInterceptor(int pos, Interceptor interceptor);
void addMethodPointcut(MethodPointcut pc);
void addMethodPointcut(int pos, MethodPointcut pc);
boolean removeInterceptor(Interceptor interceptor);
Object getTarget();   // "Will be invoked on each invocation" 需高效
```

### `AopProxy`（`AopProxy.java:41`）—— 注意：**0.9.1 中它是类不是接口**（与后来版本不同），`implements InvocationHandler`
```java
public AopProxy(ProxyConfig config) throws AopConfigException;
public final Object invoke(Object proxy, Method method, Object[] args) throws Throwable;
public Object getProxy();
public Object getProxy(ClassLoader cl);
```

## 核心实现类

### 切点实现
| 类 | 要点 |
|----|------|
| `AbstractMethodPointcut` | 便利基类：持有 `private MethodInterceptor interceptor` 字段 + getter/setter，子类只实现 `applies(...)` |
| `AlwaysInvoked` | `applies()` 恒 `true`。把**无条件执行**的 `MethodInterceptor` 包装成切点入链；`DefaultProxyConfig.addInterceptor(...)` 就用它把裸 `Interceptor` 适配成 `MethodPointcut` |
| `RegexpMethodPointcut` | 正则切点（changelog 的 "regular expression pointcut"）。基于 **Jakarta ORO**（`Perl5Compiler`/`Perl5Matcher`），"Does not require J2SE 1.4"；匹配串为 `declaringClass.getName() + "." + methodName`；语义为**全匹配**（`.*get.*` 匹配 `com.mycom.Foo.getBar()`，`get.*` 不匹配） |

### 代理工厂
| 类 | 要点 |
|----|------|
| `DefaultProxyConfig`（`implements ProxyConfig, InitializingBean`） | `ProxyFactory`/`ProxyFactoryBean` 的共同父类，做切点/接口"管家工作"，**自身不创建代理**。`addInterceptor` 强制参数 `instanceof MethodInterceptor`；`addAspectInterfacesIfNecessary`（:119）对 `IntroductionInterceptor` 自动加入引入接口；`computeTargetAndCheckValidity`（:132）强制 **`ProxyInterceptor` 只能在链尾**并据此推断 target |
| `ProxyFactory extends DefaultProxyConfig` | **编程式**使用。`ProxyFactory(Object target)` 自动探测 target 全部接口并追加 `InvokerInterceptor`；`getProxy()` → `new AopProxy(this).getProxy()` |
| `ProxyFactoryBean`（`implements FactoryBean, BeanFactoryAware`） | **BeanFactory 声明式**使用。`setInterceptorNames` / `setProxyInterfaces` 为 JavaBean 属性；`createInterceptorChain`（:136）把 bean 名物化为链，支持 `GLOBAL_SUFFIX="*"`（:60）展开**全局拦截器**；`addGlobalInterceptorsAndPointcuts`（:207）收集容器内所有 `MethodPointcut`/`Interceptor` bean，用 `OrderComparator` 排序后按前缀过滤注入；`*` 不能放链尾 |

### 调用链与目标拦截器
| 类 | 要点 |
|----|------|
| `MethodInvocationImpl implements MethodInvocation` | AOP Alliance `MethodInvocation` 的 Spring 实现，**整个 AOP 的心脏**。构造器里就地过滤构建调用链（见下文）；`proceed()` 递归驱动 |
| `InvokerInterceptor`（`implements MethodInterceptor, ProxyInterceptor`） | **链尾**目标调用器："should always be the last interceptor in the chain. It does not invoke proceed()"。反射调目标，并把 `InvocationTargetException` 解包为原始异常（:59-64） |
| `DelegatingIntroductionInterceptor implements IntroductionInterceptor` | Introduction 便利基类。构造时用 `AopUtils.findAllImplementedInterfaces` 收集委托全部接口，`suppressInterface(IntroductionInterceptor.class)` 隐藏控制接口；`invoke()` 判断方法声明类是否在已发布接口集：是→反射调 delegate（"breaking interceptor chain"），否→`mi.proceed()` 透传 |

### interceptor 子包（全是 `MethodInterceptor` = 环绕通知实例）
| 类 | 职责 |
|----|------|
| `DebugInterceptor` | 计数 + `System.out.println` 打印调用信息 |
| `PerformanceMonitorInterceptor` | 用 `util.StopWatch` 计时并 info 级记录 |
| `ClassLoaderAnalyzerInterceptor` | 打印类加载器层级 |
| `AbstractQaInterceptor` | 抽象骨架：先 `proceed()` 再 `checkInvariants(target)`——QA/不变式校验模式 |

### attributes 子包
| 类 | 状态 |
|----|------|
| `Attrib4jAttributeRegistry` | **未实现**：`getAttributes` 直接抛 `UnsupportedOperationException` |
| `MapAttributeRegistry` | 以 `Map<AccessibleObject, Object[]>` 存属性，手动注入 |
| `WildcardAttributeRegistry` | 按方法名（含通配）映射到属性；查无返回空数组（永不返回 null） |

## 代理机制：JDK 动态代理 vs CGLIB（据代码）

决策在 `AopProxy.getProxy(ClassLoader cl)`（`AopProxy.java:132-148`），**判据是"是否配置了代理接口"**：
```java
public Object getProxy(ClassLoader cl) {
    if (this.config.getProxiedInterfaces() != null && this.config.getProxiedInterfaces().length > 0) {
        // 有接口 → JDK 动态代理
        return Proxy.newProxyInstance(cl, this.config.getProxiedInterfaces(), this);
    } else {
        // 无接口 → CGLIB（内部类，隔离依赖）
        return (new CglibProxyFactory()).createProxy();
    }
}
```
- CGLIB 路径放在**内部类 `CglibProxyFactory`**（`:198-209`），Javadoc 说明原因：把 CGLIB 依赖隔离在内部类，**使仅用 JDK 代理时无需 `cglib.jar`**。内部类用 `net.sf.cglib.Enhancer.enhance(...)` 建子类，回调转发给外层 `AopProxy.invoke(...)`。
- 限制：CGLIB 要求目标类**没有 final 方法**（运行期建子类）。

**测试佐证**（`AopProxyTests`）：`testProxyIsJustInterface()`（:146）配接口→`Proxy.isProxyClass` 为 true（JDK）；`testProxyCanBeFullClass()`（:159）配空接口→结果 `instanceof TestBean`（CGLIB 子类）。

## 静态 vs 动态切点的运行时区别（据代码）

接口层区别已在上文。**运行时区别的真正实现在 `MethodInvocationImpl` 构造器**（`MethodInvocationImpl.java:94-112`）——这正是 changelog 所说 "distinction between static and dynamic method pointcuts" 的落地点：
```java
this.interceptors = new LinkedList();
for (Iterator iter = pointcuts.iterator(); iter.hasNext();) {
    Object pc = iter.next();
    if (pc instanceof DynamicMethodPointcut) {                 // 先判 Dynamic（它继承自 Static，否则会被 Static 分支误捕）
        DynamicMethodPointcut dpc = (DynamicMethodPointcut) pc;
        if (dpc.applies(m, attributeRegistry) && dpc.applies(m, arguments, attributeRegistry)) {
            this.interceptors.add(dpc.getInterceptor());        // 静态 && 动态 都通过才入链
        }
    } else if (pc instanceof StaticMethodPointcut) {
        StaticMethodPointcut spc = (StaticMethodPointcut) pc;
        if (spc.applies(m, attributeRegistry)) {
            this.interceptors.add(spc.getInterceptor());        // 仅静态判定
        }
    } else {
        throw new AspectException("Unknown pointcut type: " + pc.getClass());
    }
}
```
要点：
- **每次方法调用都重建 `interceptors` 链**并对所有切点重新判定。
- 动态切点 = "静态 && 动态"两段判定；静态切点只判一次静态条件。
- 性能含义：静态切点理论可缓存，但 0.9.1 有 TODO "Could cache static pointcut decisions"（:93），**尚未实现缓存**。

## 关键设计模式与对应类

| 模式 | 对应类 |
|------|--------|
| 代理 | `AopProxy`（JDK `InvocationHandler`）+ 内部类 `CglibProxyFactory` |
| 策略 | `ProxyConfig` 接口 + `AopProxy.getProxy` 的分支 |
| **责任链 / 递归** | `MethodInvocationImpl.proceed()`（:222-231）——核心 |
| 工厂 | `ProxyFactory`（编程式）/ `ProxyFactoryBean implements FactoryBean`（声明式） |
| 模板方法 | `AbstractMethodPointcut`、`AbstractQaInterceptor`、`DelegatingIntroductionInterceptor.invoke` |
| 装饰器 / 混入（Introduction/Mixin） | `IntroductionInterceptor` / `DelegatingIntroductionInterceptor` |
| 适配器 | `AlwaysInvoked`（把裸 `MethodInterceptor` 适配成 `MethodPointcut`） |

**责任链核心**（`MethodInvocationImpl.java:222-231`，逐字）：
```java
public Object proceed() throws Throwable {
    if (this.currentInterceptor >= this.interceptors.size() - 1)
        throw new AspectException("All interceptors have already been invoked");
    // We begin with -1 and increment early
    MethodInterceptor interceptor = (MethodInterceptor) this.interceptors.get(++this.currentInterceptor);
    return interceptor.invoke(this);
}
```
`currentInterceptor` 初值 `-1`，先自增再取。每个 `MethodInterceptor` 在 `invoke(mi)` 内自行决定是否调 `mi.proceed()` 继续链；**链尾 `InvokerInterceptor` 不调 `proceed()` 而是反射调目标**。`AopProxy.invoke`（:103）只调一次 `invocation.proceed()` 启动整条链。

## 对应测试（`test/com/interface21/aop/`）

| 测试类 | 覆盖点 |
|--------|--------|
| `framework/ProxyFactoryTests` | `ProxyFactory`/`DefaultProxyConfig`：null target、无接口、接口去重、自动探测全部接口、`ProxyInterceptor` 必须链尾、只能加 `MethodInterceptor` |
| `framework/AopProxyTests` | 核心：null 配置、空拦截器、拦截器被调用、`exposeInvocation` 开关、target 返回 `this` 时替换为代理、equals 语义、Introduction mixin（`LockMixin`）、动态/静态切点两段判定、参数改写 |
| `framework/RegexpMethodPointcutTests` | 精确匹配 `Object.hashCode`、`.*Object.hashCode`、`Object.*`、跨子类匹配声明类 |
| `framework/MethodInvocationTests` | null/空拦截器报错、合法调用、`proceed` 越界抛 `AspectException`、`addAttachment/getAttachment`、`toString` 不触发目标 |
| `framework/InvokerInterceptorTests` | void 无异常、checked 异常（`ServletException`）、runtime 异常（`NPE`）均按原异常重抛 |
| `framework/DelegatingIntroductionInterceptorTests` | 委托式 Introduction、自动接口识别、`suppressInterface`、Introduction 屏蔽 target 同名接口 |
| `framework/ProxyFactoryBeanTests` | XML 配置（`test.xml`）：单例/原型、自动 `InvokerInterceptor`、全局 `*` 展开、`&factory` 引用、动态增删 Introduction 接口、pointcut 仅命中 void 方法 |
| `attributes/WildcardAttributeRegistryTests` | 查无返回空数组非 null、按方法名取单值/多值 |

> 测试辅助类值得一读：`Lockable`/`LockedException`/`LockMixin`（Introduction demo：锁定后拦截 set 开头方法）。

## 建议学习顺序

1. **AOP Alliance 接口**（`Interceptor` → `MethodInterceptor` → `Joinpoint.proceed()` 契约）——理解一切的基线
2. `MethodPointcut` → `StaticMethodPointcut` → `DynamicMethodPointcut`（切点双继承体系）
3. `MethodInvocationImpl`（构造器的链过滤 + `proceed()` 递归）——**最核心的一份代码**
4. `InvokerInterceptor`（链尾目标调用，理解为什么 `proceed` 会终止）
5. `AopProxy.invoke` + `getProxy`（代理入口 + JDK/CGLIB 分支 + 内部类隔离）
6. `DefaultProxyConfig`（配置管家 + Introduction 自动暴露接口 + 链尾校验）
7. `ProxyFactory`（编程式）→ `ProxyFactoryBean`（声明式 + 全局 `*` 展开 + prototype 刷新）
8. `RegexpMethodPointcut`（静态切点实例 + ORO 正则）
9. `DelegatingIntroductionInterceptor` + 测试侧 `LockMixin`（Introduction/mixin）
10. `interceptor` 子包四个类（环绕通知实例）+ `attributes` 子包（了解"未完成"状态）

## 值得精读的源码片段

- **`MethodInvocationImpl.java:94-112`（构造器链过滤）** —— 静态/动态切点两段判定的唯一实现。**面试"静态切点 vs 动态切点"的硬核答案。**
- **`MethodInvocationImpl.java:222-231`（`proceed`）** —— 责任链递归驱动的最简形态。
- **`AopProxy.java:79-118`（`invoke`）** —— 代理入口：每次新建 invocation、`exposeInvocation` 的 ThreadLocal 上下文、`equals` 拦截、target 返回 this 的替换。
- **`AopProxy.java:132-148`（`getProxy`）+ `:198-209`（`CglibProxyFactory`）** —— JDK/CGLIB 选择策略与 CGLIB 依赖隔离技巧。
- **`DefaultProxyConfig.java:119-126` + `:132-148`** —— Introduction 自动暴露接口 + 链尾 `ProxyInterceptor` 校验。
- **`ProxyFactoryBean.java:136-165` + `:207-232`** —— 声明式链组装 + 全局通配展开 + `OrderComparator` 排序。
- **`InvokerInterceptor.java:47-68`（`invoke`）** —— 链尾反射调用 + `InvocationTargetException` 解包。
- **`RegexpMethodPointcut.java:62-86`** —— ORO Perl5 正则切点，匹配串为 `declaringClass.name + "." + methodName`。
- **`DelegatingIntroductionInterceptor.java:97-106`（`invoke`）** —— Introduction 的"声明类命中则自调 delegate，否则透传"判定。

---

**一句话总结（面试版）**：Spring 0.9.1 的 AOP 紧贴 AOP Alliance `MethodInterceptor`/`MethodInvocation`；`MethodPointcut` 把"条件 + 拦截器"绑在一起，分静态（只看 `Method`）与动态（再看入参）两段判定；`AopProxy` 按"有无接口"选 JDK 动态代理或 CGLIB（CGLIB 隔离在内部类以保持 JDK-only 可用）；调用链由 `MethodInvocationImpl.proceed()` 递归驱动，链尾 `InvokerInterceptor` 反射调目标。此时**还没有 `Advisor`、没有 Before/After/Throws 分离的通知类型、没有静态切点缓存**——这些都是后续版本演进出来的。

← 上一节：[04-context.md](04-context.md) ｜ 下一节：[06-dao.md](06-dao.md)
