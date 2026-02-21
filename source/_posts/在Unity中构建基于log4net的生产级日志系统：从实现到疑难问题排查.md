---
title: 在 Unity 中构建基于 log4net 的生产级日志系统：从实现到疑难问题排查
tags: Unity
categories: 技术分享
abbrlink: 878304ff
date: 2025-07-09 18:40:38
recommend: true
cover: https://image.nfasystem.top/img-nfa/Unity_log4.webp
---
在游戏开发中，一个健壮、可配置、可持久化的日志系统是不可或令的。它不仅是开发阶段调试的利器，更是项目上线后追踪线上问题、分析用户行为的关键依据。Unity 自带的 `Debug.Log` 系统虽然便捷，但其功能单一，无法满足日志分级、持久化存储、灵活配置等生产环境的需求。

本文将详细介绍如何在 Unity 2023 中，利用强大的第三方日志框架 `log4net`，从零开始构建一个生产级别的日志系统。内容将覆盖系统的设计与实现、具体使用方法，并深入剖析一个在集成过程中可能遇到的、极具迷惑性的“日志不生效”问题，提供完整的排查思路与解决方案。

## **系统设计目标**

我们旨在构建一个满足以下需求的日志系统：

*   **日志持久化**：将日志信息保存到本地文件，便于离线分析。
*   **日志分级**：支持 Info, Warning, Error 等不同级别的日志，方便筛选关键信息。
*   **统一管理**：能够捕获并统一处理 Unity 引擎自身（如 `Debug.Log`）产生的日志。
*   **高性能**：在 Release 版本中，可以轻松剥离不必要的日志输出，减少性能开销。
*   **易于使用**：通过静态类提供简洁的 API，在项目任何位置都能方便地调用。

## **核心组件：log4net**

`log4net` 是 Apache 基金会旗下的一个成熟、功能强大的日志框架。它的核心优势在于其高度的可配置性，通过一个简单的 XML 文件，就可以定义日志的输出格式、目标（文件、控制台、数据库等）以及过滤规则。

## **系统实现步骤**

### **步骤 3.1：集成 log4net**

1.  **获取 DLL**：从 `log4net` 官方网站下载二进制包，解压后找到适用于 .NET Standard 2.x 或者 .NET 4.6.2 的 `log4net.dll` 文件。
2.  **导入 Unity**：在 Unity 项目的 `Assets` 目录下创建 `Plugins` 文件夹，并将 `log4net.dll` 拖入其中。

**[点击下载 log4net ](/downloads/apache-log4net-binaries-3.1.0.zip)**

### **步骤 3.2：创建 log4net 配置文件**

`log4net` 的行为由配置文件驱动。在 `Assets/StreamingAssets/` 目录下创建一个名为 `log4net.config` 的文件。`StreamingAssets` 目录能确保该文件在打包后被原样保留。

一个推荐的基础配置如下：

```xml
<?xml version="1.0" encoding="utf-8" ?>
<log4net>
  <!-- 定义一个滚动文件输出器 (Appender) -->
  <appender name="RollingFileAppender" type="log4net.Appender.RollingFileAppender">
    <!-- 
      文件路径使用 PatternString，允许动态替换 %property{LogPath} 变量。
      这个变量将在代码中被设置为 Application.persistentDataPath。
    -->
    <file type="log4net.Util.PatternString" value="%property{LogPath}/gamelog.log" />
    <appendToFile value="true" />
    <rollingStyle value="Size" />
    <maxSizeRollBackups value="5" />
    <maximumFileSize value="5MB" />
    <staticLogFileName value="true" />
    <layout type="log4net.Layout.PatternLayout">
      <conversionPattern value="%date [%thread] %-5level %logger - %message%newline" />
    </layout>
  </appender>

  <!-- 根 Logger 配置 -->
  <root>
    <!-- 设置日志记录的最低级别为 DEBUG -->
    <level value="DEBUG" />
    <!-- 将上面定义的 RollingFileAppender 附加到根 Logger -->
    <appender-ref ref="RollingFileAppender" />
  </root>
</log4net>
```

**注意**：创建此文件时，务必确保其文本编码为 **UTF-8 (无BOM)**，否则可能导致 `log4net` 解析失败且不抛出任何错误。

