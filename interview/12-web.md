# 12 · web 模块学习要点

> 基于 `src/com/interface21/web/**`（96 个 `.java`）与 `test/com/interface21/web/**`（37 个）逐字核对编写。
> Java 1.3、Servlet 2.3、无泛型。web 是文件最多的模块，但主线非常清晰。

## 模块定位

Spring MVC 是构建在 IoC 容器之上的 Web 表现层框架，本身不实现 AOP，但深度依赖 IoC。
- **与 IoC 的关系**：`DispatcherServlet` 不自己 new 出 HandlerMapping / HandlerAdapter / ViewResolver 等，而是在 `initFrameworkServlet()` 中从 `WebApplicationContext` 按 bean 名/类型查找，找不到回退框架默认实现——**MVC 的全部协作对象都是容器管理的 bean**。
- **与 AOP 的关系**：web 不直接依赖 AOP；但 DispatcherServlet 内联注释明确"Handler 是个代理对象，可能会执行 AOP"（`doService` 第 380 行），框架预期 handler bean 可被 AOP 代理包装。
- **分层**：`HttpServletBean`（init-param 当 JavaBean 属性）→ `FrameworkServlet`（创建/管理 WebApplicationContext、发布 `RequestHandledEvent`）→ `DispatcherServlet`（前端控制器）。
- 源自 Rod Johnson《Expert One-On-One J2EE Design and Development》第 12 章。

## 子包结构

| 包 | 职责 |
|----|------|
| `web.servlet` | 前端控制器核心：`DispatcherServlet`、`FrameworkServlet`、`HttpServletBean`；核心契约 `HandlerMapping`/`HandlerAdapter`/`HandlerInterceptor`/`HandlerExecutionChain`/`View`/`ViewResolver`/`LocaleResolver`/`ThemeResolver`/`ModelAndView`/`LastModified` |
| `web.servlet.mvc` | Controller 体系 |
| `web.servlet.mvc.multiaction` | 多动作控制器：`MultiActionController` + `MethodNameResolver` 体系 |
| `web.servlet.handler` | `HandlerMapping` 实现（`BeanNameUrlHandlerMapping` 默认、`SimpleUrlHandlerMapping` 等） |
| `web.servlet.view` | `View`/`ViewResolver` 实现（`InternalResourceViewResolver` 默认等） |
| `web.servlet.view.velocity` / `xslt` / `document` | Velocity / XSLT / Excel/PDF 视图 |
| `web.servlet.i18n` | `LocaleResolver` 实现（`AcceptHeaderLocaleResolver` 默认等） |
| `web.servlet.theme` | `ThemeResolver` 实现（`FixedThemeResolver` 默认等） |
| `web.servlet.support` | 请求级支持：`RequestContext`、`RequestContextUtils` |
| `web.servlet.tags` | JSP 标签库（`spring.tld`） |
| `web.bind` | 数据绑定 |
| `web.context` / `web.context.support` | Web 层 ApplicationContext |
| `web.util` | `WebUtils`、`HtmlUtils`、`ExpressionEvaluationUtils` 等 |

## MVC 请求处理主线（最核心，必背）

完整链路集中在 `DispatcherServlet.doService(...)`（`web/servlet/DispatcherServlet.java` 第 335–404 行，源码已带中文 1~8 注释）。注意 doGet/doPost 由父类 `FrameworkServlet` 实现（final），统一委托 `serviceWrapper()` → `doService()`：

