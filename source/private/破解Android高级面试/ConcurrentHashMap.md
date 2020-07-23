## CHM的并发优化历程

- JDK 5：分段所，必要时加锁
- JDK 6：优化二次Hash算法（解决Hash不均匀分布）
- JDK 7：段懒加载，volatile & cas
- JDK 8：摒弃段，基于HashMap原理的并发实现

8之前分段锁：segment[] --> table[]

8之后：table[]