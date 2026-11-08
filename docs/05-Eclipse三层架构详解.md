# Eclipse 三层架构详解

> 面向刚开始学 Eclipse RCP（Rich Client Platform，富客户端平台）的读者。目标是让你看到任意一个 Eclipse 概念时，能判断它属于哪一层、该去哪里找、出错时该查什么。
> 前置阅读：[00-概念入门](00-概念入门.md)
>
> 说明：本文首次出现的英文缩写会给出英文全称和中文翻译；完整对照表见下方。

## 零、缩写与全称速查

| 缩写/术语       | 英文全称                                  | 中文翻译               | 备注                                           |
| ----------- | ------------------------------------- | ------------------ | -------------------------------------------- |
| RCP         | Rich Client Platform                  | 富客户端平台             | 基于 Eclipse 平台构建桌面应用                          |
| IDE         | Integrated Development Environment    | 集成开发环境             | Eclipse IDE 是 Eclipse 平台的一个产品                |
| OSGi        | Open Services Gateway initiative      | 开放服务网关规范           | 模块化规范；Equinox 是它的实现                          |
| JVM         | Java Virtual Machine                  | Java 虚拟机           | 运行 Java 字节码                                  |
| JRE         | Java Runtime Environment              | Java 运行时环境         | 只能运行 Java 程序                                 |
| JDK         | Java Development Kit                  | Java 开发工具包         | 含 JRE、编译器和工具                                 |
| JAR         | Java Archive                          | Java 归档文件          | 通常作为 Bundle 的载体                              |
| API         | Application Programming Interface     | 应用程序编程接口           | `Export-Package` 暴露的就是 API                   |
| BREE        | Bundle Required Execution Environment | Bundle 所需执行环境      | `MANIFEST.MF` 中声明的最低 Java 版本                 |
| JDT         | Java Development Tools                | Java 开发工具          | Eclipse 的 Java IDE 功能                        |
| PDE         | Plug-in Development Environment       | 插件开发环境             | 开发 Bundle、Feature、Product 的工具                |
| ECJ         | Eclipse Compiler for Java             | Eclipse Java 编译器   | Eclipse 内部使用的 Java 编译器                       |
| JDI         | Java Debug Interface                  | Java 调试接口          | JDT 调试器的基础                                   |
| EGit        | Eclipse Git Team Provider             | Eclipse 的 Git 集成   | 基于 JGit                                      |
| JGit        | Java Git                              | 纯 Java 实现的 Git 库   | EGit 的底层                                     |
| m2e         | Maven Integration for Eclipse         | Maven 与 Eclipse 集成 | Eclipse 的 Maven 插件                           |
| p2          | Provisioning Platform 2               | 配置/供应平台（第 2 代）     | Eclipse 的安装/更新系统                             |
| IU          | Installable Unit                      | 可安装单元              | p2 的安装单位                                     |
| SWT         | Standard Widget Toolkit               | 标准控件工具包            | Eclipse 的原生控件库                               |
| JFace       | —（不是缩写，是框架名）                          | Eclipse UI 框架      | 建立在 SWT 之上                                   |
| e4          | Eclipse 4                             | Eclipse 第 4 代应用平台  | 不是缩写；提供应用模型、DI、CSS                           |
| EMF         | Eclipse Modeling Framework            | Eclipse 建模框架       | e4 应用模型基于 EMF                                |
| MVC         | Model-View-Controller                 | 模型-视图-控制器          | JFace Viewer 的核心思想                           |
| DI          | Dependency Injection                  | 依赖注入               | e4 用 `IEclipseContext` 实现                    |
| CSS         | Cascading Style Sheets                | 层叠样式表              | e4 用它做主题                                     |
| UI          | User Interface                        | 用户界面               |                                              |
| JNI         | Java Native Interface                 | Java 本地接口          | SWT 调用原生控件时使用                                |
| GC          | Graphics Context                      | 图形上下文              | SWT 中为图形上下文；JVM 参数中为垃圾回收（Garbage Collection） |
| XML         | Extensible Markup Language            | 可扩展标记语言            | `plugin.xml`、`feature.xml` 等                 |
| MANIFEST.MF | Manifest file                         | Bundle 元数据清单       | 不是缩写；OSGi Bundle 的清单文件                       |
| `M` 前缀      | Model                                 | 模型                 | 如 `MApplication`、`MPart`                     |
| `I` 前缀      | Interface                             | 接口                 | 如 `IWorkbench`、`IEclipseContext`             |
| 3.x         | Eclipse 3.x                           | Eclipse 第 3 代平台    | 通过兼容层继续存在                                    |
| 4.x / e4    | Eclipse 4.x                           | Eclipse 第 4 代平台    | 现代 Workbench 运行时                             |

## 一、先给一张全景图

“三层”是一个便于理解的教学划分。实际运行时，最底下还有第 0 层，旁边还有 p2（Provisioning Platform 2，配置/供应平台）这类横切能力：

