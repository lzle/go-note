# Go 调度


M 数量是动态变化的。

M0  程序第一个线程，创建第一个G时与其他M一致

G0  每个M都有一个，负责调度G

自旋线程

自旋线程优先从全局队列中取G

发生系统调用时(阻塞/非阻塞)都解绑P与M

https://www.kancloud.cn/aceld/golang/1958305
