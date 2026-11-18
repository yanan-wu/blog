# Java 模块系统详解：从 Classpath 混沌到 JPMS 可靠边界

Java 9 之前，Java 世界靠一套延续二十多年的机制组织代码：包（package）+ 类路径（classpath）。它足够简单，也足够能扛事，但代价是——**没有真正的封装边界、没有可靠的依赖声明、没有可裁剪的运行时**。一个 `public` 类放在 `internal` 包里，全世界都能调；两个 JAR 里同名类谁先被加载靠 classpath 顺序；一个缺失的依赖要等到运行时 `NoClassDefFoundError` 才炸。

2017 年 9 月 21 日，JDK 9 正式发布，Project Jigsaw 落地的 **Java Platform Module System（JPMS）** 把"模块"第一次做成语言级、JVM 级、JDK 级的一等公民 。它不是另一个 Maven，不是 OSGi 替代品，而是一套从 `module-info.java` 到类加载、到 `jlink` 镜像构建的完整基础设施。

这篇文章把 JPMS 的核心机制、封装模型、迁移路径和工程决策讲透。



## 一、模块系统解决的四类老问题

JPMS 的出现不是炫技，而是给四个长期痛点补底层能力：

- **依赖不可见**：JAR 自己说不清"我依赖谁、需要谁在运行时存在"，错误推迟到运行期。
- **封装形同虚设**：`public` 是包内可见性的天花板，没有"模块级私有"概念，`com.foo.internal.*` 照样被外部随便引用。
- **分裂包（Split Package）**：classpath 上两个 JAR 含同一个包名，加载顺序决定行为，问题隐蔽且难复现。
- **JDK 太重**：Java 8 的 `rt.jar` 里塞满 Swing、CORBA、XML、SQL，哪怕只写 Hello World 也要拖整套运行时 。

Jigsaw 的目标很明确：让平台可伸缩、让依赖可声明、让封装可强制、让运行时可裁剪 。



## 二、一个模块到底是什么

一个模块 = **命名 + 一组包 + 一个描述符 `module-info.java`**，放在源码根目录。

```java
module com.example.orders {
    requires java.base;              // 隐式存在，可省略
    requires transitive com.example.common;
    requires static org.slf4j;

    exports com.example.orders.api;
    exports com.example.orders.spi to com.example.plugin;

    opens com.example.orders.model to com.fasterxml.jackson.databind;

    uses com.example.orders.spi.Exporter;
    provides com.example.orders.spi.Exporter with com.example.orders.CsvExporter;
}
```

七个指令，覆盖模块系统的全部语义面：

| 指令 | 作用域 | 语义 |
|---|---|---|
| `requires M` | 编译+运行 | 声明对模块 M 的依赖 |
| `requires transitive M` | 编译+运行 | 依赖传递：依赖我的模块自动可读 M |
| `requires static M` | 仅编译 | 运行时可缺席，适合注解处理器、可选集成 |
| `exports P` | 编译+运行 | 导出包 P 的公共类型 |
| `exports P to M` | 编译+运行 | 定向导出，只对指定模块可见 |
| `opens P` | 运行期反射 | 允许深度反射访问私有成员 |
| `opens P to M` | 运行期反射 | 定向开放反射 |
| `uses / provides ... with` | 运行期 | 基于 `ServiceLoader` 的服务解耦 |

这套语义由 **JSR 376** 规范定义，由 **JEP 261** 在 JDK 内实现，配套还有 JEP 200（模块化 JDK）、JEP 201（模块化源码）、JEP 220（模块化运行时镜像）、JEP 260（封装内部 API）、JEP 282（`jlink`）。



## 三、强封装：JPMS 的真正杀手锏

传统 Java 的"封装"停在类层面。JPMS 把封装抬到**模块层面**，且由 JVM 强制：

- 没被 `exports` 的包，**编译期不可见，运行期连反射都拿不到**；
- 没被 `opens` 的包，框架不能通过 `Class.getDeclaredFields()` 碰私有字段；
- JDK 内部 API（`sun.misc.Unsafe`、`com.sun.*` 大部分）默认不可达，想用要走 `--add-exports` / `--add-opens` 。

这一条直接改写了 Java 的生态规则：过去靠"碰内部实现"活着的库，必须要么适配模块系统，要么显式声明开放。

> 注意区分两个 JVM 参数：
> - `--add-exports`：把某个包在编译/运行期"假装导出"，解决类型访问；
> - `--add-opens`：把某个包在运行期开放给反射，解决私有字段访问。
> 它们只是过渡期逃生舱，不是长期架构方案。



## 四、模块路径 vs 类路径：三种"模块身份"

JPMS 最容易被忽略的设计，是它**完全向后兼容**——classpath 没被废除，而是被重新解释成三种身份：

**1. 命名模块（Explicit Module）**
带 `module-info.class` 的 JAR，放在 `--module-path` 上。依赖、导出、开放全部显式声明。