```text
┌─────────────────────────────────────────────────────────────┐
│ 第 3 层  IDE 工具层                                          │
│ JDT（Java 开发）、PDE（插件开发）、EGit、m2e、XML 编辑器      │
│ 作用：把 Eclipse 变成“开发工具”                               │
│ 你的 RCP 产品通常【不需要】这一层                             │
├─────────────────────────────────────────────────────────────┤
│ 第 2 层  桌面应用框架层                                      │
│ Workbench / e4 应用模型 / SWT / JFace / CSS                  │
│ 作用：窗口、菜单、视图、编辑器、布局、事件、数据绑定          │
│ 你的业务代码主要写在这一层                                    │
├─────────────────────────────────────────────────────────────┤
│ 第 1 层  运行时与模块层                                      │
│ Equinox（OSGi）/ Bundle / 扩展注册表 / Platform Runtime       │
│ 作用：模块加载、依赖解析、生命周期、扩展点发现                │
│ 你的插件以 Bundle 形式存在                                    │
├─────────────────────────────────────────────────────────────┤
│ 第 0 层  JVM 与启动器                                        │
│ eclipse.exe / eclipse.ini / config.ini / Java 21             │
│ 作用：启动 JVM、读取配置、拉起 Equinox                        │
└─────────────────────────────────────────────────────────────┘

横切能力：p2（安装/更新/依赖解析）、日志、偏好设置、Job、帮助系统
```

一句话记忆：

- **第 0 层**：把机器点着；
- **第 1 层**：把零件装起来；
- **第 2 层**：把界面搭起来；
- **第 3 层**：把开发工具放进来。

## 二、为什么 Eclipse 要分层

因为 Eclipse 的目标不是“做一个 IDE（Integrated Development Environment，集成开发环境）”，而是“做一个可以组装出各种桌面工具的平台”。

分层的直接好处：

1. **可裁剪**：你的产品可以只要第 1、2 层 + 自己的插件，不携带 JDT/PDE。
2. **可替换**：SWT/JFace 之上可以换 Workbench 模型；3.x 和 e4 可以共存。
3. **可扩展**：任何人通过扩展点增加功能，不用改平台源码。
4. **可自举**：Eclipse IDE 本身就是用这套平台开发的，开发工具也是插件。
5. **可排错**：知道错误属于哪一层，就知道该看 MANIFEST、模型文件还是 `eclipse.ini`。

最重要的一个认知：

> **Eclipse IDE ≠ Eclipse Platform。**
> IDE 只是“平台 + JDT/PDE/Git 等工具插件”组装出来的一个 Product。
> 你做 RCP，是在用平台做“另一个 Product”。

## 三、第 0 层：JVM（Java Virtual Machine，Java 虚拟机）与启动器

这一层最容易被忽略，但启动失败往往出在这里。

### 3.1 启动链路

```text
双击 eclipse.exe
   │
   ├─ 读取 eclipse.ini
   │    ├─ -vm：用哪个 Java
   │    ├─ -vmargs：堆大小、编码、GC（Garbage Collection，垃圾回收）
   │    └─ -product：启动哪个产品
   │
   ├─ 启动 JVM（java.exe / javaw.exe）
   │
   ├─ 读取 configuration/config.ini
   │    ├─ osgi.bundles：要安装哪些 Bundle
   │    ├─ eclipse.product：产品 ID
   │    ├─ osgi.requiredJavaVersion：最低 Java 版本
   │    └─ osgi.bundles.defaultStartLevel：默认启动级别
   │
   └─ 启动 Equinox（进入第 1 层）
```

### 3.2 关键文件

| 文件 | 作用 |
|---|---|
| `eclipse.exe` | Windows 原生启动器 |
| `eclipse.ini` | 启动器参数：JVM、内存、编码、product |
| `configuration/config.ini` | OSGi 运行时配置：Bundle 列表、启动级别、product ID |
| `plugins/` | 所有 Bundle 的存放目录 |
| `p2/` | p2 的安装/更新元数据 |
| `jre/`（如果有） | 安装包自带的 JRE（Java Runtime Environment，Java 运行时环境） |

### 3.3 为什么 Java 版本问题总是最先出现

因为 JVM 还没起来，Eclipse 的 Java 代码一行都没执行。

典型错误：

```text
Java was started but returned exit code=13
```

通常是 `-vm` 路径错了，或者指向了不存在的 `javaw.exe`。

```text
JVM terminated. Exit code=1
```

常见原因：

- `-vm` 写在了 `-vmargs` 后面；
- `-vm` 和路径写在同一行；
- `-Xmx` 超过机器内存；
- JDK（Java Development Kit，Java 开发工具包）版本低于 `osgi.requiredJavaVersion`。

### 3.4 第 0 层和 BREE（Bundle Required Execution Environment，Bundle 所需执行环境）的关系

- `eclipse.ini` / `config.ini` 决定“**运行 Eclipse 的 JVM**”。
- `MANIFEST.MF` 的 `Bundle-RequiredExecutionEnvironment`（BREE）决定“**这个 Bundle 最低需要什么 Java**”。
- 两者不匹配，就会出现“IDE 能启动，但某个 Bundle 解析失败”或“产品在客户机器上起不来”。

### 3.5 类比

第 0 层是**电闸、插头和供电**。电闸没开，后面装修得再好也没用。

## 四、第 1 层：运行时与模块层

这是 Eclipse 的“地基”。核心是 **Equinox（OSGi 实现；OSGi = Open Services Gateway initiative，开放服务网关规范）**。

### 4.1 Equinox 是什么

- OSGi 是一套**模块化规范**；
- Equinox 是 Eclipse 对 OSGi 的实现；
- 它负责：安装 Bundle、解析依赖、加载类、管理生命周期、提供 Service Registry。

你可以把它理解成一个“**带依赖解析和类加载隔离的 Java 模块容器**”。

### 4.2 Bundle 是什么

一个 Bundle 就是一个 JAR（Java Archive，Java 归档文件），加上特殊的 `META-INF/MANIFEST.MF`：

```text
Bundle-SymbolicName: com.example.hello
Bundle-Version: 1.0.0.qualifier
Bundle-RequiredExecutionEnvironment: JavaSE-21
Bundle-Activator: com.example.hello.Activator
Bundle-ActivationPolicy: lazy

Export-Package: com.example.hello.api
Import-Package: org.eclipse.core.runtime, org.eclipse.jface.viewers
Require-Bundle: org.eclipse.ui
```

