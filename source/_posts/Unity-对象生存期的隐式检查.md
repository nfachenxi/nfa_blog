---
title: Unity 对象生存期的隐式检查
tags: 
- Unity
- Fire
categories: 技术分享
abbrlink: a266d19c
date: 2025-07-10 14:18:03
cover: https://image.nfasystem.top/img-nfa/Unity%20%E5%AF%B9%E8%B1%A1%E7%94%9F%E5%AD%98%E6%9C%9F%E7%9A%84%E9%9A%90%E5%BC%8F%E6%A3%80%E6%9F%A5.webp
---
## 问题引入

今天在写代码的时候偶然注意到了Rider编辑器给我弹出的一个提示啊，也不算是警告，就是这个 Unity 对象生存期的隐式检查

![alt text](https://image.nfasystem.top/img/Unity_Rider_2.png)

![alt text](https://image.nfasystem.top/img/Unity_Rider_1.png)

处于好奇啊，我就简单去了解了一下，结果没想到啊，这个东西涉及到了 Unity 的底层工作方式了，那我就边写边学了。

---

## C#中的 `null`是什么

在标准的C#程序中，一个变量如果等于 `null`，意味着它不指向任何内存地址。它就是“空”的，如果尝试访问它的任何成员（比如方法、属性）都会立即抛出 `NullReferenceException` 异常，这非常直接。

```C#
// 这是一个普通的C#类
public class MyNormalClass {
    public void DoSomething() { }
}

MyNormalClass myInstance = null;
// 下面这行代码会立即抛出 NullReferenceException
myInstance.DoSomething(); 
```
---

## Unity 对象的特殊性：C#层与C++层的双重存在

Unity引擎的核心是用C++编写的，而我们写的逻辑代码为C#，场景中每一个 `GameObject` 或附加在上面的组件（如 `Transform` , `Rigidbody` , `MonoBehaviour` 脚本）在内存中都有两个部分：
- **C#包装器对象（Managed Object）：** 这是在C#代码中能直接访问和操作的对象。
- **C++引擎对象（Native Object）：** 这是真正在引擎底层负责渲染、物理计算等的实际对象。

C#对象持有一个指向对应C++对象的指针。

---

## `Destory()` 的工作机制

当我们调用 `Destory(myGameObject);` 时，会发生一件有趣的事情：

- **它不是立即销毁的：** Unity不会，马上从内存中清除这个对象，它只是把对应的C++引擎对象标记为了“已销毁”。
- **延迟销毁：** 真正的内存清理工作会延迟到当前帧的末尾（在所有Update之后，渲染之前）进行。

这就产生了一个中间状态：在调用 `Destory()` 之后但在帧末清理之前，**C#包装器对象依然存在，但它指向的C++引擎对象已经“死亡”了。**

如果这个时候我们用标准的C# `null` 检查，它会告诉你C#对象还存在（因为它确实还在内存里），这就会导致问题。如果我们尝试访问一个已经“死亡”的对象，可能会引发一个更难理解的错误。

---

## Unity的解决方案：重载 == 操作符

为了解决这个问题，Unity做了一个非常聪明的设计：它为所有继承自 `UnityEngine.Object` 的类（基本上就是在Unity中用到了所有东西）**重载（overloaded）了 `==` 操作符。**

当我们写下这样的代码时：

```C#
if (myGameObject == null) 
{
    // ...
}
```
你以为你在做一个简单的C# `null` 检查，但是实际上你调用的是Unity的自定义比较函数。这个函数的工作流程是：
1. 先检查C#包装器对象本身是不是真的 `null`
   
2. 如果不是，它会**跨越到C++层**，询问对应的引擎对象：“还活着吗老弟，还是说被打上销毁的标签了？”

这个“跨越到C++层去询问”的过程，就是Rider所说的“**隐式检查**”。它是“隐式”的，因为从代码表面上看，你只是在做 `== null` ，但背后却发生了很多的事情。

---

## 为什么Rider要提示呢？

Rider作为一款专业的IDE，它知道这个背后的机制，所以它的提示主要有两个原因：

1.  **性能意识 (Performance Awareness)**：这个“跨越到C++层”的检查虽然很快，但相比于一个纯粹的C# `null` 检查，它是有性能开销的。在大多数情况下，这点开销微不足道。但如果我们在 `Update` 或 `FixedUpdate` 等每帧执行多次的函数中，对大量对象进行这种检查，它可能会累积起来，对性能产生微小的影响。Rider在提醒你：“提示一下，这里正在发生一次Unity特有的、有轻微开销的操作。”

2.  **代码清晰度 (Clarity)**：Rider希望我们清楚地知道我们正在使用的是Unity的特殊行为，而不是标准的C#行为。这有助于我们写出更健壮、更易于理解的代码。
 
---

## 总结与最佳实践

所以，当Rider提示我们“Unity 对象生存期的隐式检查”时，它并不是说我们写错了，而是在对我们进行一次友好的、专业的教学。

**我们应该怎么做？**

1.  **绝大多数情况，请继续使用 `== null`**：
    这是最安全、最符合直觉的方式来判断一个Unity对象是否已经被销毁。它能正确处理“伪空”对象（即C#对象存在但C++对象已销毁的情况）。

    ```csharp
    public GameObject player;

    void Update()
    {
        // 这是正确且安全的方式，即使Rider有提示
        if (player == null)
        {
            Debug.Log("玩家对象已被销毁！");
            // 停止相关逻辑
            return;
        }
        // 在这里可以安全地使用player
        player.transform.position = Vector3.zero; 
    }
    ```

2.  **使用更符合Unity风格的简写**：
    `UnityEngine.Object` 还重载了布尔转换操作符。所以你可以这样写，效果和 `!= null` 完全一样，代码更简洁。

    ```csharp
    // if (player != null) 可以简写为:
    if (player) 
    {
        // 玩家对象存在且未被销毁
    }

    // if (player == null) 可以简写为:
    if (!player)
    {
        // 玩家对象为null或已被销毁
    }
    ```
    这种写法同样会触发“隐式检查”。

3.  **在极度追求性能的场景下（非常罕见）**：
    如果你能100%确定一个对象只会被真正地设为 `null`，而不会被 `Destroy()`，并且这段代码在性能瓶颈上，你可以使用 `System.Object.ReferenceEquals` 来强制进行纯C#的引用比较，绕过Unity的检查。

    ```csharp
    if (System.Object.ReferenceEquals(player, null))
    {
        // 这只会检查C#引用是否为null。
        // 如果player被Destroy()了，这里会判断为false，可能导致后续代码出错！
        // 新手请谨慎使用！
    }
    ```

---

## 引申几个知识点

### `MissingReferenceException` vs `NullReferenceException`

这是最直接的引申点。很多初学者会混淆这两个异常。

*   **`NullReferenceException` (空引用异常)**
    *   **发生原因**：这是纯C#层面的错误。你试图访问一个**真正为 `null`** 的变量的成员。这个变量从未被赋值，或者被显式地设为了 `null`。
    *   **例子**：
        ```csharp
        public Rigidbody rb; // 没有在Inspector里拖拽赋值
        void Start() {
            rb.AddForce(Vector3.up); // 100% 抛出 NullReferenceException
        }
        ```

*   **`MissingReferenceException` (丢失引用异常)**
    *   **发生原因**：这是Unity特有的错误。你试图访问一个**“伪空”对象**的成员。也就是说，C#包装器对象还存在，但它底层的C++引擎对象已经被 `Destroy()` 了。
    *   **例子**：
        ```csharp
        public GameObject enemy;
        void Start() {
            Destroy(enemy, 2f); // 2秒后销毁敌人
            StartCoroutine(AttackEnemyAfterThreeSeconds());
        }

        IEnumerator AttackEnemyAfterThreeSeconds() {
            yield return new WaitForSeconds(3f);
            // 此时enemy的C++对象已销毁，但C#变量enemy本身不是真null
            // 下面这行会抛出 MissingReferenceException
            enemy.transform.position = Vector3.zero; 
        }
        ```
    *   **关键点**：当你看到 `MissingReferenceException`，我们的第一反应应该是：“哦，这个东西不是不存在，而是**曾经存在过，但现在被销毁了**”。
  
  ---

### 垃圾回收 (Garbage Collection, GC) 的压力

我们知道了 `Destroy()` 并不会立即清理内存。它只负责标记C++对象。那么C#包装器对象呢？它最终由C#的**垃圾回收器（GC）**来回收。

*   **`Instantiate()` 和 `Destroy()` 的代价**：
    *   `Instantiate()`：在内存中创建新的C#和C++对象，会产生内存分配。
    *   `Destroy()`：标记C++对象，并将C#对象留在内存中，等待GC处理。

*   **问题所在**：如果在游戏中频繁地创建和销毁对象（比如射击游戏里的子弹、特效），就会产生大量的“内存垃圾”（等待被回收的C#对象）。GC会在它认为合适的时候启动，暂停你的游戏主线程，去清理这些垃圾。这个暂停过程如果很长，就会导致游戏**卡顿或掉帧**。
 
---

### 解决方案：对象池 (Object Pooling)

既然频繁创建和销毁的代价高，那么最好的办法就是**避免**它。对象池就是为此而生的设计模式。

*   **核心思想**：**重用**而不是销毁。
    1.  **预加载**：游戏开始时，一次性创建一批你需要的对象（比如100颗子弹），把它们全部设置为非激活状态(`SetActive(false)`)，并存放在一个列表里。
    2.  **“获取”**：当需要一颗子弹时，你不是 `Instantiate` 一个新的，而是从池子里取出一个非激活的，把它移动到发射位置，然后设置为激活状态(`SetActive(true)`)。
    3.  **“归还”**：当子弹击中目标或飞出屏幕时，你不是 `Destroy` 它，而是把它重新设置为非激活状态(`SetActive(false)`)，放回池子，等待下一次使用。

*   **好处**：整个游戏过程中，几乎没有新的内存分配和销毁操作，从而极大地减轻了GC的压力，让游戏运行得更平滑。这是从初级到中级开发者必须掌握的一个核心优化技巧。

---

### 值类型 (Structs) vs 引用类型 (Classes)

我们讨论的 `GameObject` 和 `Component` 都是**类（Class）**，它们是**引用类型**。这意味着变量存储的是指向内存中对象的“地址”。

但Unity中还有很多常用的**结构体（Struct）**，它们是**值类型**。比如：

*   `Vector3`
*   `Quaternion`
*   `Color`
*   `RaycastHit`

*   **关键区别**：值类型变量直接包含数据本身，而不是引用。它们没有C++/C#双重存在的概念，也不受 `Destroy()` 影响。

    ```csharp
    Vector3 position = new Vector3(1, 1, 1);
    Vector3 anotherPosition = position; // 这里是完整的数据拷贝

    anotherPosition.x = 5;
    // position.x 仍然是 1，因为anotherPosition是position的一个全新副本。
    ```

*   **知识点**：`Vector3` 这样的结构体，你永远不需要对它进行 `== null` 的判断，因为它根本不可能是 `null`。它就是一个值的集合，就像整数 `int` 一样。理解这一点有助于你写出更高效、更准确的代码。
   
---

### 总结一下这些引申点之间的关系：

1.  因为Unity对象有 **C#/C++双重结构**，所以 `Destroy()` 产生了“伪空”对象。
2.  访问这种“伪空”对象会抛出 **`MissingReferenceException`**，以区别于C#原生的 `NullReferenceException`。
3.  频繁的 `Instantiate`/`Destroy` 会给 **GC** 带来巨大压力，导致卡顿。
4.  为了解决GC压力，我们采用 **对象池** 技术进行性能优化。
5.  这个特殊的生命周期问题只存在于继承自 `UnityEngine.Object` 的**引用类型**，而像 `Vector3` 这样的**值类型**则完全不同。


