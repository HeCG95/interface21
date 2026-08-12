# 14 · remoting 模块学习要点

> 基于 `src/com/interface21/remoting/**`（12 个 `.java`）与 `test/com/interface21/remoting/**`（1 个）逐字核对编写。
> 把 RMI / Hessian / Burlap 三种异构 RPC 协议统一成"调用本地业务接口"的编程模型。

## 模块定位

客户端拿到的就是普通 Java 接口代理，协议差异被压缩为一行配置；服务端 Hessian/Burlap 以 Web Controller 暴露，RMI 以注册表暴露。

**核心哲学**（见 `RemoteAccessException` Javadoc）：业务接口不必 `extends java.rmi.Remote`、不必声明 `throws RemoteException`；协议切换只是配置改动；客户端代码"不显含任何远程特定依赖"。这是 Spring remoting 一以贯之的设计，0.9.1 已完整确立。

## 子包结构

```
remoting/
├── RemoteAccessException.java      统一远程异常（extends NestedRuntimeException）
├── support/
│   ├── RemoteProxyFactoryBean.java           所有客户端代理抽象基类
│   └── AuthorizableRemoteProxyFactoryBean.java  加 username/password（Caucho HTTP 用）
├── rmi/
│   ├── RemoteInvocationHandler（接口，extends Remote）
│   ├── RemoteInvocationWrapper（服务端，extends UnicastRemoteObject）
│   ├── StubInvocationHandler（客户端 JDK 动态代理处理器）
│   ├── RmiProxyFactoryBean（public）
│   └── RmiServiceExporter（public）
└── caucho/
    ├── HessianProxyFactoryBean / HessianServiceExporter
    └── BurlapProxyFactoryBean / BurlapServiceExporter
```

## 核心接口与实现类

### `RemoteAccessException` — `remoting/RemoteAccessException.java`
`extends NestedRuntimeException`（非受检）。统一所有协议的远程异常，使客户端"协议无关地"捕获或忽略：
```java
public RemoteAccessException(String msg, Throwable ex);
```

### `support/RemoteProxyFactoryBean` — 所有客户端代理的抽象基类
`implements FactoryBean, InitializingBean`：
```java
public void setServiceInterface(Class serviceInterface);   // 必须 interface
public void setServiceUrl(String serviceUrl);
public void afterPropertiesSet() throws MalformedURLException, RemoteAccessException;
protected abstract Object createProxy() throws MalformedURLException, RemoteAccessException;
public Object getObject();
public boolean isSingleton();   // true
```
`afterPropertiesSet` 调 `createProxy()`，并校验代理实例实现了 `serviceInterface`。

### RMI 子包三件套

`RemoteInvocationHandler`（接口，`extends Remote`）：
```java
public Object invokeRemote(String methodName, Class[] paramTypes, Object[] params) throws Exception;
```

`RemoteInvocationWrapper`（`extends UnicastRemoteObject implements RemoteInvocationHandler`，服务端）—— 用反射调用被包装对象：
```java
public Object invokeRemote(String methodName, Class[] paramTypes, Object[] params) throws Exception {
    Method method = wrappedObject.getClass().getMethod(methodName, paramTypes);
    return method.invoke(wrappedObject, params);
}
```

`StubInvocationHandler`（`implements InvocationHandler, Serializable`，客户端 JDK 动态代理处理器）：
```java
public Object invoke(Object proxy, Method method, Object[] params) throws Exception {
    if (method.getDeclaringClass().equals(Object.class)) {
        return method.invoke(this, params);    // Object 方法本地处理
    }
    return this.stub.invokeRemote(method.getName(), method.getParameterTypes(), params);
}
```

`RmiProxyFactoryBean`（`extends RemoteProxyFactoryBean`）— `createProxy()`：`java.rmi.Naming.lookup` → 包装为 `StubInvocationHandler` → `Proxy.newProxyInstance` 生成业务接口代理 → **再套一层 AOP `ProxyFactory`**，把 `UndeclaredThrowableException` 转成 `RemoteAccessException`。

`RmiServiceExporter`（`implements InitializingBean`）— `afterPropertiesSet()`：先尝试 `LocateRegistry.getRegistry().list()`，失败则 `createRegistry`，最后 `rebind(name, new RemoteInvocationWrapper(service))`。默认端口 `Registry.REGISTRY_PORT`（1099）。

