# 理解 data-oriented design

## 1. 基础

注意[Data-oriented design](https://en.wikipedia.org/wiki/Data-oriented_design) 和 [object-oriented programming](https://en.wikipedia.org/wiki/Object-oriented_programming) 是完全不同的两种编程方案，想靠简单引入ecs包就提升既有项目性能是不可能的。

**OOP**: 封装 public\private, 继承 层次结构；各种单独对象散布在内存中，对人类友好，对cpu处理并不高效。

**DOD**: 聚焦于数据。考虑需要哪些数据，如何在内存中构造数据，以便使CPU更高效访问数据。使用组合将对象分解成组件，组件分组组合成array，System按照项目算法需求遍历array。DOD考虑一次使用很多个组件，而不是一个对象。

使用DOD，需要忘记OOP的基本概念，忘记封装和数据隐藏，忘记继承和多态，这些概念没有帮助。

考虑以下图中示例，一个游戏，player移动绿球，可见OOP和DOD访问数据的区别。

![Beach Ball Simulator game screenshot](./pictures/1_ball_game.png)
![DOD vs OOP data access](./pictures/1_oop_vs_dod.png)

**OOP**: 遍历所有Sphere类，设置颜色是绿色的Sphere的位置。即使存放Sphere的数组是连续数据，但是其中存的实际上是Sphere对象的引用，Sphere对象数据实际散布在内存中，这样遍历时会导致更多的cache misses.

**DOD**: Sphere分解成Color组件和Position组件，并各自组成buffer，这样cache missg更少处理更快。

## 2. 学习资料

### DOD 入门

- Video: [Crash Course Computer Science: describing what makes modern CPUs performant](https://www.youtube.com/watch?v=rtAlC5J1U40)
- Video: [Scott Meyers : CPU Caches and Why You Care](https://www.youtube.com/watch?v=WDIkqP4JbkE)
- Video: [CppCon 2014 Mike Acton Data-Oriented Design and C++](https://www.youtube.com/watch?v=92KFSD3ObrY)
- Slide deck: [Aras on ECS and Data-Oriented Design](http://aras-p.info/texts/files/2018Academy%20-%20ECS-DoD.pdf)
- Video: [Memory and Caches](https://youtu.be/4_smHyqgDTU)
- Video: [Andreas Fredriksson: GDC 2015 - SIMD at Insomniac Games: How We Do the Shuffle](https://gdcvault.com/play/1022248/SIMD-at-Insomniac-Games-How)

### Unity中的DOD

[unity learn](https://learn.unity.com/tutorial/part-1-understand-data-oriented-design)

### 进阶阅读

[unity learn](https://learn.unity.com/tutorial/part-1-understand-data-oriented-design)

### Unity的 DOD packages

技术栈：

- [Collections](https://docs.unity3d.com/Packages/com.unity.collections@latest)
- [Job System](https://docs.unity3d.com/Manual/JobSystem.html)
- [Burst](https://docs.unity3d.com/Packages/com.unity.burst@latest)
- [Entities](https://docs.unity3d.com/Packages/com.unity.entities@latest)
- [Entities Graphics](https://docs.unity3d.com/Packages/com.unity.entities.graphics@latest)

示例：

- [ECS Samples repository](https://github.com/Unity-Technologies/EntityComponentSystemSamples)

## 3. Key Principles

- **Design before you code.** 写代码前先设计。先设计好需要哪些数据、数据格式和分组如何组织，能够使CPU最有效访问数据。设计好System应该怎样transform data.
- **Design for efficient memory cache usage.** 高效存储缓存设计。将数据打包到连续内存buffer中，Systems和Jobs需要线性迭代buffer中的数据，充分利用cache line.如果项目中不是这样组织的，那么需要重新设计。
- **Design for blittable data.** 使用blittable数据。blittable类型的数据无需引用、指针啥的额外处理，就可以在job system中使用。注意不要使用托管类型（managed types）：不要用class, reference, 理想情况下也没有string。尽可能多使用DOTS代码，尽可能用并行。用Burst也需要blittable数据类型。
- **Design for the common case.** 针对常见情况设计。不可能使所有的数据结构都是最高效的，优化edge case是浪费时间。专注于高频操作。10000 times per frame > 1 per frame > 1 per seconds > only during initialization.
- **Embrace iteration.** 拥抱迭代。最初的数据设计不一定最优。DOD更加模块化，对迭代友好。有更高效算法全部替换 systems 和 components。