逐个解释：

| 头 | 含义 |
|---|---|
| `Bundle-SymbolicName` | 全局唯一 ID，类似“包名” |
| `Bundle-Version` | 版本，支持 `1.2.3.qualifier` |
| `Bundle-RequiredExecutionEnvironment` | 最低 Java 环境，如 `JavaSE-21` |
| `Bundle-Activator` | Bundle 启动入口；能不用就不用 |
| `Bundle-ActivationPolicy: lazy` | 第一次真正用到类时才启动 |
| `Export-Package` | 哪些包是公开 API（Application Programming Interface，应用程序编程接口），别人能看见 |
| `Import-Package` | 按“包”依赖别人 |
| `Require-Bundle` | 按“整个 Bundle”依赖别人 |

**Bundle 和 Plugin 的关系：**

- 在 Eclipse 语境里，二者经常混用；
- 严格说，Bundle 是 OSGi 的模块单位，Plugin 是 Eclipse 的功能贡献单位；
- 现代 Eclipse 里通常一个 Bundle 就是一个 Plugin（带 `plugin.xml` 或 `Application.e4xmi`）。

### 4.3 类加载隔离：OSGi 最重要的机制

普通 Java：

```text
classpath = a.jar;b.jar;c.jar
所有类在一个“大池子”里，谁都能看见谁
```

OSGi：

```text
Bundle A 有自己的 ClassLoader
Bundle B 有自己的 ClassLoader
A 只能看见 B 通过 Export-Package 暴露的包
```

这带来两个结果：

**好处：**

- 模块边界清晰；
- 同一个包可以有多版本；
- 不需要把所有 JAR 堆在一起；
- 可以按需加载，启动更快。

**坏处：**

- 忘记 `Import-Package` 就 `NoClassDefFoundError`；
- 编译时能找到，运行时不一定能找到；
- 依赖版本冲突时，错误信息比较绕。

### 4.4 依赖解析（Resolution）

Equinox 启动时会做一次“拼图”：

1. 读取所有 Bundle 的 `MANIFEST.MF`；
2. 看每个 `Import-Package` / `Require-Bundle` 能不能被满足；
3. 检查版本范围、BREE；
4. 选出一组能互相兼容的 Bundle；
5. 这个过程叫 **resolve（解析）**。

解析失败时，Bundle 可能停在：

```text
INSTALLED（已安装，但依赖没满足）
RESOLVED（依赖满足，但还没启动）
ACTIVE（已启动）
```

你在 IDE 里看到的：

```text
Bundle cannot be resolved
Missing Constraint: Import-Package: xxx
```

就是第 1 层的问题。

### 4.5 生命周期

```text
INSTALLED → RESOLVED → STARTING → ACTIVE → STOPPING → UNINSTALLED
```

- **INSTALLED**：JAR 已经在，依赖还没满足；
- **RESOLVED**：依赖满足，随时可启动；
- **STARTING**：正在执行 `BundleActivator.start()`；
- **ACTIVE**：运行中；
- **STOPPING**：正在执行 `stop()`；
- **UNINSTALLED**：已卸载。

现代 Eclipse 推荐：

- 少用 Activator；
- 用 `@PostConstruct` / `@PreDestroy` 管理组件生命周期；
- 用 DI（Dependency Injection，依赖注入）获取服务，而不是到处 `BundleContext.getService()`。

### 4.6 扩展注册表：Eclipse 的“插槽系统”

OSGi 只解决“模块怎么加载”，不解决“功能怎么挂到界面上”。

Eclipse 在 OSGi 之上加了 **Extension Registry（扩展注册表）**：

- 平台定义**扩展点（Extension Point）**，例如 `org.eclipse.ui.views`；
- 插件在 `plugin.xml` 里声明**扩展（Extension）**；
- 平台启动时读取这些声明，按需实例化你的类。

示例：贡献一个视图（3.x 风格）

```xml
<extension point="org.eclipse.ui.views">
  <view
      id="com.example.hello.view"
      name="Hello View"
      class="com.example.hello.HelloView"
      category="com.example.hello.category"/>
</extension>
```

平台看到这段 XML，就知道：

- 有一个视图叫 `Hello View`；
- 用户打开它时，实例化 `com.example.hello.HelloView`；
- 这个贡献来自 `com.example.hello` 这个 Bundle。

e4 风格则把结构写进 `Application.e4xmi`，但本质一样：**声明式贡献，平台负责创建**。

### 4.7 Platform Runtime 还提供什么

第 1 层不只有 OSGi，还有一组“平台运行时”能力：

| 能力 | 代表 Bundle/API | 作用 |
|---|---|---|
| 扩展注册表 | `org.eclipse.equinox.registry` | 读取 `plugin.xml` 扩展 |
| Job/并发 | `org.eclipse.core.jobs` | 后台任务、进度、取消 |
| 偏好设置 | `org.eclipse.equinox.preferences` | 保存用户配置 |
| 适配器 | `org.eclipse.core.runtime.IAdaptable` | 对象能力转换 |
| 日志 | `org.eclipse.core.runtime.ILog` | 写 `.metadata/.log` |
| 应用模型 | `org.eclipse.equinox.app` | 定义 application/product 启动入口 |

这些是第 2 层和第 3 层共同依赖的“基础设施”。

### 4.8 p2（Provisioning Platform 2，配置/供应平台）：安装与更新

p2 是 Eclipse 的**配置/供应（provisioning）系统**，负责：

