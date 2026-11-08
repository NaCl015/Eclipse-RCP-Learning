# 第一次动手：Hello 插件

> 目标：30–45 分钟内完成“新建插件 → 贡献视图 → 运行第二个 Eclipse → 看到界面 → 用 Plug-in Spy 验证 → 改代码重跑”。
> 前置：Eclipse 2026-06 已启动，工作空间为 `D:\eclipse-workspace\rcp-learn`。
> 这次故意使用 **3.x 风格的 `ViewPart` + `plugin.xml`**，因为它是最短、最直观的“插件 + 扩展点”闭环；e4 应用模型在第 6 周再学。

## 一、开始前 5 分钟设置

### 1.1 确认工作空间

Eclipse 启动后，标题栏应包含：

```text
rcp-learn - Eclipse IDE
```

如果工作空间不对，用 `File > Switch Workspace > Other...` 切到：

```text
D:\eclipse-workspace\rcp-learn
```

### 1.2 设置 UTF-8

`Window > Preferences > General > Workspace`

- Text file encoding：`UTF-8`
- New text file line delimiter：`Windows`

### 1.3 确认 JDK 21

`Window > Preferences > Java > Installed JREs`

- 应看到 `D:\Java21`，并勾选为默认。
- 如果没有：`Add... > Standard VM > Directory...` 选择 `D:\Java21`。

`Java > Installed JREs > Execution Environments`

- `JavaSE-21` → 勾选 JDK 21。

### 1.4 切到插件开发透视图

`Window > Perspective > Open Perspective > Other... > Plug-in Development`

如果菜单里没有 `Plug-in Development`，说明 PDE 没装好；本次安装的 RCP 包自带 PDE，正常应该有。

### 1.5 确认目标平台

`Window > Preferences > Plug-in Development > Target Platform`

- 首次学习可以先用 **Running Platform**（当前 IDE 自身）。
- 勾选 Running Platform，点 `Apply and Close`。
- 正式项目再建 `.target` 文件锁定 2026-06。

## 二、新建 Plug-in Project

### 2.1 打开向导

`File > New > Plug-in Project`

如果菜单里没有，用：

`File > New > Other... > Plug-in Development > Plug-in Project`

### 2.2 第一页：项目名与目标平台

| 字段 | 填写 |
|---|---|
| Project name | `com.example.hello` |
| Use default location | 勾选 |
| Target Platform | `Eclipse 2026-06`（或 Running Platform） |

点 `Next`。

> 如果 Target Platform 下拉里没有 2026-06，先选 `Eclipse` 版本里最接近的，或者回到 1.5 确认 Running Platform 已激活。不要选 “an OSGi framework: standard”，否则模板可能找不到 `org.eclipse.ui`。

### 2.3 第二页：插件内容

| 字段 | 填写 |
|---|---|
| ID | `com.example.hello` |
| Version | `1.0.0.qualifier` |
| Name | `Hello Plugin` |
| Provider | 你的名字或 `example.com` |
| Execution Environment | `JavaSE-21` |

勾选：

- [x] `Generate an activator, a Java class that controls the plug-in's life cycle`
- [x] `This plug-in will make contributions to the UI`

**不要勾选**：

- [ ] `Would you like to create a rich client application?`

点 `Next`。

> 这次我们做的是“给现有 Eclipse IDE 增加一个视图”，不是独立 RCP 产品，所以不勾 RCP Application。

### 2.4 第三页：选择模板

在 Eclipse 2026-06 的 PDE 向导里，这个模板叫：

```text
View contribution using 3.x API
```

旧版教程可能叫 `Plug-in with a view`，是同一个东西。

点 `Next`。

### 2.5 第四页：视图信息

| 字段 | 填写 |
|---|---|
| Java package name | `com.example.hello` |
| View class name | `HelloView` |
| View name | `Hello View` |

其他保持默认，点 `Finish`。

### 2.6 如果提示切换透视图

