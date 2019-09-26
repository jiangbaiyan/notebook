baiyan

## 命令语法
命令含义：将 key 原子性地从当前实例传送到目标实例的指定数据库上，一旦传送成功， key 保证会出现在目标实例上，而当前实例上的 key 会被删除
命令格式
```c
MIGRATE host port key|"" destination-db timeout [COPY] [REPLACE] [KEYS key [key ...]]
```
使用实战：将键key1迁移到本机6380端口的redis实例上，存储到目标实例的第0号数据库，超时时间为1000毫秒。可选项COPY如果表示不移除源实例上的 key ，REPLACE表示替换目标实例上已存在的 key 。keys表示可以同时传送多个keys（前面的key参数的位置必须设置为空）
```c
127.0.0.1:6379> migrate 127.0.0.1 6380 key1 0 1000
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