**2. 自动模块（Automatic Module）**
普通 JAR 放在 `--module-path` 上，没有 `module-info`，JVM 自动给它一个名字：优先读 `MANIFEST.MF` 里的 `Automatic-Module-Name`，否则从文件名推导——`my-utils-1.2.jar` → `my.utils`，版本号丢弃、非字母数字转点 。
自动模块**导出所有包、读取所有模块**，是旧生态通往新世界的桥。

**3. 未命名模块（Unnamed Module）**
所有在 classpath 上的类，统一归到一个"没有名字的模块"。它能读所有命名模块导出的包，但**命名模块不能 `requires` 它**——因为它没名字 。

这个设计决定了迁移策略：**自下而上**。先让底层库声明 `Automatic-Module-Name` 占住名字，再逐步补 `module-info`，最后应用层切到 module path。



## 五、服务解耦：uses / provides 的语言级落地

SPI（Service Provider Interface）过去靠 `META-INF/services` 文件 + `ServiceLoader` 手工约定。JPMS 把它做成模块描述符的一等公民：

```java
// 消费方
module com.example.app {
    uses com.example.spi.PaymentService;
}

// 提供方
module com.example.alipay {
    requires com.example.spi;
    provides com.example.spi.PaymentService with com.example.alipay.AlipayImpl;
}
```

编译期双方互不知道对方存在，运行期由 `ServiceLoader.load(PaymentService.class)` 发现实现。这对插件化架构、支付渠道、日志后端、序列化协议切换是质的提升——**新增实现不需要改任何调用方代码**。



## 六、jlink：把 JDK 裁到你真正需要的样子

JPMS 给 JDK 自身拆出约 90+ 个模块（`java.base`、`java.sql`、`java.xml`、`java.desktop`……）。基于此，`jlink` 可以把"用到的模块 + 你的应用"打成**自包含运行时镜像**：

```bash
jlink --module-path $JAVA_HOME/jmods:out \
      --add-modules com.example.app \
      --launcher myapp=com.example.app/com.example.Main \
      --output myapp-runtime
```

输出物不再依赖外部 JDK 安装，体积从完整 JDK 的数百 MB 降到几十 MB 级，对容器镜像、Serverless、边缘部署价值直接。这是 JPMS 在"云原生密度"维度上最务实的回报。



## 七、迁移实战：三个高频坑

**坑 1：反射框架集体报警**
Spring、Hibernate、Jackson 默认靠反射摸私有字段。模块化后没 `opens` 就抛 `InaccessibleObjectException`。
正确做法：只 `opens` 实体包给具体框架，例如 `opens com.example.model to com.fasterxml.jackson.databind`，别用 `open module` 一把梭。

**坑 2：分裂包启动即失败**
两个模块不能声明同一个包名，否则启动直接报错（classpath 时代是静默覆盖）。迁移前用 `jdeps` 扫依赖图，提前合并或重命名包。

**坑 3：命名模块无法依赖 classpath JAR**
只要你的模块声明了 `requires xxx`，xxx 就必须在 module path 上是命名/自动模块。把遗留 JAR 挪到 module path 变自动模块，或给第三方库等它自己出 `module-info`。

`jdeps --generate-module-info` 可以基于已有 JAR 生成草稿 `module-info.java`，是迁移起点而非终点——自动推导不知道哪些包该 `opens`、哪些依赖该 `transitive`，这些要靠人对业务契约的判断。



## 八、什么时候该上 JPMS，什么时候先别

**值得投入的场景**：
- 做基础框架/中间件/SDK，需要强封装和清晰 API 边界；
- 追求极小容器镜像、Serverless 冷启动优化，用 `jlink` 定制运行时；
- 插件化架构，需要服务解耦和运行时扩展；
- 大型单体应用，依赖关系已经乱到没人敢动。

**暂缓的场景**：
- 纯业务逻辑的小型服务，Maven/Gradle 依赖管理已经够用；
- 重度依赖未适配 JPMS 的反射框架（老版本 Hibernate、字节码增强库）；
- 团队对模块边界认知不一致，强行拆只会产生"假模块"。

现实里，大多数企业应用当前仍是**混合态**：业务代码跑在未命名模块，关键库逐步模块化，用 `--add-opens` 过渡。这完全合法，也是 Jigsaw 设计时就预留的路径。



## 结语

JPMS 不是"让 Java 更好写"的工具，而是"让 Java 更大规模可维护"的底座。它把过去靠约定、靠规范文档、靠团队纪律维持的边界，变成了 JVM 强制执行的契约。

对多数团队来说，模块化的收益不是立竿见影的"开发更快"，而是**三年后架构不腐化**：依赖图可读、内部 API 不可越界、运行时可裁剪、插件可热插拔。这也是为什么它被认为是 Java 9 以来最被低估、也最影响深远的那次重构。

真正难的从来不是写 `module-info.java`，而是回答一个问题：**你的系统里，哪些东西值得被封装，哪些契约值得被显式承诺**。想清楚这一层，JPMS 才从语法变成架构。
