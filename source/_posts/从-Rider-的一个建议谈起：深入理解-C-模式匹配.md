---
title: 从 Rider 的一个建议谈起：深入理解 C# 模式匹配
tags: Unity
categories: 技术分享
abbrlink: fc4aea50
date: 2025-07-10 21:01:10
cover: https://image.nfasystem.top/img-nfa/Unity_CSharp_Match.webp
---

## 问题的引入

在日常的 Unity 开发中，我们经常会与各种集成开发环境（IDE）打交道，如 Rider 或 Visual Studio。这些强大的工具不仅能帮助我们编写代码，其智能提示功能有时更能成为我们学习新知识的契机。

最近，我在在编写一段处理对话节点的逻辑时，我写下了这样一段非常普遍的代码：

```csharp
// nodeToDisplay 是一个包含对话选项的对象
// choices 是一个字符串数组
if (nodeToDisplay.choices != null && nodeToDisplay.choices.Length > 0)
{
    // 如果存在选项，则显示选项按钮...
}
```

这段代码的意图非常明确：为了安全地访问数组的 `Length` 属性，我们首先检查 `choices` 数组本身是否为 `null`，然后再检查其长度是否大于零。这是一个健壮且完全正确的写法。

然而，Rider 编辑器在 `&&` 符号下给出了一条建议：“合并到模式中”（Merge into pattern）。这不禁让我思考：

*   这个建议是什么意思？
*   它推荐的写法是怎样的，又有什么优势？
*   其背后蕴含了 C# 语言的何种特性？

这便是本文将要深入探讨的核心：**C# 模式匹配（Pattern Matching）**。

## 从“指令”到“画像”：模式匹配的核心思想

在深入了解具体的语法之前，我们首先需要理解模式匹配的哲学思想。我们可以用一个门卫的例子来做类比。

*   **传统的过程式指令**：我们给门卫下达一系列按部就班的指令。“第一步，检查访客是否持有邀请函（`!= null`）。第二步，如果持有，打开邀请函检查其身份是否为 VIP（`.Status == "VIP"`）。第三步，如果身份是 VIP，再检查其姓名是否在白名单上...”。这一系列指令必须严格按顺序执行，逻辑链条清晰，但略显繁琐。

*   **现代的声明式“画像”**：我们直接给门卫一张目标人物的“画像”。“放一个**持有 VIP 邀请函、且姓名在白名单上的人**进来”。我们不再描述“如何检查”，而是直接描述“目标长什么样”。

C# 的模式匹配，正是这样一种让我们能够用声明式“画像”来检查数据“形状”和“值”的强大工具。它让代码的意图变得更加直观。

## 模式匹配的语法结构

要熟练运用模式匹配，我们首先需要掌握其基本的语法构建块。模式匹配主要通过 `is` 关键字（在 `if` 语句中）和 `switch` 语句的 `case` 标签来实现。

一个模式匹配表达式的基本形式是：` <待检查的变量> is <模式> `

下面是构成 `<模式>` 部分的核心组件：

1.  **常量模式 (Constant Pattern)**
    *   **语法**: 直接使用一个常量值。
    *   **示例**: `is null`, `is 10`, `is "Completed"`
    *   **用途**: 判断变量是否等于某个具体的常量值。

2.  **类型模式 (Type Pattern)**
    *   **语法**: `<类型> <新变量名>`
    *   **示例**: `is Player player`, `is Rigidbody rb`
    *   **用途**: 判断变量是否为指定类型。如果判断成功，会自动将变量转换为该类型并赋值给新的局部变量（如 `player` 或 `rb`），该变量仅在 `if` 语句块内有效。

3.  **属性模式 (Property Pattern)**
    *   **语法**: `{ 属性1: <模式1>, 属性2: <模式2> }`
    *   **示例**: `{ Health: > 0, Name: "Goblin" }`
    *   **用途**: 检查对象的一个或多个公共属性。它使用 `{}` 包裹，内部可以包含多个“属性: 模式”对，用逗号分隔。重要的是，属性后面跟的也是一个模式（可以是常量模式、关系模式等）。

4.  **关系模式 (Relational Pattern)**
    *   **语法**: `>`、`<`、`>=`、`<=`
    *   **示例**: `> 10`, `<= 0`
    *   **用途**: 用于和数值进行比较，通常嵌套在属性模式中使用。

5.  **逻辑模式 (Logical Patterns)**
    *   **语法**: `and`、`or`、`not`
    *   **示例**: `> 0 and < 100`, `not null`
    *   **用途**: 用于组合多个模式，形成更复杂的逻辑判断。