### **步骤 3.3：编写日志 API 封装类 (`GameLogger.cs`)**

为了避免与 `UnityEngine.Log` 产生命名冲突，并提供一个简洁的接口，我们创建一个静态工具类。为了代码的健壮性和可维护性，将其置于自定义的命名空间下。

```csharp
// GameLogger.cs
using log4net;
using UnityEngine;

namespace YourGame.Core.Logging
{
    public static class GameLogger
    {
        private static ILog log;

        public static void Init(string loggerName)
        {
            log = LogManager.GetLogger(loggerName);
        }

        // 在编辑器模式下，将日志消息转发到 Unity 控制台
        [System.Diagnostics.Conditional("UNITY_EDITOR")]
        private static void ForwardToUnityConsole(object message, LogType type)
        {
            switch (type)
            {
                case LogType.Log: Debug.Log(message); break;
                case LogType.Warning: Debug.LogWarning(message); break;
                case LogType.Error: Debug.LogError(message); break;
            }
        }

        public static void Info(object message, bool forwardToConsole = true)
        {
            log?.Info(message);
            if (forwardToConsole) ForwardToUnityConsole(message, LogType.Log);
        }

        public static void Warn(object message, bool forwardToConsole = true)
        {
            log?.Warn(message);
            if (forwardToConsole) ForwardToUnityConsole(message, LogType.Warning);
        }

        public static void Error(object message, bool forwardToConsole = true)
        {
            log?.Error(message);
            if (forwardToConsole) ForwardToUnityConsole(message, LogType.Error);
        }
    }
}
```
此设计通过一个可选参数 `forwardToConsole` 和条件编译特性 `[Conditional("UNITY_EDITOR")]`，实现了在编辑器模式下将日志同时输出到控制台和文件，而在发布版本中仅写入文件，兼顾了开发便利性与发布性能。

### **步骤 3.4：创建 Unity 日志桥接器 (`UnityLogBridge.cs`)**

为了捕获 `Debug.Log` 等原生日志，需要监听 `Application.logMessageReceivedThreaded` 事件。

```csharp
// UnityLogBridge.cs
using UnityEngine;

namespace YourGame.Core.Logging
{
    public static class UnityLogBridge
    {
        public static void Init()
        {
            Application.logMessageReceivedThreaded += OnLogMessageReceived;
            GameLogger.Info("UnityLogBridge Inited. Start listening to Unity's log messages.");
        }

        private static void OnLogMessageReceived(string condition, string stackTrace, LogType type)
        {
            // 拼接消息和堆栈信息
            string message = $"{condition}\n{stackTrace}";

            // 调用 GameLogger 时，将 forwardToConsole 设为 false，避免无限循环
            switch (type)
            {
                case LogType.Error:
                case LogType.Exception:
                    GameLogger.Error(message, forwardToConsole: false);
                    break;
                case LogType.Warning:
                    GameLogger.Warn(message, forwardToConsole: false);
                    break;
                default:
                    GameLogger.Info(message, forwardToConsole: false);
                    break;
            }
        }
    }
}
```
此设计的关键在于，从桥接器转发日志时，必须将 `forwardToConsole` 参数设为 `false`，以切断 `GameLogger` -> `Debug.Log` -> `UnityLogBridge` -> `GameLogger` 的无限递归调用链。

### **步骤 3.5：初始化系统**

在游戏启动时，如在一个 `GameManager` 脚本的 `Awake` 方法中，执行初始化序列。

```csharp
// GameManager.cs
using UnityEngine;
using System.IO;
using YourGame.Core.Logging; // 引入自定义的命名空间

public class GameManager : MonoBehaviour
{
    void Awake()
    {
        // 1. 设置 log4net 的动态属性，指定日志文件保存路径
        log4net.GlobalContext.Properties["LogPath"] = Application.persistentDataPath;

        // 2. 加载 log4net 配置文件
        string configFilePath = Path.Combine(Application.streamingAssetsPath, "log4net.config");
        log4net.Config.XmlConfigurator.Configure(new FileInfo(configFilePath));

        // 3. 初始化日志 API 类
        GameLogger.Init("Unity");

        // 4. 初始化 Unity 日志桥接器
        UnityLogBridge.Init();

        // --- 系统初始化完成，可以开始使用 ---
        GameLogger.Info("Log system initialized successfully!");
    }
}
```

