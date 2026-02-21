---
title: 从 Rider 的一条建议谈起：深入理解 C# 表达式体成员
tags: Unity
categories: 技术分享
abbrlink: ec87c3ca
date: 2025-07-10 21:41:52
cover: https://image.nfasystem.top/img-nfa/Unity_CSharp_member1.webp
---

## 问题引入

在日常使用 Rider 编辑器进行 Unity 开发时，我们可能会在编写属性时遇到这样一条建议：“代码主体不符合代码样式设置: 请使用表达式体 (Code body does not match code style settings: Use expression body)”。

例如，在下面这段用于控制游戏音乐开关的属性代码中，Rider 会在 `get` 访问器的 `return` 语句处给出该提示：

```csharp
public static bool MusicOn
{
    get
    {
        return PlayerPrefs.GetInt("Music", 1) == 1;
    }
    set
    {
        PlayerPrefs.SetInt("Music", value ? 1 : 0);
        SoundManager.Instance.MusicOn = value;
    }
}
```

这个建议是什么意思？它背后又涉及到了哪些 C# 的知识点？这正是我们本文将要深入探讨的主题。

## 什么是表达式体成员？

Rider 的这条建议，实际上是在引导我们使用 C# 6.0 版本引入的一个特性：**表达式体成员 (Expression-bodied Members)**。

我们可以将这个特性理解为一种“语法糖”。所谓“语法糖”，并不是什么复杂的魔法，而是一种更简洁、更优雅的语法，它能让我们用更少的代码实现同样的功能，从而提高代码的可读性和编写效率。

### 核心思想：用更少的话，说清楚一件事

想象一个生活中的场景，当朋友问我们“现在几点了？”，我们通常会直接回答“三点半”，而不是说“我将要告诉你现在的时间，现在的时间是三点半。”

表达式体成员正是基于同样直截了当的思想。对于那些逻辑非常简单、只包含**单一操作**的类成员（如方法或属性），我们无需编写完整的 `{}` 代码块，而是可以直接通过 `=>` 符号来表达其实现。

这里的 `=>` 符号，我们通常不读作 Lambda 表达式中的 "Lambda"，而是理解为 "goes to" (指向)。它的含义是：**“这个成员的实现，就是紧跟在它后面的这个表达式”。**

我们回头看最初的例子：

**传统代码块主体 (Block Body):**

```csharp
get
{
    return PlayerPrefs.GetInt("Music", 1) == 1;
}
```

**表达式体 (Expression Body):**

```csharp
get => PlayerPrefs.GetInt("Music", 1) == 1;
```

通过 `=>`，我们将三行代码缩减为一行，代码的意图一目了然：`get` 的结果就是 `PlayerPrefs.GetInt("Music", 1) == 1` 这个表达式的计算结果。

## 表达式体成员的演进

这个特性并非一开始就支持所有场景，了解它的发展历程有助于我们理解在不同版本的 C# 代码中看到的不同用法。

*   **C# 6.0**: 首次引入，支持**方法 (Method)** 和**只读属性 (Read-only Property)**。
*   **C# 7.0**: 功能得到极大扩展，开始支持**构造函数 (Constructor)**、**终结器 (Finalizer)**、**属性的 `get` 和 `set` 访问器**以及**索引器 (Indexer)**。

对于 Unity 开发者而言，随着 Unity 版本的迭代，其内置的 C# 编译器版本也在升级，使得我们现在可以广泛地使用这些现代 C# 特性。

## 表达式体成员的实际应用

下面我们通过具体的代码示例，来对比传统写法和表达式体写法在不同场景下的应用。

### 方法 (Methods)

当一个方法只包含一条 `return` 语句，或仅执行一个操作时。

*   **传统写法:**
    ```csharp
    public float GetCircleArea(float radius)
    {
        return 3.14159f * radius * radius;
    }
    ```
*   **表达式体写法:**
    ```csharp
    public float GetCircleArea(float radius) => 3.14159f * radius * radius;
    ```

### 属性 (Properties)

这是我们最开始遇到的场景。

*   **只读属性**
    *   **传统写法:**
        ```csharp
        public string PlayerStatus
        {
            get { return _isAlive ? "Alive" : "Defeated"; }
        }
        private bool _isAlive = true;
        ```
    *   **表达式体写法:**
        ```csharp
        public string PlayerStatus => _isAlive ? "Alive" : "Defeated";
        private bool _isAlive = true;
        ```

*   **可读可写属性的 `get` 和 `set` 访问器**
    *   **传统写法:**
        ```csharp
        private float _health;
        public float Health
        {
            get { return _health; }
            set { _health = Mathf.Clamp(value, 0, 100); }
        }
        ```
    *   **表达式体写法:**
        ```csharp
        private float _health;
        public float Health
        {
            get => _health;
            set => _health = Mathf.Clamp(value, 0, 100);
        }
        ```

### 构造函数 (Constructors)

当构造函数只执行一个简单的赋值操作时。

*   **传统写法:**
    ```csharp
    public class Player
    {
        private string _name;
        public Player(string name)
        {
            this._name = name;
        }
    }
    ```
*   **表达式体写法:**
    ```csharp
    public class Player
    {
        private string _name;
        public Player(string name) => this._name = name;
    }
    ```

### 索引器 (Indexers)

当索引器的 `get` 和 `set` 访问器逻辑足够简单时。

*   **传统写法:**
    ```csharp
    public class Team
    {
        private Player[] _players = new Player[5];
        public Player this[int index]
        {
            get { return _players[index]; }
            set { _players[index] = value; }
        }
    }
    ```
*   **表达式体写法:**
    ```csharp
    public class Team
    {
        private Player[] _players = new Player[5];
        public Player this[int index]
        {
            get => _players[index];
            set => _players[index] = value;
        }
    }
    ```

## 规则与限制：何时不应使用

理解一个特性的边界和限制，与掌握其用法同等重要。

1.  **必须是单一表达式**
    这是最核心的规则。`=>` 之后必须是且仅是一个表达式，不能是多个用分号隔开的语句。我们最初的例子中，`set` 访问器包含两条语句，因此它不能被简化为表达式体。

2.  **不能包含复杂的流程控制语句**
    诸如 `if-else` (三元运算符 `?:` 是例外)、`for`、`foreach`、`while`、`switch` 等语句块不能直接用于表达式体。

    *   **错误示范:**
        ```csharp
        // 编译错误！不能在表达式体中使用 if 语句块
        public string GetWeather(float temp) => if (temp > 30) return "Hot"; else return "Cool";
        ```
    *   **正确替代 (使用三元运算符):**
        ```csharp
        public string GetWeather(float temp) => temp > 30 ? "Hot" : "Cool";
        ```

## 结论与建议

表达式体成员是 C# 提供的一个强大的代码简化工具。它能让我们的代码更加紧凑，意图更加清晰。

通过理解并恰当运用表达式体成员，我们可以编写出更具现代感、更优雅、更易于维护的 C# 代码。