- 安装插件；
- 更新插件；
- 卸载插件；
- 解析安装依赖；
- 生成/读取 update site。

它不是 OSGi 运行时的一部分，而是横跨“安装/交付”的能力。

p2 的两个核心概念：

- **metadata**：有哪些可安装单元（IU，Installable Unit），依赖是什么；
- **artifact**：实际的 JAR/文件在哪里。

你在 IDE 里看到的：

```text
Help > Install New Software...
Help > Check for Updates
```

以及产品导出后的 `p2/` 目录，都是 p2。

### 4.9 第 1 层你接触到的文件/界面

| 你看到的 | 属于第 1 层的原因 |
|---|---|
| `MANIFEST.MF` | Bundle 元数据、依赖、导出包 |
| `plugin.xml` | 扩展点贡献（3.x） |
| `Application.e4xmi` | e4 应用模型（第 2 层使用，但作为 Bundle 资源） |
| Plug-in Editor 的 Dependencies 页 | 配置 Import/Require |
| Target Platform | 决定解析时能用哪些 Bundle |
| `.metadata/.log` | Bundle 解析/启动错误 |
| `configuration/config.ini` | OSGi 启动配置 |

### 4.10 第 1 层常见错误

| 错误 | 含义 | 先查什么 |
|---|---|---|
| `Bundle cannot be resolved` | 依赖不满足 | MANIFEST 的 Import/Require、目标平台 |
| `Missing Constraint: Import-Package` | 找不到某个包 | 目标平台有没有提供这个包的 Bundle |
| `NoClassDefFoundError` | 运行时类不可见 | 包是否 Export/Import，Bundle 是否 ACTIVE |
| `UnsupportedClassVersionError` | class 版本高于 JVM | BREE、编译 JDK、运行 JDK |
| `BundleException: Could not resolve module` | 解析失败 | 版本范围、可选依赖、BREE |
| `ClassCastException`（同名类） | 同一个类被两个 Bundle 加载 | 是否重复打包了 API 包 |

### 4.11 第 1 层类比

第 1 层是**地基、承重墙、水电管线**：

- Bundle 是预制墙板；
- `MANIFEST.MF` 是墙板接口说明；
- Equinox 是施工队；
- 扩展注册表是墙上的标准插座；
- p2 是物业的“加装/改造”服务。

## 五、第 2 层：桌面应用框架层

这一层决定“用户看到什么、怎么交互”。你的大部分代码在这里。

它内部还可以再分三小层：

```text
Workbench / e4（应用结构、窗口、Part、命令、DI、CSS）
        ↑
JFace（Viewer、向导、首选项、对话框、数据绑定）
        ↑
SWT（原生控件、布局、事件、绘图、资源）
```

### 5.1 SWT（Standard Widget Toolkit，标准控件工具包）：原生控件层

SWT（Standard Widget Toolkit，标准控件工具包）是 Eclipse 自己的 UI（User Interface，用户界面）工具包。

特点：

- 直接调用操作系统原生控件（Windows 上是 Win32）；
- 外观和系统一致，性能好；
- 通过 JNI（Java Native Interface，Java 本地接口）调用 native 库，所以有平台相关 fragment；
- **单 UI 线程**：所有控件操作必须在 UI 线程；
- **资源要手动释放**：`Color`、`Font`、`Image`、`GC`（Graphics Context，图形上下文）用完 `dispose()`。

核心类：

| 类 | 作用 |
|---|---|
| `Display` | 一个 UI 线程的入口，事件循环 |
| `Shell` | 窗口 |
| `Composite` | 容器 |
| `Label`/`Text`/`Button`/`Table`/`Tree` | 控件 |
| `GridLayout`/`GridData`/`RowLayout`/`FormLayout` | 布局 |
| `GC`（Graphics Context，图形上下文） | 绘图 |
| `Image`/`Color`/`Font` | 资源 |

一个最小 SWT 程序长这样：

```java
Display display = new Display();
Shell shell = new Shell(display);
shell.setText("Hello SWT");
shell.setLayout(new GridLayout(1, false));

Label label = new Label(shell, SWT.NONE);
label.setText("Hello, Eclipse RCP");

shell.pack();
shell.open();
while (!shell.isDisposed()) {
    if (!display.readAndDispatch()) {
        display.sleep();
    }
}
display.dispose();
```

这段代码说明了 SWT 的本质：**你自己管理窗口、事件循环和资源**。

### 5.2 JFace：MVC（Model-View-Controller，模型-视图-控制器）封装层

JFace 是 Eclipse 的 UI 框架（JFace 不是缩写），它不是另一个 UI（User Interface，用户界面）工具包；它建立在 SWT 之上，帮你少写重复代码。

它提供：

| 能力 | 代表类 |
|---|---|
| 表格/树/列表的 MVC | `TableViewer`、`TreeViewer`、`ListViewer` |
| 内容/标签提供者 | `IStructuredContentProvider`、`ITableLabelProvider` |
| 排序/过滤/编辑 | `ViewerSorter`、`ViewerFilter`、`EditingSupport` |
| 向导 | `Wizard`、`WizardPage`、`WizardDialog` |
| 首选项 | `PreferencePage`、`PreferenceInitializer` |
| 对话框 | `TitleAreaDialog`、`MessageDialog`、`InputDialog` |
| 数据绑定 | `DataBindingContext`、`IObservableValue` |
| 资源管理 | `ImageDescriptor`、`JFaceResources` |
| 进度 | `ProgressMonitorDialog`、`IRunnableWithProgress` |

JFace 的核心思想是 **MVC（Model-View-Controller，模型-视图-控制器）**：

