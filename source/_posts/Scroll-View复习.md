---
title: Scroll View复习
tags:
  - Unity
  - Scroll-View
categories: Unity基础&复习
abbrlink: efdc60b2
date: 2025-07-08 15:41:59
cover: https://image.nfasystem.top/img-nfa/Scroll-View.webp
---

## 什么是Scroll View

Scroll View简单来说就是Unity自带的一个可滚动区域的UI组件，即当内容超出屏幕显示范围是，其可以让用户自行滑动。

---

## Scroll View的基本构成
完整的Scroll View主要由以下部分组成
- **Scroll View**(Rect Transform): 此为整个滚动区域的**容器**，定义了用户可以看到内容的矩形区域。超出这个区域的内容会被裁剪掉（默认情况下）。
- **Viewport**(Rect Transform): 这是**很重要**的一个子对象，其为 Scroll View 的**直接子级**，并且与 Scroll View 的大小相同或略小。 Viewport 的作用是**裁剪内容**。只有位于 Viewport 内部的内容才会被显示出来。
- **Content(Rect Transform):** 这是所有可滚动内容的“**父容器**”。所有想要滚动显示的内容都应该作为 Content 的**子对象**。Content 的大小会根据其内部的内容的多少而动态变化，也可以手动设置。当Content的大小超出Viewport的大小的时候，滚动功能就会生效。
- **Scrollbar(Optional):** 滚动条是可选的，但它们提供了视觉指示和另一种滚动方式。
  - Horizontal Scrollbar: 用于水平滚动
  - Vertical Scrollbar: 用于垂直滚动
  