Eclipse 可能问是否打开 Plug-in Development 透视图，选 `Yes`。

## 三、先看懂生成了什么

在 Package Explorer / Project Explorer 里展开 `com.example.hello`，你应该看到：

```text
com.example.hello/
├─ META-INF/
│  └─ MANIFEST.MF
├─ src/
│  └─ com/example/hello/
│     ├─ Activator.java
│     └─ HelloView.java
├─ plugin.xml
├─ build.properties
└─ .project / .classpath
```

### 3.1 打开 `MANIFEST.MF`

双击 `META-INF/MANIFEST.MF`，看 `Overview` 页和 `Dependencies` 页。

你应该能对应上：

- `Bundle-SymbolicName: com.example.hello`：这个 Bundle 的唯一 ID。
- `Bundle-Version: 1.0.0.qualifier`：版本。
- `Bundle-RequiredExecutionEnvironment: JavaSE-21`：最低 Java 21。
- `Require-Bundle` / `Import-Package`：它依赖平台哪些 Bundle。

这就是第 1 层“运行时/模块层”的东西。

### 3.2 打开 `plugin.xml`

切到 `plugin.xml` 的 `Extensions` 页，或在 XML 里看：

```xml
<extension point="org.eclipse.ui.views">
  <category id="com.example.hello" name="Sample Category"/>
  <view
      id="com.example.hello.HelloView"
      name="Hello View"
      icon="icons/sample.png"
      class="com.example.hello.HelloView"
      category="com.example.hello"
      inject="true"/>
</extension>
```

`inject="true"` 表示这个视图支持 e4 风格的依赖注入（例如下面的 `@Inject IWorkbench`），但视图本身仍然是 3.x 的 `ViewPart`。

这段 XML 的含义：

> 我（`com.example.hello` 这个插件）向 `org.eclipse.ui.views` 这个扩展点贡献了一个视图；用户打开它时，请实例化 `com.example.hello.HelloView`。

这就是第 2 层“桌面框架层”的扩展机制。

### 3.3 打开 `HelloView.java`

Eclipse 2026-06 的模板不是简单的一个 Label，而是生成了一个带 `TableViewer` 的示例视图：

```java
public class HelloView extends ViewPart {
    public static final String ID = "com.example.hello.HelloView";

    @Inject IWorkbench workbench;
    private TableViewer viewer;

    @Override
    public void createPartControl(Composite parent) {
        viewer = new TableViewer(parent, SWT.MULTI | SWT.H_SCROLL | SWT.V_SCROLL);
        viewer.setContentProvider(ArrayContentProvider.getInstance());
        viewer.setInput(new String[] { "One", "Two", "Three" });
        viewer.setLabelProvider(new ViewLabelProvider());

        getSite().setSelectionProvider(viewer);
        makeActions();
        hookContextMenu();
        hookDoubleClickAction();
        contributeToActionBars();
    }
    // ...
}
```

对应关系：

- `ViewPart`：3.x 的视图基类；
- `createPartControl`：平台创建视图时回调你的方法；
- `TableViewer`：JFace 的表格 Viewer（MVC 封装）；
- `viewer.setInput(...)`：给 Viewer 提供数据；
- `Action 1` / `Action 2`：模板生成的工具栏/菜单动作；
- `@Inject IWorkbench`：模板开启了 `inject="true"`，所以可以用少量 e4 注入。

## 四、运行：打开第二个 Eclipse

### 4.1 启动运行时

右键项目 `com.example.hello`：

```text
Run As > Eclipse Application
```

第一次可能提示保存，选 `Save`。

### 4.2 观察

会打开第二个 Eclipse 窗口。这是 **运行时实例**，不是重复启动。

- 第一个 Eclipse：开发用的 IDE，运行着 JDT/PDE（第 3 层）。
- 第二个 Eclipse：加载目标平台 + 你的插件（第 1/2 层），相当于你的应用。

### 4.3 打开你的视图

在第二个 Eclipse 中：

```text
Window > Show View > Other...
```

