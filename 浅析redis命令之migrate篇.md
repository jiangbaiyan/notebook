baiyan

## 命令语法
命令含义：将 key 原子性地从当前实例传送到目标实例的指定数据库上，一旦传送成功， key 保证会出现在目标实例上，而当前实例上的 key 会被删除
命令格式
```c
MIGRATE host port key|"" destination-db timeout [COPY] [REPLACE] [KEYS key [key ...]]
```
命令实战：将键key1迁移到本机6380端口的redis实例上，存储到目标实例的第0号数据库，超时时间为1000毫秒。可选项COPY如果表示不移除源实例上的 key ，REPLACE表示替换目标实例上已存在的 key 。keys表示可以同时传送多个keys（前面的key参数的位置必须设置为空）
```c
127.0.0.1:6379> migrate 127.0.0.1 6379 "" 0 5000 KEYS key1 key2 key3
OK
```
返回值：迁移成功时返回 OK ，否则返回错误
## 源码分析
由于migrate命令的代码较多，所以我分为几部分来讲解：
### 参数校验
```c
    migrateCachedSocket *cs; // 连接另一个实例的socket
    int copy = 0, replace = 0, j;
    char *password = NULL;
    long timeout; // 超时时间
    long dbid; // 数据库id
    robj **ov = NULL; /* 要迁移的对象 */
    robj **kv = NULL; /* 键名 */
    robj **newargv = NULL; /* Used to rewrite the command as DEL ... keys ... */
    rio cmd, payload;
    int may_retry = 1;
    int write_error = 0;
    int argv_rewritten = 0;

    /* 支持同时传输多个key. */
    int first_key = 3; /* 第一个键参数的位置. */
    int num_keys = 1;  /* 默认只传送一个key. */

    /* 校验其他选项，从COPY选项开始校验 */
    for (j = 6; j < c->argc; j++) {
        int moreargs = j < c->argc-1;
        if (!strcasecmp(c->argv[j]->ptr,"copy")) { // 如果命令参数等于copy，开启copy选项
            copy = 1;
        } else if (!strcasecmp(c->argv[j]->ptr,"replace")) { // 如果命令参数等于replace，开启replace选项
            replace = 1;
        } else if (!strcasecmp(c->argv[j]->ptr,"auth")) { // 如果命令参数等于auth，开启auth选项
            if (!moreargs) { // 参数数量超出规定数量，报错
                addReply(c,shared.syntaxerr);
                return;
            }
            j++;
            password = c->argv[j]->ptr;
        } else if (!strcasecmp(c->argv[j]->ptr,"keys")) { // 如果设置了keys参数，表明要同时传输多个keys值过去
            if (sdslen(c->argv[3]->ptr) != 0) { // 如果开启了keys选项，前面key参数的位置必须设置为空
                addReplyError(c,
                    "When using MIGRATE KEYS option, the key argument"
                    " must be set to the empty string");
                return;
            }
            first_key = j+1;
            num_keys = c->argc - j - 1;
            break; /*现在first_key值指向keys的第一个值.，并将num_keys设置为keys的数量 */
        } else {
            addReply(c,shared.syntaxerr);
            return;
        }
    }

    /* 选择的db和超时时间数据校验，看是否是合法的数字格式 */
    if (getLongFromObjectOrReply(c,c->argv[5],&timeout,NULL) != C_OK ||
        getLongFromObjectOrReply(c,c->argv[4],&dbid,NULL) != C_OK)
    {
        return;
    }
    if (timeout <= 0) timeout = 1000;

    /* 接下来会检查是否有可以迁移的键 */
    ov = zrealloc(ov,sizeof(robj*)*num_keys);
    kv = zrealloc(kv,sizeof(robj*)*num_keys);
    int oi = 0;

   /* 检查所有的键，判断输入的键中，是否存在合法的键来进行迁移 */
    for (j = 0; j < num_keys; j++) {
        if ((ov[oi] = lookupKeyRead(c->db,c->argv[first_key+j])) != NULL) { // 去键空间字典中查找该键，如果该键没有超时
            kv[oi] = c->argv[first_key+j]; // 将未超时的键存到kv数组中，说明当前key是可以migrate的；否则如果超时就无法进行migrate
            oi++;
        }
    }
    num_keys = oi; // 更新当前可migrate的key总量
    if (num_keys == 0) { // 如果没有可以迁移的key，那么给客户端返回“NOKEY"字符串
        zfree(ov); zfree(kv);
        addReplySds(c,sdsnew("+NOKEY\r\n"));
        return;
    }
```
刚开始执行migrate命令的时候，由于migrate参数很多，需要对其逐个做校验。尤其是在启用keys参数同时迁移多个keys的时候，需要进行参数的动态判断。同时需要判断是否有合法的键来进行迁移。只有没有过期的键才能够迁移，否则不进行迁，最大化节省系统资源。
### 连接建立
假如我们要从当前6379端口上的redis实例迁移到6380端口上的redis实例，我们必然要建立一个socket连接：
```c
try_again:
    write_error = 0;

    /* 连接建立 */
    cs = migrateGetSocket(c,c->argv[1],c->argv[2],timeout);
    if (cs == NULL) {
        zfree(ov); zfree(kv);
        return; 
    }
```
我们看到，在主流程中调用了migrateGetSocket()函数创建了一个socket，这里是一个带缓存的socket。我们暂时不跟进这个函数，后面我会以扩展的形式来跟进。
### 组装命令
基于这个socket，我们可以将数据以TCP协议中规定的字节流形式传输到目标实例上。这就需要一个序列化的过程了。6379实例需要将keys序列化，6380需要将数据反序列化。这就需要借助我们之前讲过的DUMP命令和RESTORE命令，分别来进行序列化和反序列化了。
redis并没有立即进行DUMP将key序列化，而是首先组装要在目标redis实例上所要执行的命令，比如AUTH/SELECT/RESTORE等命令。要想在目标实例上执行命令，那么必须同样基于之前建立的socket连接，以当前的redis实例作为客户端，往与目标redis实例建立的TCP连接中，写入按照redis协议封装的命令集合（如\*2 \r\n SELECT \r\n $1 \r\n 1 \r\n）。redis使用了自己封装的I/O抽象层rio，通过它就可以往我们在建立socket的时候生成的fd中写入数据啦。首先redis建立一个rio缓冲区，并按照redis协议来组装要在目标实例上执行的redis命令：
```c
    // 初始化一个rio缓冲区
    rioInitWithBuffer(&cmd,sdsempty());

    /* 组装AUTH命令 */
    if (password) {
        serverAssertWithInfo(c,NULL,rioWriteBulkCount(&cmd,'*',2)); // 按照redis协议写入一条命令开始的标识\*2。表示命令一共有2个参数
        serverAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,"AUTH",4)); // 写入$4\r\n  AUTH \r\n
        serverAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,password, sdslen(password))); // 同上，按照协议格式写入密码
    }

    /* 在目标实例上选择数据库 */
    int select = cs->last_dbid != dbid; /* 判断是否已经选择过数据库，如果选择过就不用再次执行SELECT命令 */
    if (select) { // 如果没有选择过，需要执行SELECT命令选择数据库
        serverAssertWithInfo(c,NULL,rioWriteBulkCount(&cmd,'*',2)); // 同上，写入开始表示\*2
        serverAssertWithInfo(c,NULL,rioWriteBulkString(&cmd,"SELECT",6)); // 同上，写入$6\r\n SELECT \r\n
        serverAssertWithInfo(c,NULL,rioWriteBulkLongLong(&cmd,dbid)); // 写入$1\r\n 1 \r\n
    }
```
那么接下来需要进行DUMP的序列化操作了。由于序列化操作耗时较久，所以可能出现这种情况：在之前第一次检测是否超时的时候没有超时，但是由于这次序列化操作时间较久，执行期间，这个键超时了，那么redis简单粗暴地丢弃该超时键，直接放弃迁移这个键：
```c
    int non_expired = 0; // 暂存新的未过期的键的数量

    /* 如果在DUMP的过程中过期了，直接continue. */
    for (j = 0; j < num_keys; j++) {
        long long ttl = 0;
        long long expireat = getExpire(c->db,kv[j]);

        if (expireat != -1) {
            ttl = expireat-mstime();
            if (ttl < 0) {
                continue;
            }
            if (ttl < 1) ttl = 1;
        }

        /* 经过上面的筛选之后，都是最新的、没有过期的键，这些键可以最终被迁移了. */
        kv[non_expired++] = kv[j];
```