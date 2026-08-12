# 10 · validation 模块学习要点

> 基于 `src/com/interface21/validation/**`（7 个 `.java`）与 `test/com/interface21/validation/**`（1 个）逐字核对编写。
> 把"数据绑定 → 类型转换错误 → 业务校验错误 → 错误消息国际化"串成一条链，是 [web](12-web.md) 表单绑定的底层基础。

## 模块定位

validation 本身不依赖 Web 容器，可独立用于任何 POJO 校验。它与 web 的 `web.bind`（`ServletRequestDataBinder extends DataBinder`）共同完成请求参数到对象绑定与校验。

## 核心接口与实现类

### `Validator` — `validation/Validator.java`（校验器顶层接口）
```java
boolean supports(Class clazz);
void validate(Object obj, Errors errors);
```
把校验逻辑从领域对象中解耦。

### `Errors` — `validation/Errors.java`（错误容器接口）
> ⚠️ 0.9.1 还**没有 `BindingResult`**，`Errors` 就是唯一抽象。支持 `setNestedPath` 以校验子树。
```java
String getObjectName();
void reject(String errorCode, String defaultMessage);
void reject(String errorCode, Object[] errorArgs, String defaultMessage);
void rejectValue(String field, String errorCode, String defaultMessage);
void rejectValue(String field, String errorCode, Object[] errorArgs, String defaultMessage);
boolean hasErrors();  int getErrorCount();  List getAllErrors();
boolean hasGlobalErrors();  int getGlobalErrorCount();  List getGlobalErrors();  ObjectError getGlobalError();
boolean hasFieldErrors(String field);  int getFieldErrorCount(String field);
List getFieldErrors(String field);  FieldError getFieldError(String field);
Object getFieldValue(String field);
void setNestedPath(String nestedPath);
```

### `BindException` — `validation/BindException.java`（`Errors` 默认实现）
**自身就是一个 `Exception`**（`extends Exception implements Errors`），内部持有 `BeanWrapper`。Javadoc 原文："Slightly unusual, as it _is_ an exception."
```java
public BindException(Object target, String name);
protected BeanWrapper getBeanWrapper();
protected void addFieldError(FieldError fe);
public Object getTarget();
public final Map getModel();          // 导出给 Web MVC 的 model
public static final String ERROR_KEY_PREFIX = BindException.class.getName() + ".";
```
`getModel()` 把 `ERROR_KEY_PREFIX + objectName → this(Errors)` 与 `objectName → 目标对象` 放入 Map——这是它和 Web 层对接的契约。

### `DataBinder` — `validation/DataBinder.java`（**继承 `BindException`**）
在错误容器之上增加绑定能力（继承而非组合）：
```java
public static final String MISSING_FIELD_ERROR_CODE = "required";
public DataBinder(Object target, String name);
public void setRequiredFields(String[] requiredFields);
public void registerCustomEditor(Class requiredType, String field, PropertyEditor propertyEditor);
public void registerCustomEditor(Class requiredType, PropertyEditor propertyEditor);
public Object getFieldValue(String field);
public void bind(PropertyValues pvs);        // 核心：绑定 + 产生 required/typeMismatch 错误
public Map close() throws BindException;     // 有错则 throw this
```
`bind()` 做两件事：检查必填字段缺失（码 `"required"`）；调 `getBeanWrapper().setPropertyValues(pvs, true, null)`，捕获 `PropertyVetoExceptionsException` 转成码 `"typeMismatch"` 的 `FieldError`。

### `ValidationUtils` — `validation/ValidationUtils.java`（abstract，仅一个静态方法）
```java
public static void invokeValidator(Validator validator, Object object, Errors errors);
```
被 `web.bind.BindUtils#bindAndValidate` 使用。内部先调 `supports` 校验，不通过抛 `IllegalArgumentException`。