在过滤框输入：

```text
Hello
```

选择 `Hello View`，点 `Open`。

你应该看到：

```text
One
Two
Three
```

这是一个 `TableViewer` 示例；视图工具栏上还有 `Action 1`、`Action 2` 两个按钮。

## 五、用 Plug-in Spy 验证扩展点

在第二个 Eclipse 中，点击 `Hello View` 的空白处，然后按：

```text
Alt + Shift + F1
```

会弹出一个 `Plug-in Spy` 窗口，重点看：

| 字段 | 预期 |
|---|---|
| Active Part | `Hello View` |
| Active view class | `HelloView` |
| Contributing Plug-in | `com.example.hello (1.0.0.qualifier)` |
| Active view identifier | `com.example.hello.HelloView` |
| Active help context | `com.example.hello.viewer` |

这一步的作用：**证明你看到的界面元素确实来自你的插件。** `Contributing Plug-in` 是 `com.example.hello`，就说明这个视图是由你的 Bundle 贡献的。

想看它对应哪个扩展点，回到开发 IDE，打开 `plugin.xml > Extensions`，会看到 `org.eclipse.ui.views`。

如果快捷键没反应：

- 确认第二个 Eclipse 里 PDE 已加载；
- 或者回到开发 IDE，打开 `plugin.xml > Extensions`，同样能看到这条贡献；
- 也可以 `Window > Show View > Other > PDE > Plug-in Spy`（不同版本位置略有差异）。

## 六、改代码，再运行一次

### 6.1 修改表格数据

打开 `HelloView.java`，找到：

```java
viewer.setInput(new String[] { "One", "Two", "Three" });
```

改成：

```java
viewer.setInput(new String[] { "任务一", "任务二", "任务三" });
```

### 6.2 修改 Action 1 的提示

找到 `makeActions()` 里的：

```java
showMessage("Action 1 executed");
```

改成：

```java
showMessage("Action 1：我的第一个动作");
```

### 6.3 重新运行

先关闭第二个 Eclipse 窗口，再在开发 IDE 里重新：

```text
Run As > Eclipse Application
```

打开 `Hello View`：

- 表格应显示 `任务一`、`任务二`、`任务三`；
- 点击视图工具栏的 `Action 1`，应弹出 `Action 1：我的第一个动作`；
- 双击表格里的 `任务一`，应弹出 `Double-click detected on 任务一`。

> 如果只是想快速看效果，可以用 `Debug As > Eclipse Application`，修改方法体后 Java 热替换可能生效；但第一次建议老老实实重启，避免把“热替换失败”误认为代码问题。

### 6.4 可选挑战

如果你还有兴趣，可以试着：

1. 把 `Action 2` 的提示改成你自己的文字；
2. 在 `viewer.setInput(...)` 里多加几个任务；
3. 找到 `ViewLabelProvider`，看看 `getColumnText` 是怎么把对象变成表格文字的。

这些改动很小，但能让你确认：**Viewer 的数据、显示、动作是分开的**，这就是 JFace 的 MVC 思路。

## 七、今天必须搞懂的 6 个问题

做完后，试着回答：

1. `com.example.hello` 在 OSGi 里叫什么？它的唯一 ID 写在哪？
2. `plugin.xml` 里的 `org.eclipse.ui.views` 是什么？`view` 又是什么？
3. 为什么点 Run 会打开第二个 Eclipse？它和第一个是什么关系？
4. 第二个 Eclipse 里为什么能看到你的视图？平台是怎么找到 `HelloView` 的？
5. `HelloView` 属于第 2 层的哪一部分？`ViewPart` 和 SWT/JFace 是什么关系？
6. 如果你把 `plugin.xml` 里的 `class` 写错，会发生什么？

参考答案：

