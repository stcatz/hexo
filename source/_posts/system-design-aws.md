---
title: "[译]如何设计一个在AWS上可以扩展到数百万用户的系统"
date: 2017-03-22 17:00:58
tags: [架构, 面试]
---

{% blockquote The System Design Primer https://github.com/donnemartin/system-design-primer Design a system that scales to millions of users on AWS %}
译注：《The System Design Primer》是一位大神[Donne Martin](http://donnemartin.com/)写的关于如何回答面试中系统设计方面问题的电子书，其中《如何设计一个在AWS上可以扩展到数百万用户的系统》最具有代表性。我觉得这篇文章不仅可以用来准备面试，更是可以作为开始一个新项目的架构指南。当然一篇文章能容纳的内容有限，本篇所涉及的很多话题可以很轻易的扩展成一本书，很多大公司会有专门的团队负责开发/维护文中的一些模块。希望读者可以把本篇文章作为一个互联网系统架构设计的提纲，在自己感兴趣的领域再深挖下去。
{% endblockquote %}

# 设计一个在AWS上扩展到数百万用户的系统
*注意：本文档直接链接到[系统设计主题](https://github.com/donnemartin/system-design-primer#index-of-system-design-topics)中找到的相关区域，以避免重复。请参阅链接内容，了解一般说话要点，权衡和替代方案。*
## 步骤1：概述用例和约束
>收集问题的要求和范围。
>提出问题来澄清用例和约束。
>讨论假设。

没有面试官来解决澄清问题，我们将定义一些用例和约束。

### 用例

解决这个问题需要一个迭代的方法：1）**基准/负载测试**，2）**配置文件**的瓶颈3）解决瓶颈，同时评估替代和权衡，以及4）重复，这是很好的模式用于将基本设计演变为可扩展设计。