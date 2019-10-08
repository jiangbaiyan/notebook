baiyan

## 命令使用
命令含义：返回或保存给定列表、集合、有序集合key中经过排序的元素
命令格式：
```c
SORT key [BY pattern] [LIMIT offset count] [GET pattern [GET pattern ...]] [ASC|DESC] [ALPHA] [STORE destination]
```
在这里，我先不一个一个选项去讲解，首先我们直接看一下，这个命令在实战中是如何使用的
命令实战：
**一般使用场景**：针对某个列表、集合、有序集合键，对这个键中的所有数值做简单排序：
```c
127.0.0.1:6379> lrange mykey 0 -1
1) "1"
2) "2"
3) "3"
127.0.0.1:6379> sort mykey DESC
1) "3"
2) "2"
3) "1"
```
**对字符串进行排序**：redis默认采用数值大小的规则去排序。如果需要对字符串进行进行排序（按照字典序），需要显式添加**ALPHA选项**，否则会报错：
```c
127.0.0.1:6379> lpush name jiangbaiyan
(integer) 1
127.0.0.1:6379> lpush name grape
(integer) 2
127.0.0.1:6379> lpush name admin
(integer) 3
127.0.0.1:6379> sort name 
(error) ERR One or more scores can't be converted into double
127.0.0.1:6379> sort name ALPHA
1) "admin"
2) "grape"
3) "jiangbaiyan"
```
**限制返回结果**：排序之后返回元素的数量可以通过**LIMIT**选项进行限制，来筛选出我们需要的数据。后面需要填写offset和count两个参数，基本和我们写SQL的时候相同：
```c
127.0.0.1:6379> sort name ALPHA LIMIT 1 2
1) "grape"
2) "jiangbaiyan"
```
**使用外部key进行排序**：可以使用外部key的数据作为权重，代替默认的直接对比键值的方式来进行排序。这里就要使用我们的**BY**和**GET**选项了。这个解释听起来很别扭。这我们可以利用redis模拟一个关系型数据库的排序，大家就明白了。我们首先构造一个关系型数据库：
```c 
127.0.0.1:6379> lpush user_id 1
(integer) 1
127.0.0.1:6379> set user_name_1 admin
OK
127.0.0.1:6379> set user_level_1 9999
OK

127.0.0.1:6379> lpush user_id 2
(integer) 2
127.0.0.1:6379> set user_name_2 jiangbaiyan
OK
127.0.0.1:6379> set user_level_2 10
OK

127.0.0.1:6379> lpush user_id 3
(integer) 3
127.0.0.1:6379> set user_name_3 grape
OK
127.0.0.1:6379> set user_level_3 50
OK
```
那么，我们初始化之后的关系型数据库的结构如下：
|   user_id  |  name   |  level   |
| --- | --- | --- |
|  1   |  admin   |  9999   |
|  2   |   jiangbaiyan  |  10   |
|  3   |   grape  |   50  |
那么什么是利用外部key进行排序呢，在这里，就是根据关系型数据库中的其中一列，来对所有记录进行排序。举一个例子，我们对这个表按照level数值从大到小进行排序，并取出id：
```sql
SELECT user_id FROM redis ORDER BY level DESC;
```
在redis中，要实现这条SQL语句，我们就要使用BY选项了：
```c
127.0.0.1:6379> sort user_id by user_level_* desc
1) "1" // level 9999
2) "3" // level 50
3) "2" // level 10
```
那么GET选项怎么用呢？我们接着看一个SQL：
```sql
SELECT name FROM redis ORDER BY level DESC;
```
这个SQL和前面的SQL的区别仅仅在于筛选的列由user_id变成了name。由于我们排序的对象结果集中并没有name数据，所以需要使用GET选项关联name列：
```c
127.0.0.1:6379> sort user_id by user_level_* desc get user_name_*
1) "admin" // level 9999
2) "grape" // level 50
3) "jiangbaiyan" // level 10
```
一般情况下，BY和GET选项需要同时使用。、
返回值：如果没有使用STORE选项，返回列表形式的排序结果。否则返回排序结果的元素数量
## 源码分析
