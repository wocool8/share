# 队列
---
## 一 java内置队列
|队列|有界性|锁|数据结构|
|:-|:-|:-|:-|
|ArrayBlockingQueue|bounded|加锁|arrayList|
|LinkedBlockedQueue|optionally-bounded|加锁|linkedList|
|ConcurrentLinkedQueue|unbounded|无锁|linkedList|
|LinkedTransferQueue|unbounded|无锁|linkedList|
|PriorityBlockingQueue|unbounded|无锁|linkedList|
|DelayQueue|unbounded|无锁|linkedList|
## 二 阻塞队列
## 三 非阻塞队列
## 四 伪共享
## 五 Disruptor
## 六 Disruptor-解决伪共享