### Caucho 子包（Hessian/Burlap，完全对称）

`HessianProxyFactoryBean`（`extends AuthorizableRemoteProxyFactoryBean`）— `createProxy()`：用 Caucho `HessianProxyFactory.create(interface, url)` 生成代理 → AOP `ProxyFactory` 包一层，把 `HessianRuntimeException`/`UndeclaredThrowableException` 转成 `RemoteAccessException`。

`HessianServiceExporter`（`implements Controller`，Web MVC 控制器）：
```java
public void setService(Object service);   // new HessianSkeleton(service)
public ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException;
```
`handleRequest` 用 `HessianInput`/`HessianOutput` 包装请求/响应流，调 `skeleton.invoke(in, out)`。

`BurlapProxyFactoryBean` / `BurlapServiceExporter` — 与 Hessian 完全对称，仅协议不同（Burlap 是 XML，Hessian 是二进制）。

## 设计亮点

1. **两层代理**（RMI 尤其明显）：第一层 JDK `Proxy.newProxyInstance` + `StubInvocationHandler` 负责把方法调用序列化到网络上；第二层 AOP `ProxyFactory` 负责把协议特定异常翻译成统一 `RemoteAccessException`。两层职责分离——面试可展开。
2. **`StubInvocationHandler.invoke` 对 Object 方法特判**（:33）：`equals`/`hashCode`/`toString` 本地执行，不发起网络往返——远程代理经典细节。
3. **`RmiServiceExporter` 自动建注册表**：`getRegistry().list()` 探测，失败则 `createRegistry`，免去运维预启 rmiregistry。
4. **Hessian/Burlap 服务端即 Controller**：复用 Spring Web MVC 的 `Controller` 契约，一个 HTTP servlet 端点即一个 RPC 服务，零额外基础设施。

## 关键设计模式

| 模式 | 类 |
|------|----|
| 代理（JDK 动态代理） | `StubInvocationHandler` + `Proxy.newProxyInstance` |
| 代理（AOP） | `RmiProxyFactoryBean`/`HessianProxyFactoryBean` 套 `ProxyFactory` 做异常翻译 |
| 工厂 | `RemoteProxyFactoryBean` 抽象基类 + 各 `createProxy` |
| 适配器 | `RemoteInvocationWrapper` 把"任意业务对象"适配成 RMI 可调用的 `Remote` 对象 |
| 模板方法 | `RemoteProxyFactoryBean.afterPropertiesSet` + 抽象 `createProxy` |

## 对应测试

`test/com/interface21/remoting/RemotingTestSuite.java`（1 个）—— 以**负向测试**为主：构造各代理 FactoryBean 指向不存在的主机/URL，断言抛 `RemoteAccessException`；测试 `serviceInterface` 误传成类（非接口）抛 `IllegalArgumentException`，及 `createProxy` 返回类型不匹配的校验。

## 建议学习顺序与精读片段

1. `RemoteAccessException.java`（读 Javadoc，理解"透明"哲学）
2. `RemoteProxyFactoryBean.afterPropertiesSet`（模板与校验）
3. `rmi/RemoteInvocationHandler` + `RemoteInvocationWrapper`（:27）+ `StubInvocationHandler`（:32）三件套——看清一次远程调用的完整路径
4. `RmiProxyFactoryBean.createProxy`（:41）——注意双层代理与异常翻译
5. `RmiServiceExporter.afterPropertiesSet`（:77）——注册表探测/创建
6. `HessianProxyFactoryBean`（:34）+ `HessianServiceExporter`（:46）——与 RMI 对照，体会协议无关性

| 精读片段 | 为什么 |
|----------|--------|
| `rmi/StubInvocationHandler.java:invoke`（:32） | Object 方法本地特判 + 远程调用序列化 |
| `rmi/RmiProxyFactoryBean.java:createProxy`（:41） | 双层代理与异常翻译 |
| `rmi/RmiServiceExporter.java:afterPropertiesSet`（:77） | 注册表自动探测/创建 |
| `caucho/HessianServiceExporter.java:handleRequest`（:46） | 服务端即 Web Controller |

---

← 上一节：[13-jndi.md](13-jndi.md) ｜ 下一节：[15-ejb.md](15-ejb.md)