```text
模型（你的数据）
   ↑
ContentProvider（数据怎么取）
   ↑
Viewer（TableViewer/TreeViewer）
   ↑
LabelProvider（每行怎么显示）
   ↑
SWT 控件（Table/Tree）
```

所以你写 JFace 时，通常不直接 `new TableItem`，而是：

1. 给 Viewer 设置 ContentProvider；
2. 给 Viewer 设置 LabelProvider；
3. 给 Viewer 设置输入 `setInput(...)`；
4. 数据变化时刷新 Viewer。

### 5.3 Workbench 与 e4：应用结构层

Workbench 负责“整个应用长什么样、怎么组织”。

在 e4（Eclipse 4，Eclipse 第 4 代应用平台；不是缩写）里，应用结构是一个 **EMF（Eclipse Modeling Framework，Eclipse 建模框架）模型**，保存在 `Application.e4xmi` 中。

核心模型元素：

> 下表中的 `M` 前缀表示 Model（模型），例如 `MApplication` 是“模型层的 Application”。

| 元素 | 含义 |
|---|---|
| `MApplication` | 应用根节点 |
| `MWindow` | 窗口 |
| `MPerspective` | 透视图（一组 Part 的布局） |
| `MPart` | 视图/编辑器的统一抽象 |
| `MPartStack` | Part 的堆叠容器（Tab） |
| `MMenu` / `MHandledMenuItem` | 菜单 |
| `MToolBar` / `MTrimBar` | 工具栏/窗口边缘 |
| `MCommand` / `MHandler` | 命令与处理器 |

e4 的运行时大致是：

```text
Application.e4xmi（模型）
   ↓ 读取
EModelService / 模型处理器
   ↓ 渲染
SWT Shell + Composite + 控件
   ↓
你看到的窗口
```

**关键点：模型是数据，渲染器负责把它变成 SWT 控件。**
所以你可以：

- 用模型编辑器改 UI 结构；
- 运行时用 `EModelService` 动态加 Part；
- 用 fragment 给已有应用追加结构；
- 用 CSS（Cascading Style Sheets，层叠样式表）改样式；
- 用 DI 注入服务。

#### e4 的依赖注入

e4 用 `IEclipseContext`（`I` 前缀表示 Interface，接口）管理对象和服务，层级大致是：

```text
Application Context
   └─ Window Context
        └─ Perspective Context
             └─ Part Context
```

`@Inject` 会从当前层往上找：

```java
public class HelloPart {
    @Inject
    private EPartService partService;

    @PostConstruct
    public void createComposite(Composite parent) {
        // 创建 UI
    }

    @Execute
    public void save() {
        // 命令执行
    }

    @CanExecute
    public boolean canSave() {
        // 命令是否可用
    }
}
```

常用注解：

| 注解 | 作用 |
|---|---|
| `@Inject` | 注入对象/服务 |
| `@Named` | 按名称注入 |
| `@Optional` | 可选注入 |
| `@Preference` | 注入偏好设置 |
| `@EventTopic` | 订阅事件 |
| `@PostConstruct` | Part 创建后执行 |
| `@PreDestroy` | Part 销毁前执行 |
| `@Focus` | 获得焦点时执行 |
| `@Execute` | 命令执行 |
| `@CanExecute` | 命令是否可用 |

常用服务：

| 服务 | 作用 |
|---|---|
| `EPartService` | 打开/关闭/激活 Part |
| `ESelectionService` | 全局选择 |
| `EEventBroker` | 事件发布/订阅 |
| `ECommandService` | 命令管理 |
| `EHandlerService` | 处理器管理 |
| `EModelService` | 操作应用模型 |
| `MApplication` | 应用根对象 |

### 5.4 3.x 兼容层

Eclipse 4 并不是把 3.x 全删了，而是保留了兼容层：

- `org.eclipse.ui` 提供 `IWorkbench`、`ViewPart`、`EditorPart`、透视图、advisor；
- 旧的 `plugin.xml` 扩展点仍然可用；
- 旧的 `ApplicationWorkbenchAdvisor`、`WorkbenchWindowAdvisor` 仍然存在；
- 现代 Eclipse 的 Workbench 内部是 e4，3.x API 跑在兼容层上。

所以你会看到两种风格：

**3.x 风格：**

```java
public class HelloView extends ViewPart {
    @Override
    public void createPartControl(Composite parent) {
        // ...
    }
}
```

**e4 风格：**

```java
public class HelloPart {
    @PostConstruct
    public void createComposite(Composite parent) {
        // ...
    }
}
```

为什么两种都要认识？

- 企业老项目大量使用 3.x API；
- 新项目更推荐 e4；
- 很多项目是混合的；
- 看懂 3.x 才能维护/迁移老代码。

### 5.5 第 2 层你接触到的文件/界面

| 你看到的 | 说明 |
|---|---|
| `Application.e4xmi` | e4 应用模型 |
| `plugin.xml` | 3.x 扩展点贡献 |
| `@PostConstruct` / `@Execute` | e4 生命周期/命令 |
| `ViewPart` / `EditorPart` | 3.x 视图/编辑器 |
| `Perspective` | 透视图 |
| 菜单/工具栏/状态栏 | Workbench 提供 |
| CSS 文件 | e4 主题样式 |
| `workbench.xmi` | 工作空间里保存的布局状态 |

### 5.6 第 2 层常见错误