```
HTTP 请求
  │
  ▼
FrameworkServlet.doGet/doPost [final]  ── serviceWrapper(request, response) ── doService()  [抽象，DispatcherServlet 实现]
  │
  │【1】设置 request 属性（:341-347）
  │     WEB_APPLICATION_CONTEXT_ATTRIBUTE / LOCALE_RESOLVER_ATTRIBUTE / THEME_RESOLVER_ATTRIBUTE
  │
  │【2】查 handler 执行链：HandlerExecutionChain mappedHandler = getHandler(request)
  │     遍历所有 handlerMappings，第一个 hm.getHandler(request) 返回非 null 即用 → HandlerExecutionChain(handler, interceptors[])
  │     找不到 → response.sendError(404); return
  │
  │【3】查 handler 适配器：HandlerAdapter ha = getHandlerAdapter(handler)
  │     遍历 handlerAdapters，第一个 ha.supports(handler) 为 true 即用
  │     找不到 → throw new ServletException("No adapter for handler " + handler)
  │
  │【HTTP 缓存】wasRevalidated(...)  仅 GET：读 If-Modified-Since，调 ha.getLastModified(...)，未修改→304; return
  │
  │【4】执行拦截器 preHandle（正序）：任一返回 false 即中止
  │     for (i=0; i<interceptors.length; i++) if (!interceptor.preHandle(...)) return;
  │
  │【5】真正调用 handler：ModelAndView mv = ha.handle(request, response, handler)
  │     SimpleControllerHandlerAdapter.handle 转调 controller.handleRequest(...)
  │
  │【6】执行拦截器 postHandle（逆序）
  │     for (i=interceptors.length-1; i>=0; i--) interceptor.postHandle(...)
  │
  │【7/8】渲染视图（若 mv != null）：
  │     Locale locale = localeResolver.resolveLocale(request); response.setLocale(locale);
  │     render(mv, request, response, locale):
  │       mv.isReference() ? view = viewResolver.resolveViewName(viewName, locale) : view = mv.getView()
  │       view.render(model, request, response)
  │         → AbstractView.render: 合并 staticAttributes + 动态 model → renderMergedOutputModel(...) [模板方法]
  │            InternalResourceView: exposeModelsAsRequestAttributes → RequestDispatcher.forward
  │            RedirectView:         model 转 query string → response.sendRedirect
  │            JstlView:             额外暴露 JSTL LocalizationContext
  │
  ▼
finally：webApplicationContext.publishEvent(new RequestHandledEvent(...))   // 无论成败都发
```

### 初始化阶段（servlet 启动一次性完成）
- `HttpServletBean.init()`（final）：用 `BeanWrapper` 把 `<init-param>` 当 bean 属性绑到 servlet → `initServletBean()`。
- `FrameworkServlet.initServletBean()`（final）：`createWebApplicationContext()`（默认 `new XmlWebApplicationContext(parent, namespace)`，父 context 来自 `WebApplicationContextUtils.getWebApplicationContext(sc)`）→ `initFrameworkServlet()`。
- `DispatcherServlet.initFrameworkServlet()`：依次 `initLocaleResolver()` / `initThemeResolver()` / `initHandlerMappings()` / `initHandlerAdapters()` / `initViewResolver()`。每个用"按名/类型找 bean，找不到回退默认"：
  - 默认 LocaleResolver = `AcceptHeaderLocaleResolver`
  - 默认 ThemeResolver = `FixedThemeResolver`
  - 默认 HandlerMapping = `BeanNameUrlHandlerMapping`
  - 默认 HandlerAdapter = `SimpleControllerHandlerAdapter`
  - 默认 ViewResolver = `InternalResourceViewResolver`
  - handlerMappings / handlerAdapters 用 `OrderComparator` 排序

**面试常考点**：
- `doService` 不是 `doGet/doPost`；后者在父类 final，统一进 `serviceWrapper`，保证发 `RequestHandledEvent`。
- "找不到 handler"返回 404；"找不到 adapter"抛 `ServletException`——语义不同。
- 拦截器 preHandle 任一返回 false 即中止（含 handler）；postHandle 总执行（handler 正常返回后），且逆序。
- `ModelAndView` 可持 `View` 实例或 viewName 字符串；`isReference()` 为 true 走 ViewResolver。

## 核心接口（路径 + 逐字签名）

### `Controller` — `web/servlet/mvc/Controller.java`
```java
ModelAndView handleRequest(HttpServletRequest request, HttpServletResponse response) throws ServletException, IOException;
```
MVC 的"命令"接口（类似 Struts Action）；返回 null 表示已自行处理响应。

