## 图例

![Compact 行格式](https://vip2.loli.io/2022/06/12/8PbUZrEctyuaKRM.png)

Compact 行格式

![索引结构](https://vip2.loli.io/2022/06/12/UdxkBHDy5e9nw7s.png)

B+树，非叶子节点-索引目录；叶子节点是数据

![B-Tree](https://vip2.loli.io/2022/06/12/1xyK4WpE2mkBOP5.png)

B树

![页分裂](https://vip2.loli.io/2022/06/12/mRiBNfQaCbMen4K.png)

页分裂

## 索引设计原则

### 适合创建索引

> 区分度高、频繁查询、索引字段长度尽可能小

1. 字段的数值有唯一性的限制
2. 频繁作为 WHERE 查询条件的字段

3. 经常 GROUP BY 和 ORDER BY 的列

4. UPDATE、DELETE 的 WHERE 条件列
5. DISTINCT 字段需要创建索引
6. 使用列的类型小的创建索引
7. 使用字符串前缀创建索引
8. 使用最频繁的列放到联合索引的左侧
9. 区分度高(散列性高)的列适合作为索引

### 不适合创建索引

> 数据重复区分度低、表数据少、无序、经常更新

1. 在where中使用不到的字段，不要设置索引

2. 数据量小的表最好不要使用索引

3. 有大量重复数据的列上不要建立索引

4. 避免对经常更新的表创建过多的索引

5. 不建议用无序的值作为索引

6. 不要定义冗余或重复的索引

   

## 索引失效

1. 计算、函数、类型转换(自动或手动)
2. 最左前缀原则， like以通配符%开头
3.  OR 前后存在非索引的列
4. 数据库和表的字符集统一使用utf8mb4(字符集)
5. 不等于
6.  is null可以使用索引，is not null无法使用索引

## 索引创建

### 建表时创建

![建表时索引](https://vip2.loli.io/2022/06/12/Y6lE3NUx4PgLbfC.png)

### 表存在时，创建索引

![存在时创建](https://vip2.loli.io/2022/06/12/bctdCoGTmNULDAy.png)