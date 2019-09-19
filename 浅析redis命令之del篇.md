baiyan

## 命令语法
命令格式：
```c
DEL [key1 key2 …]
```
命令实战：
```c
127.0.0.1:6379> del key1
(integer) 1
```
返回值：被删除 key 的数量
## 源码分析
首先我们开启一个redis客户端，使用gdb -p redis-server的端口。由于del命令对应的处理函数是delCommand()，所以在delCommand处打一个断点，然后在redis客户端中执行以下几个命令：
```c
127.0.0.1:6379> set key1 value1 EX 100
OK
127.0.0.1:6379> get key1
"value1"
127.0.0.1:6379> del key1
```
首先设置一个键值对为key1-value1、过期时间为100秒的键值对，然后在100秒之内对其进行删除，执行del key1，,删除这个还没有过期的键。我们看看redis服务端是如何执行的：
```c
Breakpoint 1, delCommand (c=0x7fb46230dec0) at db.c:487
487	    delGenericCommand(c,0);
(gdb) s
delGenericCommand (c=0x7fb46230dec0, lazy=0) at db.c:468
468	void delGenericCommand(client *c, int lazy) {
(gdb) n
471	    for (j = 1; j < c->argc; j++) {
(gdb) p c->argc 
$1 = 2
(gdb) p c->argv[0]
$2 = (robj *) 0x7fb462229800
(gdb) p *$2
$3 = {type = 0, encoding = 8, lru = 8575647, refcount = 1, ptr = 0x7fb462229813}
```
我们看到，delCommand()直接调用了delGenericCommand(c,0)，貌似是一个封装好的通用删除函数，我们s进去，看下内部的执行流程。
```c
void delGenericCommand(client *c, int lazy) {
    int numdel = 0, j;

    for (j = 1; j < c->argc; j++) {
        expireIfNeeded(c->db,c->argv[j]); //看这个键是否过期，过期需要额外做一些其他操作
        int deleted  = lazy ? dbAsyncDelete(c->db,c->argv[j]) : dbSyncDelete(c->db,c->argv[j]); // 异步或同步删除
        if (deleted) {
            signalModifiedKey(c->db,c->argv[j]); // 事务相关
            notifyKeyspaceEvent(NOTIFY_GENERIC, "del",c->argv[j],c->db->id); //发布/订阅相关通知
            server.dirty++; //AOF持久化相关
            numdel++; //删除键的数量++，返回删除的数量就是这个值
        }
    }
    addReplyLongLong(c,numdel);
}
```
首先直接进行了一个for循环，循环次数为c->argc，我们打印出来是2。看这个变量名我们可以猜测，可能是del和key1这两个命令参数。我们打印c->argv\[0]，发现它是一个redis-object，然后我们看它的encoding所对应的类型为8，即embstr类型，一种优化版的字符串存储形式。为了查看它的内容，将该指针强转为char \*：
```c
(gdb) p *((char *)($3.ptr)) 
$4 = 100 'd'
(gdb) p *((char *)($3.ptr+1))
$6 = 101 'e'
(gdb) p *((char *)($3.ptr+2))
$7 = 108 'l'
(gdb) p *((char *)($3.ptr+3))
```
然后打印下一个参数：
```c
(gdb) p *c->argv[1]
$10 = {type = 0, encoding = 8, lru = 8575647, refcount = 1, ptr = 0x7fb4622297e3}
(gdb) p *(char *)($10.ptr)
$11 = 107 'k'
(gdb) p *((char *)($10.ptr+1))
$12 = 101 'e'
(gdb) p *((char *)($10.ptr+2))
$13 = 121 'y'
(gdb) p *((char *)($10.ptr+3))
$14 = 49 '1'
```
我们看到，c->argv数组中存储的就是del和key1的值。但是下面的for循环是从1开始，没有包含命令的名称，所以只对key1这个参数进行了一次循环。随后，调用expireIfNeeded()函数：
```c
472	        expireIfNeeded(c->db,c->argv[j]);
(gdb) s
expireIfNeeded (db=0x7fb46221a800, key=0x7fb4622297d0) at db.c:1167
1167	int expireIfNeeded(redisDb *db, robj *key) {
(gdb) n
1168	    if (!keyIsExpired(db,key)) return 0;
(gdb) 
1187	}
(gdb)
```
我们发现，它判断了键是否过期，如果没有过期，那就直接返回0。那么看来，对于删除命令，过期和非过期的删除是有区别的。先跳过这里，继续往下走：
```c
473	        int deleted  = lazy ? dbAsyncDelete(c->db,c->argv[j]) : dbSyncDelete(c->db,c->argv[j]);
```
这里有一个lazy的参数，lazy参数决定了后面是调用dbAsyncDelete()还是dbSyncDelete()函数，我们打印一下这个值：
```c
(gdb) p lazy
$15 = 0
```
lazy的值为0。看字面意思，好像代表不需要懒删除的意思，非懒删除对应的是dbSyncDelete()函数，即同步删除。这里我们可以知道，非懒删除对应同步删除。我们跟进dbSyncDelete(c->db,c->argv[j])：
```c
int dbSyncDelete(redisDb *db, robj *key) {
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr); //删除key1对应的过期时间字典entry
    if (dictDelete(db->dict,key->ptr) == DICT_OK) { // 删除字典中的key1-value1键值对
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```
在redis中，过期时间和数据是分别存在redis数据库下面的两个字典中的。字典的结构我们已讨论过。在过期时间字典中，key就是我们的key key1；而键空间字典会存储真正的key-value对。所以要去字典中分别删除过期时间以及数据的值。具体的删除逻辑我们先不深入去了解，只知道过期时间和key-value对是存在字典dict中就好：
```c
typedef struct redisDb {
    ...
    dict *dict;        // 键空间字典，保存数据库中所有键值对
    dict *expires      // 过期时间字典，保存键的过期时间
    ...
} redisDb;
```
接续往下走，删除成功后，deleted变量值为1，会执行下面的if，并调用两个函数，做一些事务、发布/订阅的操作，我们不深入讲解这两块，然后会将删除的数量++，然后将结果存到输出缓冲区，命令执行结束。
## 扩展
### 懒删除（lazy delete）
之前我们的例子是调用了dbSyncDelete()方法，是同步来进行删除操作的。但是如果lazy的值为true，即如果开启了c懒删除策略，就会调用dbAsyncDelete()方法：
```c
#define LAZYFREE_THRESHOLD 64
int dbAsyncDelete(redisDb *db, robj *key) {

    // 从过期键字典中删除过期键的时间
    if (dictSize(db->expires) > 0) dictDelete(db->expires,key->ptr);

	// 找到key在键空间字典中的位置指针
    dictEntry *de = dictUnlink(db->dict,key->ptr);
    if (de) {
	    // 通过键指针获取值
        robj *val = dictGetVal(de);
		// 计算并判断是否需要懒删除。如果只删除一个很小的key，不需要采用懒删除策略，直接同步删除即可。
        size_t free_effort = lazyfreeGetFreeEffort(val);
        // 当代价超过阈值64的时候，就会将懒删除任务分发给后台线程去做，不阻塞主进程
        if (free_effort > LAZYFREE_THRESHOLD && val->refcount == 1) {
            atomicIncr(lazyfree_objects,1); //懒删除对象数量++
            bioCreateBackgroundJob(BIO_LAZY_FREE,val,NULL,NULL); // 下发懒删除任务
            dictSetVal(db->dict,de,NULL);
        }
    }
    if (de) {
        dictFreeUnlinkedEntry(db->dict,de);
        if (server.cluster_enabled) slotToKeyDel(key);
        return 1;
    } else {
        return 0;
    }
}
```
我们可以看到，在懒删除之前需要计算当前的key-value对是否适合懒删除。当懒删除超出阈值64的时候才会开启懒删除策略。所以，我们知道，它使用异步线程对删除的键值对，进行延后内存回收。那么为什么要这样做呢？原因是如果所要删除的键值对所占用的内存空间非常大，一次性做同步删除的时间是非常久的，这样会导致主进程一直处于阻塞状态，无法为外部提供服务。所以，为了解决这个问题，redis在4.0版本中提出了懒删除的概念，通过异步线程的方式，解决了大key删除时的阻塞问题。异步线程在Redis内部有一个特别的名称，它就是BIO，全称是Background IO，意思是在背后默默干活的IO线程，它就是懒删除的载体与核心。
## 参考资料
[【Redis源码分析】Redis 懒删除(lazy free)简史](https://segmentfault.com/a/1190000017394458)