1. Bundle；`MANIFEST.MF` 的 `Bundle-SymbolicName`。
2. 扩展点；扩展（具体贡献）。
3. 第一个是开发 IDE，第二个是运行时实例，加载目标平台 + 你的插件。
4. 平台读取 `plugin.xml` 的扩展注册信息，按需实例化 `HelloView`。
5. Workbench/3.x 兼容层；`ViewPart` 是视图基类，内部用 SWT 创建控件，常用 JFace 做 Viewer。
6. 视图打开时报 `ClassNotFoundException` / `PartInitException`，或在 `.metadata/.log` 里看到实例化失败。

## 八、常见问题

| 现象 | 原因 | 处理 |
|---|---|---|
| `File > New` 里没有 Plug-in Project | 不在 Plug-in Development 透视图 / PDE 未装 | 切换透视图；确认 RCP 包完整 |
| 向导里没有视图模板 | 没勾 `Create a plug-in using a template` / 没勾 “make contributions to the UI” | 勾选后选择 `View contribution using 3.x API` |
| 项目有红叉，提示 `org.eclipse.ui` 无法解析 | 目标平台没设好 | 设置 Running Platform 或 2026-06 target |
| 运行时 Eclipse 打开但没有视图 | 视图没打开 / 扩展点没注册 | `Window > Show View > Other` 搜索 Hello；检查 `plugin.xml` |
| 视图打开报 `ClassNotFoundException` | `plugin.xml` 的 `class` 写错 / 类没编译 | 检查类名、包名、`bin/` 输出 |
| `Alt+Shift+F1` 没反应 | 运行时没加载 PDE Spy | 用开发 IDE 的 `plugin.xml > Extensions` 验证 |
| 第二个 Eclipse 很慢 | 首次启动要初始化运行时工作空间 | 正常，等一会；不要重复点 Run |
| 修改代码后界面没变 | 运行时实例还在用旧类 | 关闭第二个 Eclipse，重新 Run |
| 关闭第二个 Eclipse 后开发 IDE 也退了 | 误关了开发 IDE 窗口 | 重新启动开发 IDE，工作空间不变 |

## 九、完成清单

- [ ] 成功创建 `com.example.hello` 插件
- [ ] 打开 `MANIFEST.MF`，能指出 Bundle ID、版本、BREE
- [ ] 打开 `plugin.xml`，能指出 `org.eclipse.ui.views` 扩展
- [ ] `Run As > Eclipse Application` 打开第二个 Eclipse
- [ ] 在第二个 Eclipse 中打开 `Hello View`
- [ ] 用 `Alt+Shift+F1` 看到 Contributing Plug-in 是 `com.example.hello`
- [ ] 修改表格数据和 Action 文案并重新运行成功
- [ ] 点击 `Action 1`、双击表格行，能看到对话框
- [ ] 能回答第七节的 6 个问题

## 十、接下来 3 天

### 第 1 天（今天）

完成本文的 Hello 插件。

### 第 2 天：加一个菜单命令

学习并实践：

- `org.eclipse.ui.commands`：定义命令；
- `org.eclipse.ui.handlers`：实现 `@Execute` / `@CanExecute`（3.x 风格是 `AbstractHandler`）；
- `org.eclipse.ui.menus`：把命令挂到主菜单/工具栏；
- `org.eclipse.ui.bindings`：加快捷键。

目标：菜单里出现 `Hello > Say Hello`，点击弹出对话框。

### 第 3 天：加一个首选项页

学习并实践：

- `org.eclipse.ui.preferencePages`：贡献首选项页；
- `PreferencePage` / `FieldEditorPreferencePage`；
- `IEclipsePreferences` 读写配置；
- 重启后配置仍生效。

目标：首选项里能设置“问候语”，Hello 视图读取并显示。

### 第 4 天：SWT 布局练习

- 用 `GridLayout` + `GridData` 做一个“姓名/邮箱/备注”表单；
- 窗口缩放不破版；
- 加一个表格，用 `TableViewer` 显示数据。

然后回到 [03-12周行动计划](03-12周行动计划.md) 的第 2 周，系统学习 OSGi 与 PDE。
