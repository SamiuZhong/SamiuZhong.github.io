## Zygote的作用是什么？

- 启动SytemServer
  - SystemServer需要用到Zygote里面准备好的一些系统资源，如常用类、JNI函数、主题资源、共享库等
- 孵化应用进程

## 启动三段式

Android进程启动的常用套路

1. 进程启动
2. 准备工作
3. Loop

## Zygote的启动流程

### Zygote进程是怎么启动的？

- fork + handle
- fork + execve

### 进程启动之后做了什么？

