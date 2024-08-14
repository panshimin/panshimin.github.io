---
layout:     post
title:      知乎连续消费容器
subtitle: 
date:       2020-11-15
author:     潘世民
header-img: img/2022-12-10/2022-12-10-知乎详情页.jpeg
catalog: true
tags:
    - Hybrid
    - 详情页
    - 回答
    - WebView
    - ScrollView
    - 翻页
---

## 背景
为了给用户提供沉浸式阅读体验，需要把回答和文章详情页的翻页方案从通过前端实现调整为原生实现；同时需要支持回答、文章等不同内容的流通；这就需要一个容器或框架来支持，但是目前公司内没有相关的技术方案，故设计了统一容器来实现；

### 架构图

![编组：Final2](/img/2020-11-15/omni_container_arch.jpg)

### 架构细节

#### 一、类图关系

```
.
├── OmniProtocol
│   ├── OmniViewControllerDataSource
│   ├── OmniViewControllerDelegate
│   └── OmniViewControllerAppearanceDelegate
├── OmniViewController
│   └── OmniView
│       └── OmniScrollView
│           └── OmniScrollViewWrapperView
│               ├── OmniRefresh
│               └── Hybrid
```

#### 一、面向协议

统一容器作为基础交互 SDK，需要保证上层接口的稳定性，内部逻辑对业务黑盒，业务接入只需要面向协议接口编程即可

协议从三方方面考虑：

- 数据源提供 
- 代理事件回调
- 生命周期监听

#### 二、交互逻辑

##### 联动

![编组：Final3](/img/2020-11-15/omni_container_interaction.jpg)

1、内容视图是直接处理滚动事件，通过手势依赖，用户的滚动事件会被「交互视图」托管处理

2、滚动视图上添加两个 wrapper view 作为内容的容器载体，视觉上的内容窗口实际上就是 wrapper view

3、监听交互视图的滚动事件，当用户上下滚动的时候，会针对以下两种情况做联动的判断处理：

- 头部展示的内容未滚动到底部的时候，将交互视图的滚动距离 contentOffset 实时的同步传递到当前展示的内容视图上，视觉上保持同步； 实时更新 wrapper view 位置，使其始终展示在屏幕中
- 当内容滑动到底部的时候，交互的滚动距离不再同步给展示视图，此时开始移动 wrapper view，实现 next 内容露出预览的效果

4、交互视图为了实现与内容一致的滚动距离，内部需要监听 content scrollview 的 contentSize 的变化，同时添加额外的 footer 与预览高度后，同步配置到交互视图上

注：OmniScrollView 中的 wrapper 以及 content 在视图层级上都需要以 OmniScrollView 作为父视图，只有这样，才能实现 content scrollview 与 OmniScrollView 的手势依赖

##### 转场

![编组：Final4](/img/2020-11-15/omni_container_animation.jpg)

1、为了避免转场动效的干扰，转场中禁止交互视图与内容视图的 contentOffset、contentSize 的同步

2、转场之前，会改变 wrapper view 的图层关系，将其添加到 OmniView 上，这样能够避免交互视图 scrollView 滚动的影响

3、转场过程就是单纯的提交动画，修改两个 wrapper view 的位置，previous 与 next 的操作相当于循环的交换两个视图的位置来实现更替

4、结束转场，将交互视图的 contentOffset 重置到置顶位置，同时恢复 wrapper view 的图层到 OmniScrollView 上

#### 三、尺寸自适应

##### 初始大小可配

![编组：Final5](/img/2020-11-15/omni_container_size.jpg)

1、统一容器默认对 content 的 size 处理：设置与容器相同的高度

2、由于 content 可能是短内容，因此需要支持业务自定义配置 content 的（初始）高度：Omni 通过代理像业务询问每个 content 的 preferredContentsRect，最终来确定展示内容的大小

- **preferredContentsRect**：类似于 CALayer.contentsRect，参数的取值是相对的倍数，即 1 表示一倍的容器宽高，0 表示无宽高，默认值是 **{0, 0, 1, 1}** - CGRect(x: 0, y: 0, width: 1, height: 1)
- **CGRect.omniContentsRect**：默认值获取方法

##### Size 自适应

![编组：Final6](/img/2020-11-15/omni_contianer_auto_size.jpg)

1、内容在添加到 wrapper view 上的时候，size 通过 contentsRect 进行初始设置

2、当监听到内容视图的 contentSize 发生变化的时候，以动画的方式更新 wrapper view 的大小，同时根据自适应的策略决定是否允许小于或者大于初始值的 contentSize 触发自适应

3、AdjustSizePolicy

- none：不处理自适应
- shrink：只允许小于初始 contentsRect 配置高度的 contentSize 进行自适应
- expand：只允许大于初始 contentsRect 配置高度的 contentSize 进行自适应
- either：完全根据 contentSize 进行大小自适应

#### 四、生命周期

##### 托管生命周期

 1、容器整体托管内容页面的生命周期，内部禁止了子页面的生命周期自动调用

2、手动管理内容页面添加移除时的生命周期回调

3、预览 next 页面的生命周期额外处理滑动页面触发的回调：

- 上滑页面，next 页面刚露出的时候手动触发 **viewWillAppear**
- 下滑页面，next 页面刚消失的时候手动触发 **viewWillDisappear**、**viewDidAppear**
- 上拉或者点击 next 按钮，使 next 页面成为主内容页面，此时手动触发 **viewWillAppear**（如果 next 页面的状态已经是处于 viewWillAppear 了，此触发会被过滤），转场完成触发 **viewDidAppear**