### `ObjectError` / `FieldError` — 错误模型
继承链：`FieldError → ObjectError → MessageSourceResolvableImpl`（后者位于 `context/support`，实现 `MessageSourceResolvable`）。
```java
// ObjectError
public ObjectError(String objectName, String code, Object[] args, String defaultMessage);
// FieldError（单 code 构造器自动扩展为 3 个消息码）
public FieldError(String objectName, String field, Object rejectedValue, String code, Object[] args, String defaultMessage);
public String getField();
public Object getRejectedValue();
public static final String CODE_SEPARATOR = ".";
```

## 设计亮点（面试核心）

1. **错误码即消息 key + 3 级消息码**：`FieldError` 单码构造器自动生成 3 级消息码，顺序为 `code.objectName.field` → `code.field` → `code`（见 `FieldError.java:60-62`）。例如 `typeMismatch`、`age`、`user` 会尝试 `typeMismatch.user.age`、`typeMismatch.age`、`typeMismatch`。`FieldError` 本身就是 `MessageSourceResolvable`，可直接传给 `MessageSource.getMessage(MessageSourceResolvable, Locale)` 解析——**这是面试高频考点"Spring 校验错误如何国际化"的 0.9.1 版答案**。

2. **`Errors` 既当异常又当容器**：`BindException extends Exception implements Errors`，`DataBinder.close()` 直接 `throw this`。这种"异常即状态"的设计为简化 Web 层 catch（后续版本才拆出 `BindingResult` 接口）。

3. **嵌套路径支持子对象复用校验器**：`Errors.setNestedPath("spouse")` 后，所有 `rejectValue` 自动加前缀，使同一个 `Validator` 可校验任意子树（测试 `ValidationTestSuite.testValidatorWithErrors` 校验 `spouse.age`）。

4. **DataBinder 通过继承复用错误容器**：直接 `extends BindException`。`bind()` 的错误码 `required`/`typeMismatch` 由框架硬编码，业务 `Validator` 只需关心理校验。

## 关键设计模式

| 模式 | 类 |
|------|----|
| 策略 | `Validator`（校验逻辑可替换） |
| 异常即状态 | `BindException extends Exception implements Errors` |
| 值对象 | `ObjectError` / `FieldError`（携带多 code + 参数 + 默认消息） |
| 桥接（消息解析） | `FieldError is-a MessageSourceResolvable`，把校验错误与 i18n 桥接 |

## 对应测试

`test/com/interface21/validation/ValidationTestSuite.java`（单文件）：绑定成功、绑定失败（`typeMismatch`）、自定义 `PropertyEditor`（单字段/全 String）、`Validator` 嵌套校验（`TestBeanValidator` + `SpouseValidator`，断言 6 个错误、3 级消息码 `NOT_ROD.tb.name` / `NOT_ROD.name` / `NOT_ROD`）。

## 建议学习顺序与精读片段

1. `Validator.java`（2 个方法，理解契约）
2. `Errors.java`（看接口全貌，注意 `setNestedPath` 的 Javadoc 例子）
3. `BindException.java`（`extends Exception implements Errors`）、`getModel()`
4. `DataBinder.java:114`（`bind()` 方法，看 `required`/`typeMismatch` 如何产生）
5. `FieldError.java:57-64`（**3 级消息码生成算法——面试核心**）
6. `MessageSourceResolvableImpl.java`（确认 `getCode()` 返回数组最后一个，即原始 code）

| 精读片段 | 为什么 |
|----------|--------|
| `validation/FieldError.java:57-64` | 3 级消息码生成算法——面试"校验错误如何国际化"的核心 |
| `validation/DataBinder.java:114`（`bind`） | `required`/`typeMismatch` 错误码如何产生 |
| `validation/BindException.java`（`extends Exception implements Errors`） | "异常即状态"设计 + `getModel()` 与 web 层对接契约 |

---

**一句话总结**：validation 用 `Validator`（校验）+ `Errors`/`BindException`（错误容器，且自身是异常）+ `DataBinder`（绑定）+ `FieldError`（3 级消息码 + 是 `MessageSourceResolvable`）把"绑定/校验错误"与"i18n 消息"桥接起来。0.9.1 还**没有 `BindingResult`**，`Errors` 是唯一抽象。

← 上一节：[09-orm.md](09-orm.md) ｜ 下一节：[11-ui.md](11-ui.md)