| 错误 | 原因 | 处理 |
|---|---|---|
| `Invalid thread access` | 非 UI 线程操作控件 | `Display.asyncExec/syncExec` |
| `Widget is disposed` | 控件已销毁还在用 | 检查 Part 生命周期 |
| 视图不显示 | 扩展点/模型没配对 | 查 `plugin.xml` / `Application.e4xmi` |
| 布局错乱 | Layout/GridData 用错 | 查父容器 Layout 和子控件 GridData |
| 资源泄漏 | Color/Font/Image 没 dispose | 用 `ResourceManager` 或手动释放 |
| `Referenced part does not exist yet` | e4 启动顺序/Part 引用问题 | 查模型和启动日志，通常是警告 |
| 命令灰色不可用 | `@CanExecute` 返回 false | 查选择/上下文/处理器 |

### 5.7 第 2 层类比

第 2 层是**房间、门窗、家具和装修**：

- SWT 是墙、窗、门、桌椅（原生控件）；
- JFace 是模块化家具（Viewer、向导、首选项）；
- Workbench/e4 是户型图和智能家居系统（Part、Perspective、命令、DI、CSS）。

## 六、第 3 层：IDE 工具层

这一层把 Eclipse 从“平台”变成“开发工具”。

### 6.1 JDT：Java 开发工具

JDT（Java Development Tools，Java 开发工具）提供：

- Java 编译器 **ECJ**（Eclipse Compiler for Java，Eclipse Java 编译器）；
- Java 模型（`IJavaProject`、`IType`）；
- Java 编辑器（语法高亮、内容辅助、重构）；
- 调试器（JDI，Java Debug Interface，Java 调试接口）；
- JUnit 集成。

注意：Eclipse 编译 Java 代码用的是 **ECJ**，不是 `javac`。所以有时 `javac` 能编过的代码，ECJ 会报错；反之亦然。

### 6.2 PDE：插件开发工具

PDE（Plug-in Development Environment，插件开发环境）提供：

- Plug-in 项目向导；
- `MANIFEST.MF` 编辑器；
- `plugin.xml` 扩展点编辑器；
- Target Platform 配置；
- Eclipse Application 启动配置；
- Plug-in Spy / CSS Spy / Model Spy；
- Feature/Product 编辑与导出；
- PDE Error Log。

**PDE 是第 3 层的工具，但它开发的是第 1/2 层的产物。**

### 6.3 EGit、m2e、XML 编辑器

- **EGit（Eclipse Git Team Provider）/ JGit（Java Git）**：Git 集成；JGit 是纯 Java 实现的 Git 库；
- **m2e（Maven Integration for Eclipse）**：Maven 与 Eclipse 集成；
- **XML（Extensible Markup Language，可扩展标记语言）编辑器**：编辑 `plugin.xml`、`.product`、`feature.xml` 等；
- 其他：Terminal、Marketplace Client、Help。

它们都是插件，只是“开发工具类插件”。

### 6.4 为什么你的 RCP 产品通常不需要第 3 层

假设你做一个“设备监控客户端”：

- 需要：窗口、表格、图表、菜单、偏好设置、数据持久化；
- 不需要：Java 编辑器、Git 视图、Maven 向导、PDE。

所以你的产品目标平台通常只包含：

```text
第 1 层：Equinox / Platform Runtime
第 2 层：SWT / JFace / e4 Workbench
你的插件
（可选）EMF、图表库、数据库驱动
```

而不是整个 Eclipse IDE。

如果你把整个 IDE 当目标平台，产品会变得很大，启动变慢，还会带入一堆无关依赖。

### 6.5 自举：Eclipse 开发 Eclipse

Eclipse IDE 的有趣之处：

```text
Eclipse IDE
   = 第 1 层（Equinox）
   + 第 2 层（Workbench/SWT/JFace）
   + 第 3 层（JDT/PDE/EGit/m2e）
```

你在 IDE 里用 JDT 写 Java，用 PDE 写插件，用 EGit 提交代码。这些工具本身也是插件，跑在同一个平台上。

这就是“自举（self-hosting）”：**用 Eclipse 开发 Eclipse 插件**。

### 6.6 第 3 层类比

第 3 层是**工具间**：

- 电钻、锯子、水平仪（JDT/PDE）；
- 图纸柜（Target Platform、Product 配置）；
- 材料清单（Feature）；
- 施工日志（PDE Error Log）。

工具间不是房子的一部分；你的客户搬进房子时，不需要把工具间也搬过去。

## 七、三层如何协作

### 7.1 启动的 11 个步骤

| 步骤 | 层 | 发生什么 |
|---|---|---|
| 1 | 0 | 双击 `eclipse.exe`，读取 `eclipse.ini` |
| 2 | 0 | 启动 `-vm` 指定的 JVM（如 `D:\Java21`） |
| 3 | 0/1 | 读取 `configuration/config.ini`，知道 Bundle 列表和 product ID |
| 4 | 1 | Equinox 从 `plugins/` 安装 Bundle |
| 5 | 1 | 解析 `Import-Package` / `Require-Bundle` / BREE（Bundle Required Execution Environment） |
| 6 | 1 | 按 start level 启动 Bundle |
| 7 | 1→2 | 根据 application/product ID 启动 Workbench |
| 8 | 2 | e4 读取 `Application.e4xmi`，创建 `MApplication` |
| 9 | 2 | 渲染器创建 `Display`、`Shell`、`Part`、菜单 |
| 10 | 2 | 你的 `@PostConstruct` 执行，SWT/JFace 界面出现 |
| 11 | 3 | 如果产品包含 JDT/PDE，Java 编辑器/插件工具才出现 |

### 7.2 `Run As > Eclipse Application` 发生了什么