![alt text](https://image.nfasystem.top/img/ScrollView结构.png)

---

## Scroll View组件详解(Inspector面板)

![alt text](https://image.nfasystem.top/img/ScrollView面板.png)

**Scroll Rect组件**————这是Scroll View的核心逻辑组件。
> **这里只讲重要部分**
- Horizontal:
  - 勾选此项允许水平滑动
- Vertical:
  - 勾选此项允许竖直滑动
- Movement Type: 决定了滚动到达边界时的行为
  - Unrestricted: 内容可以无限滚动，即使超出边界也不会弹回。
  - Elastic: (最常用) 内容可以稍微超出边界，然后像橡皮筋一样弹回。可以设置 Elasticity (弹性系数)。
  - Clamped: 内容到达边界后立即停止，不能超出。
  
![alt text](https://image.nfasystem.top/img/ScrollView_Movement_Type.png)

- **Elasticity:** (仅当 Movement Type 为 Elastic 时可用) 弹性系数，值越大，弹回的速度越快。
- Inertia: 
  - Inertia: 勾选此项后，当用户松开鼠标/手指时，内容会根据之前的拖动速度继续滚动一段距离，然后逐渐减速停止。
  - Deceleration Rate: (仅当 Inertia 勾选时可用) 减速速率，值越小，滚动停止得越慢。
- Scroll Sensitivity: 滚动灵敏度。值越大，鼠标滚轮或拖动一小段距离就能滚动更多内容。
- **Horizontal Scrollbar(Vertical Scrollbar原理相同):** 
  - Visibility:
    - Permanent: 滚动条始终可见。
    - Auto Hide: 只有在内容超出 Viewport 且正在滚动时才显示。
    - Auto Hide And Expand Viewport: 只有在内容超出 Viewport 且正在滚动时才显示，并且当滚动条隐藏时，Viewport 会自动扩展以填充滚动条留下的空间。

![alt text](https://image.nfasystem.top/img/ScrollView_Visibility.png)

- **On Value Changed (Scroll Rect):** 这是一个事件回调。当滚动位置发生变化时，会触发这个事件。可以通过脚本监听这个事件，例如，根据滚动位置加载更多内容（无限滚动）。

---

## Viewport 和 Content 的 Rect Transform 设置
> **这是 Scroll View 正常工作的关键**
- Viewport:
  - 通常，Viewport 的 Rect Transform 会被设置为与 Scroll View 的大小相同，并且锚点和位置都设置为 (0,0)。
  - **重要：** Viewport 上通常会有一个 Image 组件（用于背景）和一个 Mask 组件。Mask 组件是实现内容裁剪的关键！它会确保只有 Viewport 区域内的内容才可见。
- **Content:**
  - **锚点 (Anchors):** Content 的锚点设置非常重要，它决定了内容如何对齐和扩展。
    - 垂直滚动： 如果你主要进行垂直滚动，Content 的锚点通常设置为 Top-Left 或 Top-Center，并且宽度设置为与 Viewport 相同，高度根据内容动态调整。
    - 水平滚动： 如果你主要进行水平滚动，Content 的锚点通常设置为 Top-Left 或 Center-Left，并且高度设置为与 Viewport 相同，宽度根据内容动态调整。
    - **重要：** 如果内容是动态生成的，可能需要使用 Vertical Layout Group 或 Horizontal Layout Group 配合 Content Size Fitter 组件来自动调整 Content 的大小。
---
## 常用辅助组件(配合 Content 使用)
- **Vertical Layout Group / Horizontal Layout Group:**
  - 如果想让 Content 中的子元素（如多个按钮、文本行）自动按垂直或水平方向排列，并且自动调整间距，就需要用到这些组件。将对应组件添加到 Content 对象上。
- **Content Size Fitter:**
  - 这个组件非常有用，当 Content 对象上添加了 Layout Group 后，Content Size Fitter 可以根据 Layout Group 中子元素的总大小，自动调整 Content 的宽度或高度。
  - **Fit:**
    - Unconstrained: 不受限制。
    - Min Size: 最小尺寸。
    - Preferred Size: 优先尺寸（根据子元素总大小）。
  - 通常会将 Horizontal Fit 或 Vertical Fit 设置为 Preferred Size，以确保 Content 的大小能够容纳所有子元素。
---
## 常见问题与调试技巧
- **内容不滚动**
  - **检查 Content 的大小：** 确保 Content 的宽度/高度确实大于 Viewport 的宽度/高度。如果 Content 太小，自然不需要滚动。
  - **检查 Content 的锚点和轴心：** 确保它们设置正确，以便 Content 能够正确扩展。
  - **检查 Scroll Rect 的 Horizontal/Vertical 勾选：** 确保允许了正确的滚动方向。
  - **检查 Viewport 上的 Mask 组件：** 确保它存在且启用。
- **滚动条不显示**
  - **检查 Scrollbar 是否拖拽到 Scroll Rect 的对应字段：** 确保 Horizontal Scrollbar 和 Vertical Scrollbar 字段都关联了正确的滚动条对象
  - **检查 Scrollbar 的 Visibility 设置：** 如果是 Auto Hide，只有在内容超出且正在滚动时才显示。
  - **检查 Content 的大小：** 如果 Content 没有超出 Viewport，滚动条是不会显示的。
- **内容排列混乱**
  - **使用 Layout Group：** 确保 Content 对象上添加了 Vertical Layout Group 或 Horizontal Layout Group 来自动排列子元素。
  - **使用 Content Size Fitter：** 确保 Content 对象上的 Content Size Fitter 设置正确，以便 Content 能够根据子元素自动调整大小。
---
## 编程控制 Scroll View
> 可以通过脚本来控制 Scroll View 的行为
- `scrollRect.verticalNormalizedPosition` / `scrollRect.horizontalNormalizedPosition`: 获取或设置当前的滚动位置。这是一个0到1之间的值，0表示最底部/最左边，1表示最顶部/最右边。
  - 例如，将 `verticalNormalizedPosition` 设置为0可以滚动到最底部（常用于聊天记录）。
- `scrollRect.velocity`: 获取或设置当前的滚动速度。
- `scrollRect.StopMovement()`: 立即停止滚动。
- 监听 `On Value Changed` 事件:
``` C#
using UnityEngine;
using UnityEngine.UI;

public class MyScrollViewHandler : MonoBehaviour
{
    public ScrollRect myScrollRect;

    void Start()
    {
        if (myScrollRect != null)
        {
            myScrollRect.onValueChanged.AddListener(OnScroll);
        }
    }

    void OnScroll(Vector2 scrollPosition)
    {
        // scrollPosition.x 是水平滚动位置 (0-1)
        // scrollPosition.y 是垂直滚动位置 (0-1)
        Debug.Log("Scroll Position: " + scrollPosition);

        // 示例：当滚动到顶部时做一些事情
        if (scrollPosition.y >= 0.99f) // 接近顶部
        {
            Debug.Log("Reached near top!");
            // 可以在这里加载更多历史数据
        }
        // 示例：当滚动到底部时做一些事情
        if (scrollPosition.y <= 0.01f) // 接近底部
        {
            Debug.Log("Reached near bottom!");
            // 可以在这里加载更多新数据
        }
    }

    // 示例：通过代码滚动到最底部
    public void ScrollToBottom()
    {
        if (myScrollRect != null)
        {
            myScrollRect.verticalNormalizedPosition = 0f;
        }
    }

    // 示例：通过代码滚动到最顶部
    public void ScrollToTop()
    {
        if (myScrollRect != null)
        {
            myScrollRect.verticalNormalizedPosition = 1f;
        }
    }
}
```
---
## 后期目标
- [ ] 通过代码简单仿制一个Scroll View

**第一次编辑-2025年7月8日**
