# Eclipse RCP 开发学习工作区

> 建立日期：2026-11-08
> 本机环境：Windows 11 + Eclipse 2026-06 + JDK 21（保留 JDK 17）

## 1. 本机环境（本次安装后）

| 项目 | 现状 | 结论 |
|---|---|---|
| 操作系统 | Windows 11 家庭中文版 25H2（10.0.26200） | 可用 |
| 内存 | 15.6 GB | 够用 |
| 磁盘 | C: 143 GB 空闲 / D: 107 GB 空闲 | 够用 |
| JDK 21 | Temurin 21.0.13 LTS，`D:\Java21` | 已安装 |
| JDK 17 | 17.0.10，`D:\Java17\jdk-17.0.10` | 保留兼容 |
| Eclipse | RCP/RAP 2026-06（4.40），`D:\eclipse\rcp-2026-06` | 已安装 |
| Maven | 3.9.8（`D:\Maven\apache-maven-3.9.8`） | 可用 |
| Git | 2.45.2 | 可用 |
| 工作空间 | `D:\eclipse-workspace\rcp-learn` | 已创建 |

## 2. 推荐技术栈

- IDE：Eclipse IDE for RCP and RAP Developers **2026-06**（4.40）
- JDK：**Temurin 21 LTS**（`D:\Java21`）
- 应用目标级别：**Java 21**
- 构建：Maven 3.9.8 + Tycho（pomless）
- 版本控制：Git
- 测试：JUnit 5 + SWTBot
- 打包：Feature + Product + p2 + Tycho

> 为什么不用最新的 2026-09？因为 2026-09 的 IDE 安装包要求 Java 25。既然确定使用 Java 21，就选择 2026-06（4.40），它要求 Java 21，功能也足够新。

## 3. 文档索引

0. [00-概念入门](docs/00-概念入门.md)：Eclipse/RCP/OSGi/插件/目标平台/JDK/Tycho 等概念，建议动手前先读。
1. [01-学习路线图](docs/01-学习路线图.md)：能力模型、7 个阶段、每阶段知识点/练习/验收标准、资料清单。
2. [02-软件安装与配置清单](docs/02-软件安装与配置清单.md)：软件清单、版本选择、配置说明、常见问题。
3. [03-12周行动计划](docs/03-12周行动计划.md)：按周拆解的任务表、Capstone 项目需求、每周节奏。
4. [04-本次安装手册](docs/04-本次安装手册.md)：本次实际安装的完整步骤、命令、验证结果与回滚方法。
5. [05-Eclipse三层架构详解](docs/05-Eclipse三层架构详解.md)：运行时/模块层、桌面框架层、IDE 工具层，以及它们如何协作。
6. [06-第一次动手-Hello插件](docs/06-第一次动手-Hello插件.md)：新建插件、贡献视图、运行第二个 Eclipse、用 Plug-in Spy 验证。

## 4. 启动 Eclipse

双击桌面快捷方式，或运行：

```text
D:\tools\start-eclipse-rcp.cmd
```

等价于：

```text
D:\eclipse\rcp-2026-06\eclipse.exe -data D:\eclipse-workspace\rcp-learn
```

## 5. 已验证的命令

```powershell
D:\Java21\bin\java -version     # 21.0.13
D:\Java21\bin\javac -version    # 21.0.13
D:\tools\mvn21.cmd -v           # Maven 3.9.8 + Java 21
```

## 6. 学习原则

- 20% 看资料，80% 动手写代码。
- 每个阶段必须有一个“能运行、能演示”的产物。
- 每完成一个阶段打一个 Git tag（`stage-1`、`stage-2`…）。
- 先学纯 e4，再回头看 3.x 兼容层；企业老项目大量使用 3.x API。
- 遇到问题先看 `workspace/.metadata/.log` 和 PDE Error Log。
- 不要盲目追最新版：目标平台要与公司/产品要求一致。
