# JUC

## 一、Future

### 1、FutureTask

缺点

+ get方法阻塞
+ 无法进行异步任务的编排

### 2、CompleteFuture

+ 异步编排
+ 回调方式，不阻塞
+ 函数式编程
+ 链式调用

![函数式方法命名格式](C:\Users\admin\Desktop\函数式方法命名格式.png)

![image-20220102190306654](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220102190306654.png)

![image-20220102190354674](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220102190354674.png)

![image-20220102202432965](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220102202432965.png)

![image-20220102215536103](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220102215536103.png)

## sync字节码分析

+ 普通方法
+ 代码块
+ 静态方法

## 公平、非公平

+ 非公平
  + 锁饥饿
+ 公平
  + 上下文切换，有前驱节点的话肯定不是当前thread
  + 前驱锁判断，浪费CPU时间片

## 可重入锁

+ sync
+ ReentreLock

## 死锁、排查

+ jps + jstack
+ jconsole

## 中断

+ 中断协商机制![image-20220103151508423](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220103151508423.png)
+ ![image-20220103151705491](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220103151705491.png)
+ valatila
+ AutomaticIBoolean
+ static interrupt()

## LockSupport

+ sync-wait-notify
  + 必须在同步区域
  + 具有严格的执行顺序
  + ![image-20220104204017848](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220104204017848.png)
+ lock-awaite-singal
  + ![image-20220104204407034](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220104204407034.png)
  + ![image-20220104204545524](C:\Users\admin\AppData\Roaming\Typora\typora-user-images\image-20220104204545524.png)

+ LockSupport (park-unpark) permit许可证方式
  + 