## **使用方法**

在项目中的任何脚本里，只需引入命名空间，即可调用：

```csharp
using YourGame.Core.Logging;

public class MyFeature : MonoBehaviour
{
    void Start()
    {
        GameLogger.Info("MyFeature has started.");
        GameLogger.Warn("Player health is low.");
      
        try
        {
            // ... some code that might fail ...
        }
        catch (System.Exception ex)
        {
            GameLogger.Error($"An exception occurred: {ex.Message}");
        }
    }
}
```

## **疑难问题排查：日志系统“静默失败”**

在集成过程中，可能会遇到一个非常棘手的问题：调用 `GameLogger.Info()` 后，**既没有在控制台看到输出，也没有在目标路径生成日志文件**，但程序不抛出任何异常。

### **问题现象分析**

*   **自定义日志无响应**：`GameLogger.Info()` 等调用无效。
*   **原生日志正常**：`Debug.Log()` 正常在控制台输出。
*   **调试器行为迷惑**：使用调试器“步入”`GameLogger.Info()` 方法，可以正常进入，但方法内的 `log.Info()` 调用似乎被跳过。
*   **IDE 警告**：IDE (如 Rider) 可能会在调用点提示 `'GameLogger' is a type, which is not valid in the given context.`，即使已经改名避免了与 `UnityEngine.Log` 的直接冲突。

### **根本原因定位**

经过层层排查，最终定位到问题根源：**`log4net` 的 XML 配置文件未能被正确解析，导致 Appender (输出器) 未能被创建并附加到 Logger 对象上。**

`log4net` 在解析配置文件时，如果遇到无法识别的文件编码（如带BOM的UTF-8）或文件开头的隐藏字符，它会选择**静默失败 (Silently Fail)**。这意味着它不会抛出异常，也不会给出任何警告，只是简单地停止解析。

这就解释了所有现象：
*   `log4net.Config.XmlConfigurator.Configure()` 方法执行了，但内部什么都没做。
*   `LogManager.GetLogger()` 返回了一个有效的 Logger 对象，但这个对象没有任何 Appender 关联。
*   调用 `log.Info()` 时，因为没有 Appender，所以日志消息无处可去，被直接丢弃。

### **解决方案与诊断流程**

解决此问题的关键是确保 `log4net.config` 文件是“纯净”的。

1.  **确认文件编码**：删除现有配置文件，使用专业的文本编辑器（如 VS Code, Notepad++）重新创建，并在保存时**明确选择 `UTF-8 (无BOM)` 编码**。
2.  **复制标准内容**：直接从可靠来源（如本文档）复制配置内容，避免从网页等处复制时引入不可见字符。
3.  **开启内部调试**：在 `log4net.config` 的根节点 `<log4net>` 中添加 `debug="true"` 属性。如果 `log4net` 能够开始解析，它会将其内部的初始化步骤、警告和错误输出到Unity控制台，为诊断提供关键线索。
4.  **代码级诊断**：如果问题依旧，可以通过代码直接检查 Logger 的状态，确认 Appender 是否被成功附加。

    ```csharp
    // 在 GameLogger.Init() 中添加诊断代码
    var logger = log.Logger as log4net.Repository.Hierarchy.Logger;
    if (logger != null)
    {
        Debug.Log($"Logger Appender count: {logger.Appenders.Count}");
        if (logger.Appenders.Count == 0)
        {
            Debug.LogError("No Appenders attached! Check log4net.config file integrity and encoding.");
        }
    }
    ```
    当看到 `Appender count: 0` 的输出时，即可确信问题出在配置文件的解析上。

### **总结**

通过结合 `log4net` 和自定义的封装层，我们可以在 Unity 中构建一个功能完善、易于扩展的日志系统。它不仅解决了 Unity 原生日志系统的局限性，还通过精巧的设计兼顾了开发效率和发布性能。理解并掌握像“静默失败”这类疑难问题的排查方法，是提升项目稳定性和开发能力的重要一环。一个健壮的日志系统，是保障大型、长期项目成功的基石。

### 由这个问题引发的思考在这里面的第三条{% post_link '各类杂碎知识点' '零碎知识点' %}