```text
你在 IDE 里点 Run
   │
   ├─ 第 3 层：PDE 用 ECJ（Eclipse Compiler for Java，Eclipse Java 编译器）编译你的 Bundle
   ├─ 第 3 层：PDE 生成启动配置
   │
   ├─ 第 0 层：启动一个新的 JVM
   ├─ 第 1 层：Equinox 解析目标平台 + 你的 Bundle
   ├─ 第 2 层：Workbench 创建窗口
   └─ 第 2 层：你的视图/Part 出现
```

所以：

- 第一个 Eclipse = 第 3 层工具在运行；
- 第二个 Eclipse = 第 1/2 层 + 你的插件在运行；
- 两个窗口用的可能是同一套 JDK（Java Development Kit，Java 开发工具包），但**不是同一个 JVM（Java Virtual Machine，Java 虚拟机）**。

### 7.3 Target Platform 和三层的关系

Target Platform 是你“编译/运行所依赖的 Bundle 集合”。

它通常包含：

```text
第 1 层 Bundle（Equinox、Platform Runtime）
第 2 层 Bundle（SWT、JFace、e4 Workbench）
可选的第 3 层 Bundle（只有你需要 JDT/PDE 时才加）
```

正式项目的 `.target` 文件应该明确列出这些来源，例如：

```text
https://download.eclipse.org/releases/2026-06
```

然后只勾选你需要的 feature，而不是“全选”。

### 7.4 Product 和三层的关系

Product 是最终交付物：

```text
你的业务插件（第 1/2 层）
   + 目标平台里的第 1/2 层 Bundle
   + 品牌资源
   + 启动配置
   = Product
```

Product **通常不包含**第 3 层的 JDT/PDE，除非你做的产品本身就是 IDE/开发工具。

### 7.5 依赖方向规则

健康的分层依赖：

```text
第 3 层 → 第 2 层 → 第 1 层 → JVM
```

也就是：

- 第 3 层可以依赖第 2、1 层；
- 第 2 层可以依赖第 1 层；
- 第 1 层不应该依赖第 2/3 层；
- 第 2 层不应该依赖第 3 层。

对你的业务插件来说：

- **可以**依赖 `org.eclipse.swt`、`org.eclipse.jface`、`org.eclipse.e4.*`；
- **谨慎**依赖 `org.eclipse.ui`（3.x 兼容层）；
- **不要**随意依赖 `org.eclipse.jdt.ui`、`org.eclipse.pde.ui`，除非你确实在做 IDE 类工具。

为什么？因为依赖第 3 层会把整个 Java IDE 拖进你的产品，体积、启动、升级都会变糟。

### 7.6 错误定位表

| 现象 | 先查哪一层 | 具体位置 |
|---|---|---|
| 双击没反应 / 启动就崩 | 0 | `eclipse.ini`、`-vm`、JDK 版本、内存 |
| `Bundle cannot be resolved` | 1 | `MANIFEST.MF`、Target Platform、版本 |
| `NoClassDefFoundError` | 1 | `Import-Package`/`Export-Package`、Bundle 状态 |
| `UnsupportedClassVersionError` | 1 | BREE、编译 JDK、运行 JDK |
| 视图不出现 | 2 | `plugin.xml` / `Application.e4xmi` |
| 布局错乱 | 2 | Layout/GridData |
| `Invalid thread access` | 2 | UI（User Interface，用户界面）线程 |
| 资源泄漏/卡顿 | 2 | `dispose()`、Job、UI 阻塞 |
| 新建插件/导出失败 | 3 | PDE、Product/Feature 配置 |
| 命令行构建失败 | 3 | Tycho、Target Platform、toolchains |

## 八、常见误解

| 误解 | 事实 |
|---|---|
| “Eclipse IDE 就是 Eclipse Platform” | IDE 是平台 + 工具插件组成的一个 Product |
| “插件就是浏览器插件” | 这里的插件是 OSGi Bundle |
| “所有 RCP 产品都要带 JDT/PDE” | 通常不需要，只有 IDE 类产品才需要 |
| “Target Platform 就是当前 IDE 安装” | 可以是，但正式项目应使用独立的 `.target` |
| “SWT 就是 Swing” | SWT 用原生控件，Swing 是纯 Java 绘制 |
| “e4 完全取代了 3.x” | e4（Eclipse 4，Eclipse 第 4 代应用平台）是运行时，3.x API 通过兼容层继续存在 |
| “p2 就是 Maven” | p2（Provisioning Platform 2）管 Eclipse 安装/更新，Maven/Tycho 管构建 |
| “Workspace 就是 Product” | Workspace 是开发时容器，Product 是交付物 |
| “依赖越多越省事” | 依赖越多，解析越慢，产品越大，冲突越多 |

## 九、一张总表

| 层 | 关键 Bundle/API | 你写的文件 | 你看到的界面 | 典型错误 |
|---|---|---|---|---|
| 0 JVM/启动器 | `eclipse.exe`、`eclipse.ini`、`config.ini` | `eclipse.ini` | 启动画面、启动失败对话框 | exit code 13、JVM terminated |
| 1 运行时/模块 | `org.eclipse.osgi`、`org.eclipse.core.runtime`、`org.eclipse.equinox.*` | `MANIFEST.MF`、`plugin.xml` | Plug-in Editor、Target Platform | Bundle cannot be resolved、NoClassDefFoundError |
| 2 桌面框架 | `org.eclipse.swt`、`org.eclipse.jface`、`org.eclipse.e4.*`、`org.eclipse.ui` | `Application.e4xmi`、Java 类、CSS | 窗口、视图、菜单、向导 | Invalid thread access、视图不显示 |
| 3 IDE 工具 | `org.eclipse.jdt.*`、`org.eclipse.pde.*`、EGit、m2e | 无（使用工具） | Java 编辑器、PDE、Git、Maven | 导出失败、构建失败 |