### `HandlerMapping`（extends `ApplicationContextAware`）— `web/servlet/HandlerMapping.java`
```java
HandlerExecutionChain getHandler(HttpServletRequest request) throws ServletException;
```

### `HandlerAdapter`（extends `ApplicationContextAware`）— `web/servlet/HandlerAdapter.java`
```java
boolean supports(Object handler);
ModelAndView handle(HttpServletRequest request, HttpServletResponse response, Object handler) throws ServletException, IOException;
long getLastModified(HttpServletRequest request, Object handler);
```
让 DispatcherServlet **不绑定任何具体 handler 类型**（handler 是 `Object`），可接入任意第三方处理器。

### `HandlerInterceptor` — `web/servlet/HandlerInterceptor.java`
```java
boolean preHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws ServletException, IOException;
void postHandle(HttpServletRequest request, HttpServletResponse response, Object handler) throws ServletException, IOException;
```
preHandle 返回 false 中止链。Javadoc 对比 Servlet Filter：Filter 更强（可换 request/response），但 Interceptor 配在 ApplicationContext 内、聚焦 handler 预处理。

### `View` / `ViewResolver`
```java
// web/servlet/View.java
void addStaticAttribute(String name, Object o);
void render(Map model, HttpServletRequest request, HttpServletResponse response) throws IOException, ServletException;
void setName(String name);  String getName();
// web/servlet/ViewResolver.java（extends ApplicationContextAware）
View resolveViewName(String viewName, Locale locale) throws ServletException;
```

### `LocaleResolver` / `ThemeResolver`
```java
// web/servlet/LocaleResolver.java
Locale resolveLocale(HttpServletRequest request);
void setLocale(HttpServletRequest request, HttpServletResponse response, Locale locale);
// web/servlet/ThemeResolver.java
String resolveThemeName(HttpServletRequest request);
void setThemeName(HttpServletRequest request, HttpServletResponse response, String themeName);
```

### `LastModified` — `web/servlet/LastModified.java`
```java
long getLastModified(HttpServletRequest request);
```

## Controller 继承体系

```
ApplicationObjectSupport (context.support：setApplicationContext 生命周期)
      ▼
WebContentGenerator (preventCaching / cacheForSeconds；useExpiresHeader)
  │
  ├──► AbstractController ──implements──► Controller (handleRequest)
  │        │   final handleRequest → 模板方法 handleRequestInternal
  │        ├──► ParameterizableViewController  (handleRequestInternal → new ModelAndView(viewName))
  │        └──► BaseCommandController (commandClass / validator / bindAndValidate)
  │                ├──► AbstractCommandController (handleRequestInternal → userObject → bindAndValidate → handle)
  │                └──► AbstractFormController (isFormSubmission? 提交 : showNewForm)
  │                        ├──► SimpleFormController (formView/successView; 有错→formView, 无错→onSubmit→successView)
  │                        └──► AbstractWizardFormController (sessionForm=true; pages[]; _finish/_cancel/_targetN)
  │
  └──► MultiActionController ──implements──► Controller, LastModified
          (方法名由 MethodNameResolver 解析；反射缓存 handler/lastModified/exception 方法)
```

### 关键实现类要点
| 类 | 要点 |
|----|------|
| `AbstractController` | `handleRequest`（final）校验 supportedMethods/requireSession/cacheSeconds 后调抽象 `handleRequestInternal` |
| `BaseCommandController` | `bindAndValidate`（final）：createBinder → bind → (validateOnBinding)`ValidationUtils.invokeValidator` → `onBindAndValidate` 回调。属性 `beanName="command"`、`commandClass`、`validator` |
| `AbstractFormController` | `handleRequestInternal`：POST 当提交、GET 当新表单；`formBackingObject`/`referenceData`/`showForm`/`processSubmit`；`sessionForm`(默认false)/`bindOnNewForm`；session 表单模式取不到对象→`handleInvalidSubmit` |
| `SimpleFormController` | `processSubmit`：有错→`showForm`，无错→`onSubmit` 链（全参→单参）→`successView`。属性 `formView`/`successView` |
| `AbstractWizardFormController` | 构造强制 `setSessionForm(true)`+`setValidateOnBinding(false)`；请求参数 `_finish`/`_cancel`/`_targetN`；`validatePage`/`processFinish`/`processCancel`；`allowDirtyBack`(默认true)/`allowDirtyForward`(默认false) |
| `MultiActionController` | 单类多方法，方法签名 `ModelAndView xxx(HttpServletRequest, HttpServletResponse[, HttpSession/command])`；`setDelegate` 时反射缓存 handler 方法 |
| `ParameterizableViewController` | 最简：`handleRequestInternal` 直接 `return new ModelAndView(successView)` |

