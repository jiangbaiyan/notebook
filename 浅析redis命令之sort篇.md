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
一般情况下，BY和GET选项需要同时使用。
**保存排序结果**：为了客户端不重复进行排序，提高性能，redis提供了**STORE选项**将排序结果缓存的功能，将排序结果缓存到另一个键中：
```c
127.0.0.1:6379> sort user_id desc
1) "3"
2) "2"
3) "1"
127.0.0.1:6379> sort user_id desc store sort_cache
(integer) 3
127.0.0.1:6379> lrange sort_cache 0 -1
1) "3"
2) "2"
3) "1"
```
返回值：如果没有使用STORE选项，返回列表形式的排序结果。否则返回排序结果的元素数量
## 源码分析
sort命令的入口函数是sortCommand()，由于该命令较复杂，我将其分为几个阶段来讲解：
### 初始化及类型校验
```c
void sortCommand(client *c) {
    list *operations;
    unsigned int outputlen = 0;
    int desc = 0, alpha = 0;
    long limit_start = 0, limit_count = -1, start, end;
    int j, dontsort = 0, vectorlen;
    int getop = 0; /* GET operation counter */
    int int_conversion_error = 0;
    int syntax_error = 0;
    robj *sortval, *sortby = NULL, *storekey = NULL;
    redisSortObject *vector; /* Resulting vector to sort */

    /* 必须是集合/列表/有序集合类型才能够排序  */
    sortval = lookupKeyRead(c->db,c->argv[1]); // 去键空间字典中查找
    if (sortval && sortval->type != OBJ_SET &&
                   sortval->type != OBJ_LIST &&
                   sortval->type != OBJ_ZSET)
    { // 判断键对应的值的类型，如果不符合要求则报错
        addReply(c,shared.wrongtypeerr);
        return;
    }
    /* GET参数使用的链表  */
    operations = listCreate();
    listSetFreeMethod(operations,zfree);
    j = 2; /* 选项从第二个参数argv[2]开始 */

    /* 保护排序操作过程中的键不被修改、删除*/
    if (sortval)
        incrRefCount(sortval); // 排序对象存在，一般都会进这个case
    else
        sortval = createQuicklistObject(); // 排序对象不存在，为什么创建快表？
```
在这一阶段，redis做了一些变量初始化，并判断了排序对象的类型，只有集合、列表、有序集合才允许排序操作，最终创建了一个链表用来暂存GET操作涉及的元素，最后为了保护sort操作过程中的键不被破坏，使用引用计数来记录键的使用情况。
### 选项参数的处理
接下来，redis对我们在输入的各个选项，如LIMIT/BY/GET做一些处理及判断：
```c
    // 逐个处理我们输入的选项参数
    while(j < c->argc) { 
        int leftargs = c->argc-j-1;
        if (!strcasecmp(c->argv[j]->ptr,"asc")) { // ASC升序
            desc = 0;
        } else if (!strcasecmp(c->argv[j]->ptr,"desc")) { // DESC降序
            desc = 1;
        } else if (!strcasecmp(c->argv[j]->ptr,"alpha")) { // ALPHA是否按照字典序排序字符串
            alpha = 1;
        } else if (!strcasecmp(c->argv[j]->ptr,"limit") && leftargs >= 2) { // LIMIT是否筛选排序集合
            if ((getLongFromObjectOrReply(c, c->argv[j+1], &limit_start, NULL) // 判断输入的第一个参数offset是否符合整数规范
                 != C_OK) ||
                (getLongFromObjectOrReply(c, c->argv[j+2], &limit_count, NULL) // 判断输入的第二个参数limit是否符合整数规范
                 != C_OK))
            {
                syntax_error++; // 不符合规范，报错计数器++
                break;
            }
            j+=2;
        } else if (!strcasecmp(c->argv[j]->ptr,"store") && leftargs >= 1) { // STORE是否缓存排序结果
            storekey = c->argv[j+1];
            j++;
        } else if (!strcasecmp(c->argv[j]->ptr,"by") && leftargs >= 1) {
            sortby = c->argv[j+1];
            /* 对于没有通配符*的BY选项，不需要排序 */
            if (strchr(c->argv[j+1]->ptr,'*') == NULL) {
                dontsort = 1;
            } else { 
                /* 带有通配符*的BY选项，需要排序，但是不能在集群模式下执行 */
                if (server.cluster_enabled) {
                    addReplyError(c,"BY option of SORT denied in Cluster mode.");
                    syntax_error++; // 报错计数器++
                    break;
                }
            }
            j++;
        } else if (!strcasecmp(c->argv[j]->ptr,"get") && leftargs >= 1) { //GET选项不能在集群模式运行
            if (server.cluster_enabled) {
                addReplyError(c,"GET option of SORT denied in Cluster mode.");
                syntax_error++; // 报错计数器++
                break;
            }
            listAddNodeTail(operations,createSortOperation(
                SORT_OP_GET,c->argv[j+1])); // 将GET选项的正则参数暂存在链表中
            getop++;
            j++;
        } else { // 不属于以上选项中的任何一个
            addReply(c,shared.syntaxerr); // 使用了不合法的参数选项，需对客户端报错
            syntax_error++; // 报错计数器++
            break;
        }
        j++;
    }

    /* 处理以上几种报错计数器自增的情况. */
    if (syntax_error) {
        decrRefCount(sortval); // 引用计数-1
        listRelease(operations); // 释放GET操作使用的链表
        return;
    }
```
在这个阶段，redis对我们传入的各个参数进行了处理，并判断相关输入是否合法，然后将不合法的参数集中处理，减少引用计数并释放空间，最后将错误信息返回给客户端。
###