# IO多路复用

[参考](https://xiaolincoding.com/os/8_network_system/selete_poll_epoll.html)

## 一. 什么是多路复用

1. 多路 -- 多个IO请求
2. 复用 -- 由同一个线程进行处理

<img src="https://vip2.loli.io/2022/10/21/R8zg7A36e4poVUm.png" alt="image-20221021162258965" style="zoom:33%;" />

## 二. select

> **都是使用「线性结构」存储进程关注的 Socket 集合**

### 过程

1. 已连接的 Socket 都放到一个`文件描述符集合`
2. 调用 select 函数将文件描述符**集合拷贝到内核**里
3. 内核通过`遍历`文件描述符集合的方式，当有事件产生时，将socket`标记`为可读可写
4. 内核再把整个文件描述符集合**拷贝回用户态**
5. 用户态还需要再通过`遍历`的方法找到可读或可写的 Socket，然后再对其处理。

### 缺点

1. 2 次「**遍历**」文件描述符集合
2. 2 次「**拷贝**」文件描述符集合
3. 使用固定长度的 BitsMap，表示文件描述符集合，有大小`1024`的限制

> poll，取而代之用动态数组，以链表形式来组织，突破了 select 的文件描述符个数限制，当然还会受到系统文件描述符限制

## 三. epoll 

### 过程

1. epoll_create 创建一个 epoll对象 epfd 
2. 再通过 epoll_ctl 将需要监视的 socket 添加到epfd中
3. 最后调用 epoll_wait 等待数据。

```c
int s = socket(AF_INET, SOCK_STREAM, 0);
bind(s, ...);
listen(s, ...)

int epfd = epoll_create(...);
epoll_ctl(epfd, ...); //将所有需要监听的socket添加到epfd中

while(1) {
    int n = epoll_wait(...);
    for(接收到数据的socket){
        //处理
    }
}
```

### 解决select/poll问题

1. epoll 在内核里使用**红黑树来跟踪进程所有待检测的文件描述字**，把需要监控的 socket 通过 `epoll_ctl()` 函数加入内核中的红黑树里，红黑树是个高效的数据结构，增删改一般时间复杂度是 `O(logn)`。所以只需要传入一个待检测的 socket，减少了内核和用户空间大量的数据拷贝和内存分配。
2. epoll 使用**事件驱动**的机制，内核里**维护了一个链表来记录就绪事件**，当某个 socket 有事件发生时，通过**回调函数**内核会将其加入到这个就绪事件列表中，当用户调用 `epoll_wait()` 函数时，只会返回有事件发生的文件描述符

<img src="https://vip2.loli.io/2022/10/21/l5M2wWfNBXhaTAo.png" alt="image-20221021164049260" style="zoom: 50%;" />