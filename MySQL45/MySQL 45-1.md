# MySQL 45-1

## MySQL基本架构示意图

![截屏2020-07-03下午3.08.40](https://raw.githubusercontent.com/Elton123456/img_Repository/master/MySQL%E9%80%BB%E8%BE%91%E6%9E%B6%E6%9E%84%E5%9B%BE.png)

MySQL大致分为Server层和存储引擎两部分。

> Server层涵盖MySQL的大多数核心服务功能，及所有内置函数（如日期、时间、数学、加密函数等），所有跨存储引擎的功能在该层实现，如存储过程、触发器、试图等。

> 存储引擎层负责数据的存储和提取。_架构模式是插件式的_，支持InnoDB（5.5.5版本后默认存储引擎）、MyISAM、Memory等多个存储引擎。

__从图可看出，不同的存储引擎共用一个Server层__



### 连接器

负责数据库与客户端建立连接、获取权限、维持和管理连接。

连接命令：

```mssql
mysql -h$ip -P$port -u$user -p
```

过程：

1.TCP握手

2.认证身份，用户名和密码

​	-认证失败，“Access denied for user”，客户端程序结束执行；

​	-认证通过，连接器在权限表中查权限，之后该连接中的权限判断逻辑，都将依	赖于此时读到的权限；

__（即一个用户建立连接后，即使管理员对该用户权限做了修改，也不会影响已存在的连接权限，修改完后的权限只有再新建的连接才会生效）__



连接后若无后续动作，则该连接处于空闲状态，show processlist命令中显示为Sleep的即为空闲连接，长时间没有动静，连接器会将其自动断开，该时间由wait_timeout控制，默认8小时。

当连接被断开后，再请求，则会收到“ Lost connection to MySQL server during query。” 需要重连，然后再执行请求。

##### 数据库中的长连接和短连接

> 长连接：连接成功后，如果客户端持续有请求，则一直使用同一个连接。
>
> 短连接：连接成功后，每次执行完很少几次查询就断开连接，下次查询再重新建立一个。

__建立连接通常很复杂，所以尽可能的使用长连接__

当全部使用长连接后，会发现MySQL占用内存涨得很快，是因为MySQL执行过程中临时使用的内存是管理在连接对象里面的。这些资源在连接断开时才释放，所以长连接累积下来导致内存占用太大，被系统强行杀掉（OOM），表现为MySQL重启。

__解决方案__ :

1.定期断开长连接，或判断程序里执行过一个占用内存大的查询，则断开连接，之后查询再重连。

2.如果你用的是MySQL 5.7或更新版本，可以在每次执行一个比较大的操作后，通过执行 mysql_reset_connection来重新初始化连接资源。这个过程不需要重连和重新做权限验证，但是会将连接恢复到刚刚创建完时的状态。

### 查询缓存

连接建立后，执行select语句，执行逻辑到了查询缓存。

MySQL拿到查询请求后，会先到查询缓存判断是否之前执行过。之前执行过的语句及结果会以key-value对的形式，被直接缓存在内存中。其中key为查询语句，value为查询结果。如果在缓存中找打key，则直接讲value返回客户端。若不在缓存中，则继续后面的执行阶段。执行后，执行结果将存入查询缓存中。

__但查询缓存往往弊大于利，所以通常情况下不建议使用__

查询缓存失效非常频繁，只要有对一个表更新，该表上所以的查询缓存都会被清空。所以对于更新频繁的数据库，查询缓存的命中率很低。当业务是静态表，很久才更新一次时，该表的查询可以用查询缓存。

### 分析器

接没有命中查询缓存，开始执行语句。分析器会对SQL语句做“词法分析”，进而使MySQL知道用户要“干什么”，比如识别select、表名、列名等。之后做“语法分析”，根据语法规则，判断是否满足MySQL语法。如果语法不对，会显示：“You have an error in your SQL syntax”的错误提醒。

### 优化器

经过分析器使MySQL知道要做什么，准备执行之前，还需经过优化器处理。

优化器在表中有多个索引时，决定使用哪个索引；或一个表有多表关联（join）时，决定各表的连接顺序。

当有两种及以上匹配方案时，优化器决定才用哪种方案。

### 执行器

执行语句前先判断是否有执行的权限，若没有则返回没有权限的错误，（在工程实现上，若命中查询缓存，会在查询缓存返回结果时，做权限验证。查询也会在优化器之前调用precheck验证权限）

若有权限则继续根据表的引擎定义，使用该引擎提供的接口。

__引擎扫描行数和rows_examined（表述执行语句执行过程中扫描了多少行）并不是完全相同。因为有些场景下，执行器调用一次，在引擎内部扫描了多行__

