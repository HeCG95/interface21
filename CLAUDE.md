# CLAUDE.md

本文件用于指导 Claude（及任何 AI 助手）在本仓库中高效工作。

## 项目概述

本仓库是 **Spring Framework 早期版本（0.9.1，2003 年 8 月）** 的源码，即 Rod Johnson 所著
《Expert One-on-One J2EE Design and Development》(Wrox, 2002) 一书附带的框架版本。

- **根包名**：`com.interface21`（这是 1.0 里程碑版本之前的命名；1.0 之后才改为 `org.springframework`）。
- **定位**：这是一个**学习/研读型仓库**。在原始 Spring 英文源码基础上，仓库维护者添加了中文 Javadoc
  翻译、中文注释以及 UML 模型图（`docs/model/*.svg`），用于研读 Spring 早期设计。
- **Git 历史**：混合了两类提交——原始 Spring 的英文提交（如 "let RuntimeExceptions through"）与
  维护者的中文学习提交（如 "添加注释"、"添加翻译"、按包记录的进度标记）。
- **技术栈**：J2SE 1.3 + J2EE 1.3（Servlet 2.3 / JSP 1.2 / JTA 1.0 / EJB 2.0）。**这是 2003 年的代码**，
  无泛型、无注解、无 `java.util.concurrent`、无增强 for 循环；依赖 JDK 1.3 API。

> ⚠️ 修改源码时**不要“现代化”**：不要引入泛型/注解/try-with-resources/Lambda 等新语法，不要升级第三方库。
> 保持与 1.3 兼容的写法和既有风格。

## 构建与常用命令

构建工具为 **Ant**（`build.xml`），同时保留了 **Maven 1.x** 配置（`project.xml` / `maven.xml`，非现代 Maven）。
第三方依赖以 jar 形式 vendored 在 `lib/` 下（见 `lib/<库>/` 各子目录）。

常用 Ant target（在仓库根目录执行）：

| 命令 | 作用 |
|------|------|
| `ant build` | 编译 `src/` 主源码到 `.classes/`（含 `rmic` 生成 RMI 桩，并拷贝 `*.xml`/`*.dtd` 资源） |
| `ant alljars` | 生成全部发布 jar：`beans`、`jdbc`、`full`、`srczip` |
| `ant beans` / `ant jdbc` / `ant full` | 分别生成对应 jar（见下方“模块边界”） |
| `ant buildtests` | 编译 `test/` 到 `.testclasses/` |
| `ant tests` | 编译并运行 JUnit 测试，报告输出到 `junit-reports/` |
| `ant javadoc` | 生成 API 文档到 `docs/api/` |
| `ant clean` | 清理所有构建产物目录 |
| `ant release` | 生成发布 zip（`release/`） |

- 编译目标：Java **1.3**（`target="1.3"`），`debug=on`。
- 运行指定测试：`ant tests -Dtest.includes=com/interface21/jdbc/**/*Test*`
  （通过 `test.includes` / `test.excludes` 覆盖 `build.properties` 中的通配规则）。
- 运行/构建前请确认 `lib/` 完整（构建依赖其中的 j2ee、commons-logging、aopalliance、cglib、dom4j 等 jar）。

## 仓库结构

```
src/com/interface21/   主源码（约 404 个 .java）
test/com/interface21/  单元测试（约 141 个）
livetest/              真实数据库测试（MySQL，需手动环境）
sandbox/               实验性/未定型代码（不打入发布 jar）
samples/               示例应用：petclinic、countries、skeletons
lib/                   第三方 jar（vendored，按库名分子目录）
docs/                  文档：HTML 文章、tutorial.pdf、MVC-step-by-step
docs/model/            维护者绘制的 UML 模型图（*.svg）—— 学习产物
build.xml              Ant 构建脚本
build.properties       Ant 属性（版本、目录、测试通配规则）
project.xml/maven.xml  Maven 1.x POM（非现代 Maven，勿用 mvn 现代插件思维解读）
checkstyle.xml         代码规范配置
changelog.txt          版本变更记录（0.9 / 0.9.1 / 1.0 M1）
```

### `com.interface21.*` 包地图