6.  **`when` 子句 (附加条件)**
    *   **语法**: `case <模式> when <布尔表达式>:`
    *   **用途**: 这是 `switch` 语句中的一个强大补充。当仅靠模式无法完全表达判断条件时（例如需要调用一个方法），可以使用 `when` 关键字来添加一个额外的布尔表达式作为判断条件。

在掌握了这些基础语法之后，我们再通过具体的案例来加深理解。

---

## 模式匹配的“兵器谱”：实战案例

### 类型模式 (Type Pattern)

*   **场景**：在 `OnTriggerEnter` 中判断碰撞到的物体是否为“玩家”。
*   **语法应用**：`other.GetComponent<Player>() is Player player`
*   **代码示例**：
    ```csharp
    void OnTriggerEnter(Collider other)
    {
        // "如果 GetComponent 的结果是 Player 类型，则将其赋值给新变量 player"
        if (other.GetComponent<Player>() is Player player)
        {
            player.TakeDamage(10);
        }
    }
    ```
    这里，我们使用了类型模式，将类型检查、`null` 检查、类型转换和变量赋值四个步骤一气呵成。

### 属性模式 (Property Pattern)

*   **场景**：我们希望攻击一个敌人，但前提是该敌人必须存活（`Health > 0`）且并非处于无敌状态（`isInvincible == false`）。
*   **语法应用**：`enemy is { Health: > 0, isInvincible: false }`
*   **代码示例**：
    ```csharp
    void TryAttack(Enemy enemy)
    {
        // 这里组合了属性模式、关系模式(> 0)和常量模式(false)
        if (enemy is { Health: > 0, isInvincible: false })
        {
            enemy.TakeDamage(25);
        }
    }
    ```
    `{...}` 构成的属性模式就像一个清晰的模板，直观地描述了我们所期望的数据状态。它自带的 `null` 检查保证了代码的安全性。

### 逻辑模式 (Logical Patterns)

*   **场景**：一个玩家角色只有在生命值满格（`Health >= 100`）或者法力值大于 50（`Mana > 50`）时，才能释放某个终极技能。
*   **语法应用**：`player is { Health: >= 100 } or { Mana: > 50 }`
*   **代码示例**：
    ```csharp
    if (player is { Health: >= 100 } or { Mana: > 50 })
    {
        // Cast ultimate skill
    }
    ```
    `or` 关键字让这个组合条件的意图变得非常清晰，比传统的 `||` 连接的多个表达式更易于阅读。

---

## `switch` 表达式：模式匹配的终极舞台

如果说 `if` 语句是模式匹配的“训练场”，那么 `switch` 语句和 `switch` 表达式就是其威力的“终极舞台”。

*   **场景**：重构一个复杂的 `OnTriggerEnter` 方法，根据碰撞到的不同对象执行不同逻辑。
*   **代码示例**：
    ```csharp
    void OnTriggerEnter(Collider other)
    {
        switch (other.gameObject)
        {
            // 类型模式 + when 子句
            case GameObject g when g.CompareTag("Player"):
                Debug.Log("碰到玩家了！");
                break;

            // 属性模式 + when 子句 + 类型模式
            case { tag: "Enemy" } when other.GetComponent<Enemy>() is { Health: > 0 } enemy:
                enemy.TakeDamage(99);
                break;

            // 属性模式，并把 layer 值赋给新变量 l
            case { layer: int l } when l == LayerMask.NameToLayer("Collectable"):
                Destroy(other.gameObject);
                break;
          
            // 常量模式，优雅地处理 null
            case null:
                Debug.LogWarning("碰到的物体已被销毁！");
                break;

            // 默认情况
            default:
                Debug.Log("碰到了其他东西。");
                break;
        }
    }
    ```
    这个 `switch` 结构清晰地罗列了所有可能的情况，将条件判断与逻辑执行解耦，每个 `case` 分支都是一个独立的“画像”，使得整个逻辑一目了然。

---

## 结论：为何我们应当拥抱模式匹配

通过以上的分析，我们可以总结出在项目中使用模式匹配的几点核心优势：

1.  **简洁性 (Conciseness)**：用更少的代码表达更复杂的逻辑，告别冗长的 `if-null-then-check-property` 链条。
2.  **可读性 (Readability)**：代码即文档。模式匹配的声明式语法使其意图更加清晰，更接近自然语言描述。
3.  **安全性 (Safety)**：内置的 `null` 检查机制能从源头上规避大量潜在的 `NullReferenceException`，这对于所有开发者来说，都是一个巨大的福音。
4.  **表现力 (Expressiveness)**：尤其是在 `switch` 语句中，模式匹配允许我们构建出以往用 `if-else` 难以优雅实现的复杂、扁平的逻辑分支结构。

C# 模式匹配是现代 C# 语言赠予开发者的一把利器。熟练掌握它，将使我们的代码在健壮性、可读性和维护性上都迈上一个新的台阶。

