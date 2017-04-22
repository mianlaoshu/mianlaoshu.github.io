# RxJava源码分析

从角色划分上，RxJava主要可分为三大块：
1. 创建可观测数据源
2. Operator进行数据变换
3. 设置观察者。

严格来讲，第二点Operator数据变换不仅仅包含对数据流本身操作，也还包括其他如线程切换。

从数据流状态，包括onNext onComplete/onError，以及背压Backpressure，其中onError牵扯到的RxJava的错误处理，后面详细了解可以单独分析。

此外就我目前看到的源代码，从框架层面还包括hook机制RxJavaPlugins，调度器，等等

RxJava代码量较大，2.0.1时大约1400+文件，打包后大约1.9M.作为一个纯Java库还是很大的，而且一般不能混淆，避免一些莫名其妙的bug。

为了学习RxJava，本文尝试深入分析RxJava的源代码，主要以2.0为主。初学者难免纰漏，欢迎多提意见:)