## 视图层实现

| 类 | 要点 |
|----|------|
| `AbstractCachingViewResolver` | 模板方法 + 缓存。`resolveViewName`（final）：按 `viewname + "_" + locale` 查 HashMap，未命中调子类 `loadView` 并配置。cache 默认 true，Javadoc 警告关闭性能下降 ≥20% |
| `InternalResourceViewResolver`（默认） | `loadView`：`new InternalResourceView()`，`url = prefix + viewName + suffix`。**不支持按 locale 解析不同资源** |
| `InternalResourceView` | `renderMergedOutputModel` → `exposeModelsAsRequestAttributes`（model 逐个 `request.setAttribute`）→ `RequestDispatcher.forward` |
| `JstlView` | extends 上者，额外把 Spring locale/messageSource 包成 JSTL `LocalizationContext` 暴露 |
| `RedirectView` | model 序列化成 query string（`URLEncoder.encode`）→ `response.sendRedirect` |
| `AbstractHandlerMapping`（extends `ApplicationObjectSupport implements HandlerMapping, Ordered`） | `getHandler`（final）= `getHandlerInternal` 回退 `defaultHandler` → 组装 `HandlerExecutionChain(handler, interceptors[])` |
| `AbstractUrlHandlerMapping` | URL→handler 的 Map 注册与查找；`lookupHandler` 先精确匹配再 Ant（`PathMatcher.match`） |
| `BeanNameUrlHandlerMapping`（默认） | 遍历所有 bean 名，以 `/` 开头即视为 URL 映射，按空格拆分多 URL |
| `SimpleControllerHandlerAdapter` | `supports`：`Controller.class.isAssignableFrom`；`handle` 转调 `controller.handleRequest`——**把 Controller 接口适配到通用 DispatcherServlet 的桥梁** |

## 数据绑定（web.bind 如何把请求参数绑到对象）

绑定类层级（跨 web 与 validation 包）：
```
BindException (validation: extends Exception implements Errors, 持 target + 错误列表)
   ▲ extends
DataBinder (validation)
   ▲ extends
ServletRequestDataBinder (web/bind)
```

调用链（以 `BaseCommandController.bindAndValidate` 为例）：
1. `createBinder(request, command)` → `new ServletRequestDataBinder(command, getBeanName())` → 回调 `initBinder`（子类可注册自定义 `PropertyEditor`）
2. `binder.bind(request)` → 内部 `new ServletRequestParameterPropertyValues(request)` → 委托父类 `DataBinder.bind(PropertyValues)`
3. `ServletRequestParameterPropertyValues`：用 `WebUtils.getParametersStartingWith(request, prefix)` 把请求参数包成 `MutablePropertyValues`（支持前缀+分隔符，默认 `_`）
4. `DataBinder.bind(pvs)`（`validation/DataBinder.java:114`）：先按 `requiredFields` 检查缺失（码 `"required"`）；调 `getBeanWrapper().setPropertyValues(pvs, true, null)`；捕获 `PropertyVetoExceptionsException` 转成码 `"typeMismatch"` 的 `FieldError`
5. `close()`：有错则 `throw this`（即 `BindException`）
6. 开启 validateOnBinding 时 `ValidationUtils.invokeValidator(getValidator(), command, binder)`
7. 最后回调 `onBindAndValidate`

绑定结果（`BindException`/`Errors`）通过 `errors.getModel()` 进入 `ModelAndView`，被 `InternalResourceView.exposeModelsAsRequestAttributes` 暴露为 request attribute，供 JSP `<spring:bind>`（`tags/BindTag`）渲染。`BindUtils`（`web/bind`）是上述流程的静态便捷封装。

