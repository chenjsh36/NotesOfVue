## Vue 源码学习笔记

本项目的源码学习笔记是基于 Vue 1.0.9 版本的也就是最早的 tag 版本，之所以选择这个版本，是因为这个是最原始没有太多功能拓展的版本，有利于更好的看到 Vue 最开始的骨架和脉络以及作者的最初思路。而且能和后续的 1.x.x 版本做对比，发现了作者为了修复 bug 而做出的很多有趣的改进甚至回退，如 [vue nextTick](./Vue_8.md) 的版本迭代经历了更新、回退和再次更新


### 目录

* [项目结构、目录](./Vue_1.md) 了解了 Vue 项目的各个目录的功能、构建方案，学习 Vue 实例构造函数等

* [vue 工具类函数实现](./Vue_2.md) util 目录的工具类实现

* [vue 工具类函数实现2](./Vue_3.md) util 目录的工具类实现

* [vue 实例构造函数](./Vue_4.md) 

* [vue 订阅观察者类](./Vue_5.md) observer 目录下实现的 dep 类是一个订阅观察者，用于后续讲到了实现数据响应化里

* [vue 指令解析类](./Vue_6.md) parsers 目录主要作为指令解析实现，在 vue 的 compile 阶段使用到

* [vue transition](./Vue_7.md) transition 目录为实现元素在插入和移出 dom 时的过渡逻辑

* [vue nextTick](./Vue_8.md) 从 nextTick 中了解 JS 的真实运行机制， 了解 microtask 和 task 的本质

* [vue 数据响应化](./Vue_9.md)  defineProperty 可以用于实现数据响应化，但是只能解决部分功能，并不是全部

* [vue 编译过程](./Vue_10.md) 