## 十、学习建议

1. **先理解第 1 层，再写第 2 层。**
   不知道 Bundle/扩展点，写 UI 只会“照抄能跑，换个版本就废”。

2. **把第 3 层当工具，不要当依赖。**
   你会用 JDT/PDE，但你的产品不应该依赖它们（除非你就是做 IDE）。

3. **Target Platform 越小越好。**
   只放你需要的 Bundle，产品更小、启动更快、冲突更少。

4. **报错先分层。**
   启动前 → 第 0 层；依赖/类找不到 → 第 1 层；界面/线程 → 第 2 层；构建/导出 → 第 3 层。

5. **每学一个概念，问三个问题：**
   - 它属于哪一层？
   - 它依赖谁？
   - 它被谁使用？

## 十一、术语速查

| 缩写/术语 | 英文全称 | 中文翻译 | 所属层/说明 |
|---|---|---|---|
| RCP | Rich Client Platform | 富客户端平台 | 总览；基于 Eclipse 平台构建桌面应用 |
| IDE | Integrated Development Environment | 集成开发环境 | 总览；Eclipse IDE 是一个产品 |
| JVM | Java Virtual Machine | Java 虚拟机 | 0 层；运行 Java 字节码 |
| JRE | Java Runtime Environment | Java 运行时环境 | 0 层；只能运行 Java 程序 |
| JDK | Java Development Kit | Java 开发工具包 | 0 层；含 JRE、编译器、工具 |
| Launcher | —（启动器） | 启动器 | 0 层；读取 ini、启动 JVM 和 Equinox |
| Equinox | Eclipse Equinox | Eclipse 的 OSGi 实现 | 1 层；模块运行时 |
| OSGi | Open Services Gateway initiative | 开放服务网关规范 | 1 层；模块化规范 |
| Bundle | —（OSGi 模块） | OSGi 模块 | 1 层；JAR + MANIFEST |
| JAR | Java Archive | Java 归档文件 | 1 层；Bundle 的载体 |
| BREE | Bundle Required Execution Environment | Bundle 所需执行环境 | 1 层；最低 Java 环境 |
| Extension Registry | —（扩展注册表） | 扩展注册表 | 1 层；读取 `plugin.xml` |
| p2 | Provisioning Platform 2 | 配置/供应平台（第 2 代） | 横切；安装/更新系统 |
| IU | Installable Unit | 可安装单元 | 横切；p2 的安装单位 |
| API | Application Programming Interface | 应用程序编程接口 | 通用；`Export-Package` 暴露的接口 |
| SWT | Standard Widget Toolkit | 标准控件工具包 | 2 层；原生控件库 |
| JFace | —（框架名，不是缩写） | Eclipse UI 框架 | 2 层；SWT 之上的 MVC 框架 |
| MVC | Model-View-Controller | 模型-视图-控制器 | 2 层；JFace Viewer 的核心思想 |
| Workbench | —（工作台） | 工作台 | 2 层；窗口、Perspective、Part 的容器 |
| e4 | Eclipse 4 | Eclipse 第 4 代应用平台 | 2 层；应用模型 + DI + CSS |
| MApplication | Model Application | 模型应用根节点 | 2 层；`M` 前缀 = Model |
| MPart | Model Part | 模型 Part | 2 层；视图/编辑器抽象 |
| IEclipseContext | Interface Eclipse Context | Eclipse 上下文接口 | 2 层；`I` 前缀 = Interface |
| DI | Dependency Injection | 依赖注入 | 2 层；e4 的注入机制 |
| CSS | Cascading Style Sheets | 层叠样式表 | 2 层；e4 主题样式 |
| UI | User Interface | 用户界面 | 通用 |
| JNI | Java Native Interface | Java 本地接口 | 2 层；SWT 调原生控件时使用 |
| GC | Graphics Context | 图形上下文 | 2 层；SWT 绘图；JVM 参数中的 GC 则是 Garbage Collection（垃圾回收） |
| JDT | Java Development Tools | Java 开发工具 | 3 层；ECJ、编辑器、调试器 |
| PDE | Plug-in Development Environment | 插件开发环境 | 3 层；插件开发工具 |
| ECJ | Eclipse Compiler for Java | Eclipse Java 编译器 | 3 层；Eclipse 的 Java 编译器 |
| JDI | Java Debug Interface | Java 调试接口 | 3 层；JDT 调试器的基础 |
| EGit | Eclipse Git Team Provider | Eclipse 的 Git 集成 | 3 层 |
| JGit | Java Git | 纯 Java 实现的 Git 库 | 3 层；EGit 的底层 |
| m2e | Maven Integration for Eclipse | Maven 与 Eclipse 集成 | 3 层 |
| XML | Extensible Markup Language | 可扩展标记语言 | 通用；`plugin.xml` 等 |
| Target Platform | —（目标平台） | 目标平台 | 1/2 层；编译/运行依赖的 Bundle 集合 |
| Feature | —（功能部件） | 功能部件 | 交付；一组插件的可安装集合 |
| Product | —（产品） | 产品 | 交付；带品牌的最终应用 |

## 十二、最后用一句话总结

**第 1 层让 Eclipse “能装、能解析、能启动”；第 2 层让 Eclipse “有窗口、有视图、有交互”；第 3 层让 Eclipse “能开发 Eclipse 插件”。**

你做 RCP 时，主要写第 1/2 层，使用第 3 层作为工具，最后用 Feature/Product/p2 把第 1/2 层交付出去。