| 包 | 职责 |
|----|------|
| `core` | 框架基础工具（底层支撑） |
| `util` | 通用工具类 |
| `beans` / `beans.factory.*` / `beans.propertyeditors` | IoC 容器：BeanFactory、BeanDefinition、BeanWrapper、XML 配置、属性编辑器 |
| `context` / `context.support` | ApplicationContext，事件、国际化、资源扩展 |
| `aop` / `aop.framework` / `aop.interceptor` / `aop.attributes` | AOP 框架（对接 AOP Alliance，可选 CGLIB） |
| `jdbc.core` / `jdbc.datasource` / `jdbc.object` / `jdbc.util` | JDBC 抽象层：JdbcTemplate、异常体系、RDBMS 对象 |
| `dao` | 数据访问异常体系（DataAccessException 等） |
| `transaction.*` | 事务抽象：策略、拦截器、JTA、支持类 |
| `orm.hibernate.*` / `orm.jdo.*` | Hibernate 2.0 / JDO 1.0 集成 |
| `jndi` | JNDI 抽象 |
| `ejb.*` | EJB 访问与支持类 |
| `remoting.*` | 远程调用：RMI、Caucho Hessian/Burlap |
| `validation` | 数据校验框架 |
| `ui` / `ui.context` | UI 层抽象（视图上下文） |
| `web.*` | Web/MVC 框架：servlet、mvc、view、tags、handler、i18n、theme、bind |

### 三个发布 jar 的模块边界（见 `build.xml` 的 `beans`/`jdbc`/`full`）

- **spring-beans**：仅 Bean 容器 —— `beans/**`、`core/**`、`util/*`
- **spring-jdbc**：容器 + AOP + 事务 + JDBC + ORM —— 在 beans 基础上加 `aop`、`dao`、`jdbc`、`jndi`、`orm`、`transaction`
- **spring-full**：全部 `com/interface21/**`

> 改动某个类时，注意它会被打入哪个 jar，避免破坏模块分层（例如 web 包只出现在 full 中）。

## 代码规范（源自 `checkstyle.xml`）

修改代码须遵守既有约定：

- **缩进**：4 空格，**禁止 Tab 字符**（`TabCharacter` 检查）。
- **大括号**：左花括号换行放置（`LeftCurly` = `nl`），右花括号独占一行（`RightCurly` = `alone`）。
- **行长上限 132**；方法体上限 175 行。
- **Javadoc**：方法、类型、变量均需 Javadoc（`JavadocMethod`/`JavadocType`/`JavadocVariable`）。
- **import**：禁止 `*` 通配导入、禁止未使用/冗余 import。
- **命名**：常量允许 `log` 或标准 `^[A-Z][A-Z0-9_]*$`；其余遵循 Sun 命名规范。
- **禁止**：行尾空格、内联三元（`AvoidInlineConditionals`）、魔法数字、双重检查锁定（DCL）。
- **设计**：`DesignForExtension`、`FinalClass`、`HideUtilityClassConstructor`、`VisibilityModifier`
  （字段默认需为 private/protected，受 `ConstantName` 例外约束）。
- **日志**：统一使用 **Commons Logging**（`org.apache.commons.logging.Log` / `LogFactory`）。

### 学习型注释约定

维护者在原始英文 Javadoc 下方追加**中文翻译**（参见 `BeanWrapperImpl.java` 等文件的类注释）。
新增或修改 Javadoc 时：保留原英文，在其下补充对应中文译文，保持双语对照风格，不删改原作者署名
（`@author Rod Johnson` / `@author Juergen Hoeller` 等）与 `@since` / `@version $Id$` 标签。

## 测试约定

- 测试位于 `test/`（镜像 `com.interface21` 包结构）。
- 命名约定：`*TestSuite` 或 `*Tests`（见 `build.properties` 的 `test.includes`）；抽象基类以 `Abstract*`
  命名并被自动排除（`test.excludes=**/Abstract*`）。
- 测试框架：**JUnit**（配合 easymock / mockobjects）。改完代码应编译并运行相关 `ant tests`。
- `livetest/` 为真实数据库集成测试（MySQL），默认不纳入常规 `ant tests`，需额外环境。

## 工作注意事项

- 这是研读用历史代码：**优先理解、谨慎改动**；如用户未要求，不要重构或“改进”既有设计。
- 翻译/注释类任务：沿用现有双语 Javadoc 风格；提交信息可用中文（与历史提交风格一致）。
- 涉及构建产物（`.classes/`、`dist/`、`junit-reports/` 等）均为生成物，不应手动编辑或提交。
- 添加新类时确认其所属包与 jar 边界，并在对应包补充 `package.html`（`PackageHtml` 检查要求）。
