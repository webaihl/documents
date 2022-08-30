---

---

# 集群

## 节点分布

### 节点启动

```shell
redis-server master_node.conf
```



<img src="https://vip2.loli.io/2022/08/28/zfOTpuBhwJFAL2r.png" alt="image-20220828121857943" style="zoom:67%;" />

### 节点连接状态

| 集群数据结构 | 说明                                        | 备注                                                         |
| ------------ | ------------------------------------------- | ------------------------------------------------------------ |
| clusterNode  | 保存一个节点的当前状态                      | ctime、name(40个十六进制字符),ip,port,负责的槽,currentEpoch(故障转移) |
| clusterStat  | 每个节点一个,记录`当前节点视角`下的集群状态 | 上下线、在线节点数，节点清单<name,clusterNode>，所有节点负责的槽，currentEpoch |
| clusterLink  | 连接节点所需要的信息                        | ctime,socket_fd,S_R_buf,连接到的节点清单                     |

<img src="https://vip2.loli.io/2022/08/28/zeCYdXvs8g3iFhp.png" alt="image-20220828123114320" style="zoom: 50%;" />

### 节点meet

> 客户端连接A节点，A收到meet命令，将指定的B<ip,port>节点加入A所在的集群，加入之后，通过Gossip协议传播到其他节点

```shell
# 连接到A节点，指定B的ip和端口
cluster meet <ip> <port>
```

<img src="https://vip2.loli.io/2022/08/28/YGvAF9Ssqk12rKi.png" alt="image-20220828125125770" style="zoom:67%;" />

## 槽指派

> 各个节点形成一个集群之后，依旧处于`下线`状态，需要将所有的`16384`个slot分配到各个节点，才可上线，任意一个槽未被主节点处理，集群就不能正常提供服务

1. 使用客户端依次连接到各节点

2. 执行以下命令

   ```shell
   # 添加 since 3.0.0 
   # cluster addslots 1 2 3 4 ... 5000
   cluster addslots slot [slot ...]
   # 范围添加 since 7.0.0
   #  CLUSTER ADDSLOTSRANGE 1 5    ==  CLUSTER ADDSLOTS 1 2 3 4 5
   cluster addslotsrange start-slot end-slot [start-slot end-slot ...]
   ```

3. 各个节点槽信息保存，以及传播

   <img src="https://vip2.loli.io/2022/08/28/EIrDnib9xMpKUha.png" alt="image-20220828141158970" style="zoom:50%;" />

## 在集群中执行命令

### 判断流程

<img src="https://vip2.loli.io/2022/08/28/coHYfr4NUJ3wQ2q.png" alt="image-20220828142744371" style="zoom:50%;" />

<img src="https://vip2.loli.io/2022/08/28/Xm4zpsInwqWufAE.png" style="zoom: 67%;" />

#### 槽计算

```c
i = CRC16(key) & 16383
```

#### 判断槽是否是由当前节点处理

```c
clusterState.myself == clusterState.solts[i]
```

### Moved错误

> redis-server -c -p xxx

如果槽不是由当前节点处理，节点则会向client处理返回错误，指引到正在负责该槽的节点

## 重新分片

> 将任意数量的`槽`，由当前节点指派给某个节点。可以`在线`执行，集群无需下线，迁移过程中仍会处理用户请求

+ 有新的节点加入
+ 有节点需要下线
+ 某些节点数据分布不均匀需要重新调整

### 怎样分片

> 需要先安装ruby

```sh
# 人工使用redis-trib工具
./src/redis-trib.rb
```

### 分片流程

### ASK错误

client请求的槽，正在迁移中(clusterState.migrating_slots_to),重定向后(需接收asking命令)，目标节点会查询clusterState.importing_solts_from数组中查询。

## 新增节点

+ 加入节点

  ```shell
  cluster meet ip port
  ```

+ 或者。利用redis-trib工具,增加节点

  ```sh
  redis-trib add-node new_host:new_port existing_host existing_port
  ```

+ 槽迁移

  ```sh
  redis-trib reshard host:port --from <> --to <> --slots <> --yes --timeout <> --pipleline <>
  # host:port require 集群内任意节点，获取集群信息
  # from 源节点ID
  # to 目标节点ID
  # slots 迁移的槽总数
  # yes 执行计划是否需用户确认
  # timeout 每次migrate操作的超时时间，default 60s
  # pipeline 每次批量迁移的键数量
  ```

## 下线节点

> + 有槽先迁移到其他节点，与新增节点操作流程一致，保证完整性
> + 不再负责槽或者是从节点，通知其他节点忘记下线节点
> + 所有节点都忘记下线节点时，关闭该节点

+ 只下线主节点，提前将修改从节点的主节点指向

  ```sh
  # 连接到从节点
  cluster replicate master_node_id
  ```

+ 主从都下，先下从再下主

```sh
redis-trib del-node host:port down_node_id
```

## 相关面试题

1. 如果新增节点槽正在迁移，向该节点发送请求，会出现什么情况
   1. ask错误

2. 相关语言客户端是否每次都要查询键属于哪个槽

   1. 客户端会更新key-slot-node的信息
   2. 在碰到moved错误或者最后一次重试时更新，ask不会