> 详见 [10-validation.md](10-validation.md)。

## 关键设计模式

| 模式 | 类 |
|------|----|
| 前端控制器 | `DispatcherServlet` |
| 模板方法 | `AbstractController.handleRequest`→`handleRequestInternal`；`AbstractView.render`→`renderMergedOutputModel`；`AbstractCachingViewResolver.resolveViewName`→`loadView`；`AbstractHandlerMapping.getHandler`→`getHandlerInternal`；`BaseCommandController.bindAndValidate` + `initBinder`/`onBindAndValidate` 钩子；`SimpleFormController.processSubmit`→`onSubmit` 钩子链 |
| 策略 | `HandlerMapping`、`HandlerAdapter`、`ViewResolver`、`LocaleResolver`、`ThemeResolver`、`MethodNameResolver`（Javadoc 自述 "Strategy GoF Design pattern"） |
| 命令 | `Controller.handleRequest`；命令对象 = `commandClass` 实例 |
| 适配器 | `HandlerAdapter`/`SimpleControllerHandlerAdapter`；`JstlView` 适配 JSTL |
| 责任链 | `HandlerExecutionChain` + `HandlerInterceptor`（preHandle 正序、postHandle 逆序，preHandle 可中断） |
| 工厂 | `ViewResolver` 创建 `View`；`InternalResourceViewResolver.loadView` 反射 new |
| 装饰器 | `EscapedErrors` 包装 `Errors` 加 HTML 转义 |
| 组合/分层 | `NestingThemeSource.setParent`；根 context + servlet 子 context |
| 单例+缓存 | `AbstractCachingViewResolver`、`ResourceBundleThemeSource` |

## 对应测试（`test/com/interface21/web/**`）

测试用自带 mock 基础设施（`test/.../web/mock/`：`MockHttpServletRequest/Response/Session/ServletContext/ServletConfig/RequestDispatcher`）：

- `servlet/DispatcherServletTestSuite`：端到端验证（构造 simple/complex 两个 servlet，测 namespace、ServletContext 属性名、404、GET 表单 forward）
- `servlet/handler/`：`BeanNameUrlHandlerMappingTestSuite`、`SimpleUrlHandlerMappingTestSuite`、`PathMatchingUrlHandlerMappingTestSuite`
- `servlet/mvc/`：`CommandControllerTestSuite`、`FormControllerTestSuite`、`WizardFormControllerTestSuite`、`MultiActionControllerTestSuite`、`ParameterizableViewControllerTestSuite`
- `servlet/view/`：`InternalResourceViewTests`、`RedirectViewTests`、`ViewResolverTestSuite`、`ResourceBundleViewResolverTestSuite`
- `servlet/LocaleResolverTestSuite`、`ThemeResolverTestSuite`、`tags/TagTestSuite`
- `bind/`：`BindUtilsTestSuite`、`EscapedErrorsTestSuite`、`RequestUtilsTestSuite`、`ServletRequestParameterPropertyValuesTestSuite`
- `context/`：`ContextLoaderTestSuite`、`WebApplicationContextTestSuite`

## 建议学习顺序

1. **入口与装配**：`HttpServletBean.init` → `FrameworkServlet.initServletBean`/`serviceWrapper` → `DispatcherServlet.initFrameworkServlet`（"一切皆 bean、无则回退默认"）
2. **请求主线**：`DispatcherServlet.doService`（getHandler → getHandlerAdapter → preHandle → ha.handle → postHandle → render）
3. **Controller 体系**：`Controller` → `AbstractController` → `BaseCommandController` → `AbstractFormController` → `SimpleFormController`，再旁支 `AbstractWizardFormController`、`MultiActionController`
4. **视图层**：`View`/`ViewResolver` → `AbstractCachingViewResolver` → `InternalResourceViewResolver` → `AbstractView.render` → `InternalResourceView.renderMergedOutputModel`（forward）→ `RedirectView`/`JstlView`
5. **数据绑定**：`ServletRequestDataBinder` → `ServletRequestParameterPropertyValues` → `DataBinder.bind` → `BindException`/`Errors`；再读 `BaseCommandController.bindAndValidate` 串联
6. **拦截器与 i18n/theme**：`HandlerInterceptor` → `LocaleResolver`/`ThemeResolver` 各实现 → `RequestContext`/`RequestContextUtils` → `Theme`/`ThemeSource`
7. **Web 上下文**：`WebApplicationContext` → `ContextLoader`/`ContextLoaderListener` → `XmlWebApplicationContext` → `WebApplicationContextUtils`
8. **JSP 标签**：`RequestContextAwareTag` → `MessageTag`/`ThemeTag`/`BindTag`/`BindErrorsTag`

