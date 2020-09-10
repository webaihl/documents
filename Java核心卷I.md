### 异常
+ 都继承自Throwable ，分为Error和Exception
+ 受检异常应该**早抛出、晚捕获**
+ 将正常代码与异常处理分开
+ RuntimeException属于非受检异常，不需要显示捕获
+ try-with-resource 会在代码块执行完后自动关闭资源(实现AutoCloseable接口)
+ finally中的return会替换逻辑中的return

### 数值
+ Java中都是值传递