3. ask和moved区别

   | 命令  | 说明                                                         |
   | :---- | ------------------------------------------------------------ |
   | moved | 说明该槽的负责权**已经**转移为返回的节点，客户端下次可以**直接**与返回的节点通信，第三方客户端可立即更新solts缓存 |
   | ask   | 重定向节点接收asking命令**后**，**临时尝试**执行命令，不会影响之后对该槽的请求 |


## redis-cli 命令总结

> since 5.0.0

### 将B加入A所在集群

```sh
redis-cli -h A_host -p A_port meet up <B_ip> <B_port>
```

### 将C1作为C的从节点

```sh
redis-cli -h C1_host -p C1_port replicate <C_node_id>
```

### 独立的多个节点建立集群

```sh
# 所有主机，以及每个节点的备份数量
redis-cli --cluster create  127.0.0.1:70001 127.0.0.1:70002  127.0.0.1:7003 127.0.0.1:7004  127.0.0.1:7005 127.0.0.1:7006  --cluster-replicas 1
```

### 集群扩容

1. 加入集群
2. 从节点设置
3. 分配槽

```sh
# 通过已知节点加入新节点
redis-cli --cluster add-node 127.0.0.1:7007 127.0.0.1:7001
redis-cli --cluster add-node 127.0.0.1:7008 127.0.0.1:7001

# 将7008作为7007的从节点
redis-cli -h 127.0.0.1 -p 7008 replicate <7007_node_id>

# 新节点已经加入了集群，但是还未分配槽
# 选择迁移的槽数量，源节点，目标节点
redis-cli --cluster reshard 127.0.0.1 7001  
```

### 集群缩容

1. 将槽迁移到其他节点
2. 删除节点

```sh
# 将槽迁移到其他节点
redis-cli --cluster reshard 127.0.0.1 7001

# 删除节点7007，7008,并下线
redis-cli --cluster del-node 127.0.0.1:7007  7007_node_id 
redis-cli --cluster del-node 127.0.0.1:7008  7008_node_id 
```

### 节点槽平衡

```sh
redis-cli --cluster rebalance 127.0.0.1 7001
```

### 参数

```sh
redis-cli --cluster help
Cluster Manager Commands:
  create         host1:port1 ... hostN:portN   #创建集群
                 --cluster-replicas <arg>      #从节点个数
  check          host:port                     #检查集群
                 --cluster-search-multiple-owners #检查是否有槽同时被分配给了多个节点
  info           host:port                     #查看集群状态
  fix            host:port                     #修复集群
                 --cluster-search-multiple-owners #修复槽的重复分配问题
  reshard        host:port                     #指定集群的任意一节点进行迁移slot，重新分slots
                 --cluster-from <arg>          #需要从哪些源节点上迁移slot，可从多个源节点完成迁移，以逗号隔开，传递的是节点的node id，还可以直接传递--from all，这																  样源节点就是集群的所有节点，不传递该参数的话，则会在迁移过程中提示用户输入
                 --cluster-to <arg>            #slot需要迁移的目的节点的node id，目的节点只能填写一个，不传递该参数的话，则会在迁移过程中提示用户输入
                 --cluster-slots <arg>         #需要迁移的slot数量，不传递该参数的话，则会在迁移过程中提示用户输入。
                 --cluster-yes                 #指定迁移时的确认输入
                 --cluster-timeout <arg>       #设置migrate命令的超时时间
                 --cluster-pipeline <arg>      #定义cluster getkeysinslot命令一次取出的key数量，不传的话使用默认值为10
                 --cluster-replace             #是否直接replace到目标节点
  rebalance      host:port                                      #指定集群的任意一节点进行平衡集群节点slot数量 
                 --cluster-weight <node1=w1...nodeN=wN>         #指定集群节点的权重
                 --cluster-use-empty-masters                    #设置可以让没有分配slot的主节点参与，默认不允许
                 --cluster-timeout <arg>                        #设置migrate命令的超时时间
                 --cluster-simulate                             #模拟rebalance操作，不会真正执行迁移操作
                 --cluster-pipeline <arg>                       #定义cluster getkeysinslot命令一次取出的key数量，默认值为10
                 --cluster-threshold <arg>                      #迁移的slot阈值超过threshold，执行rebalance操作
                 --cluster-replace                              #是否直接replace到目标节点
  add-node       new_host:new_port existing_host:existing_port  #添加节点，把新节点加入到指定的集群，默认添加主节点
                 --cluster-slave                                #新节点作为从节点，默认随机一个主节点
                 --cluster-master-id <arg>                      #给新节点指定主节点
  del-node       host:port node_id                              #删除给定的一个节点，成功后关闭该节点服务
  call           host:port command arg arg .. arg               #在集群的所有节点执行相关命令
  set-timeout    host:port milliseconds                         #设置cluster-node-timeout
  import         host:port                                      #将外部redis数据导入集群
                 --cluster-from <arg>                           #将指定实例的数据导入到集群
                 --cluster-copy                                 #migrate时指定copy
                 --cluster-replace                              #migrate时指定replace
  help           

For check, fix, reshard, del-node, set-timeout you can specify the host and port of any working node in the cluster.
```