## 值得精读的源码片段

| 片段 | 为什么 |
|------|--------|
| `DispatcherServlet.java:doService` | 请求处理主线全貌（源码已带中文 1~8 注释） |
| `DispatcherServlet.java:initFrameworkServlet` + `initHandlerMappings`/`initHandlerAdapters`/`initViewResolver` | 默认策略与回退 |
| `DispatcherServlet.java:render` | 视图名引用 vs View 实例两条分支 |
| `FrameworkServlet.java:initServletBean`/`createWebApplicationContext`/`serviceWrapper` | 父子 context、事件发布 |
| `HttpServletBean.java:init` | BeanWrapper 把 init-param 绑属性（IoC 在 servlet 上的雏形） |
| `mvc/AbstractController.java:handleRequest` | 模板方法：方法校验→session 校验→缓存头→handleRequestInternal |
| `mvc/BaseCommandController.java:bindAndValidate` | 绑定/验证的编排与扩展点 |
| `mvc/AbstractFormController.java:handleRequestInternal` | GET=新表单/POST=提交分派 |
| `mvc/SimpleFormController.java:processSubmit` / `onSubmit` 链 | 错误回 formView，成功走 successView |
| `mvc/AbstractWizardFormController.java:processSubmit` | finish/cancel/target 分支 + allowDirtyBack/Forward |
| `view/AbstractView.java:render` | 合并静态/动态属性 + `renderMergedOutputModel` 模板方法 |
| `view/InternalResourceView.java:renderMergedOutputModel`/`exposeModelsAsRequestAttributes` | model→request attribute→forward 完整逻辑 |
| `view/AbstractCachingViewResolver.java:resolveViewName` | 缓存键 `viewname_locale`、View 生命周期配置 |
| `handler/AbstractUrlHandlerMapping.java:lookupHandler` | 精确匹配 + Ant 模式 `PathMatcher.match` |
| `handler/BeanNameUrlHandlerMapping.java:initApplicationContext` | bean 名以 `/` 开头即 URL 映射 |
| `mvc/SimpleControllerHandlerAdapter.java` | 适配器模式最小完整范例 |
| `tags/BindTag.java:doStartTagInternal` | path 拆解 → 取 Errors/字段值/错误码 → 暴露 `BindStatus` |

## 几处易被忽略但值得注意的事实

- 仓库 `DispatcherServlet.doService` 源码已带中文注释（"1. 设置属性" … "8. 渲染 View 返回"），本仓库已被加工为学习版。
- `WebApplicationContext extends ApplicationContext, ThemeSource`：MVC 的 context 天生是 `ThemeSource`，主题能力由 [ui 模块](11-ui.md) 定义。`XmlWebApplicationContext extends AbstractXmlUiApplicationContext`（`ui.context.support`），后者在 `onRefresh()` 用 `UiApplicationContextUtils.initThemeSource` 装配 `ThemeSource`（默认 `ResourceBundleThemeSource`）——**这是 web 与 ui 两个模块的真正耦合点**。
- `FrameworkServlet.doGet`/`doPost` 是 **final** 的，子类无法重写；扩展点只有 `doService` 与 `initFrameworkServlet`。

---

← 上一节：[11-ui.md](11-ui.md) ｜ 下一节：[13-jndi.md](13-jndi.md)
