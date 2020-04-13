## O(1)调度算法

Linux是一个支持多任务的操作系统，而多个任务之间的切换是通过 `调度器` 来完成，`调度器` 使用不同的调度算法会有不同的效果。

Linux2.4版本使用的调度算法的时间复杂度为O(n)，其主要原理是通过轮询所有可运行任务列表，然后挑选一个最合适的任务运行，所以其时间复杂度与可运行任务队列的长度成正比。

而Linux2.6开始替换成名为 `O(1)调度算法`，顾名思义，其时间复杂度为O(1)。虽然在后面的版本开始使用 `CFS调度算法（完全公平调度算法）`，但了解 `O(1)调度算法` 对学习Linux调度器还是有很大帮助的，所以本文主要介绍 `O(1)调度算法` 的原理与实现。

