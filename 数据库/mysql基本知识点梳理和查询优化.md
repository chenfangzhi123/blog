本文主要是总结了工作中一些常用的操作，以及不合理的操作，在对慢查询进行优化时收集的一些有用的资料和信息，本文适合有mysql基础的开发人员。 

## 一、索引相关

1. 索引基数：基数是数据列所包含的不同值的数量。例如，某个数据列包含值1、3、7、4、7、3，那么它的基数就是4。索引的基数相对于数据表行数较高（也就是说，列中包含很多不同的值，重复的值很少）的时候，它的工作效果最好。如果某数据列含有很多不同的年龄，索引会很快地分辨数据行。如果某个数据列用于记录性别（只有"M"和"F"两种值），那么索引的用处就不大。如果值出现的几率几乎相等，那么无论搜索哪个值都可能得到一半的数据行。在这些情况下，最好根本不要使用索引，因为查询优化器发现某个值出现在表的数据行中的百分比很高的时候，它一般会忽略索引，进行全表扫描。惯用的百分比界线是"30%"。
5. **索引失效原因**：
    > 1. 对索引列运算，运算包括（+、-、*、/、！、<>、%、like'%_'（%放在前面）
    > 2. 类型错误，如字段类型为varchar，where条件用number。
    > 3. 对索引应用内部函数，这种情况下应该建立基于函数的索引
    >    如select * from template t  where ROUND(t.logicdb_id) = 1   
    >    此时应该建ROUND(t.logicdb_id)为索引，mysql8.0开始支持函数索引，5.7可以通过虚拟列的方式来支持，之前只能新建一个ROUND(t.logicdb_id)列然后去维护
    > 4. 如果条件有or，即使其中有条件带索引也不会使用（这也是为什么建议少使用or的原因），如果想使用or，又想索引有效，只能将or条件中的每个列加上索引
    > 5. 如果列类型是字符串，那一定要在条件中数据使用引号，否则不使用索引；
    > 6. B-tree索引 is null不会走,is not null会走,位图索引 is null,is not null 都会走 
    > 7. 组合索引遵循最左原则

**索引的建立**

1. 最重要的肯定是根据业务经常查询的语句
2. 尽量选择区分度高的列作为索引，区分度的公式是 COUNT(DISTINCT col) / COUNT(*)。表示字段不重复的比率，比率越大我们扫描的记录数就越少
3. 如果业务中唯一特性最好建立唯一键，一方面可以保证数据的正确性，另一方面索引的效率能大大提高


## 二、EXPLIAN中有用的信息

**基本用法**

1. desc 或者 explain 加上你的sql
2. extended explain加上你的sql，然后通过show warnings可以查看实际执行的语句，这一点也是非常有用的，很多时候不同的写法经过sql分析之后实际执行的代码是一样的


**提高性能的特性**   

1. **索引覆盖(covering index)**：需要查询的数据在索引上都可以查到不需要回表 EXTRA列显示using index
2. **ICP特性(Index Condition Pushdown)**：本来index仅仅是data access的一种访问模式，存数引擎通过索引回表获取的数据会传递到MySQL server层进行where条件过滤,5.6版本开始当ICP打开时，如果部分where条件能使用索引的字段，MySQL server会把这部分下推到引擎层，可以利用index过滤的where条件在存储引擎层进行数据过滤。EXTRA显示using index condition。需要了解mysql的架构图分为server和存储引擎层
3. **索引合并(index merge)**：对多个索引分别进行条件扫描，然后将它们各自的结果进行合并(intersect/union)。一般用OR会用到，如果是AND条件，考虑建立复合索引。EXPLAIN显示的索引类型会显示index_merge，EXTRA会显示具体的合并算法和用到的索引


**extra字段**
1、using filesort： 说明MySQL会对数据使用一个外部的索引排序，而不是按照表内的索引顺序进行读取。MySQL中无法利用索引完成的排序操作称为“文件排序”  ，其实不一定是文件排序，内部使用的是快排
2、using temporary：  使用了临时表保存中间结果，MySQL在对查询结果排序时使用临时表。常见于排序order by和分组查询group by   
3、using index： 表示相应的SELECT操作中使用了覆盖索引（Covering Index），避免访问了表的数据行，效率不错。    
6、impossible where： WHERE子句的值总是false，不能用来获取任何元组   
7、select tables optimized away： 在没有GROUP BY子句的情况下基于索引优化MIN/MAX操作或者对于MyISAM存储引擎优化COUNT(*)操作， 不必等到执行阶段再进行计算，查询执行计划生成的阶段即完成优化   
8、distinct： 优化distinct操作，在找到第一匹配的元祖后即停止找同样值的操作   

> using filesort,using temporary这两项出现时需要注意下，这两项是十分耗费性能的，在使用group by的时候，虽然没有使用order by，如果没有索引，是可能同时出现using filesort,using temporary的，因为group by就是先排序在分组，如果没有排序的需要，可以加上一个order by NULL来避免排序，这样using filesort就会去除，能提升一点性能。

**type字段**
system：表只有一行记录（等于系统表），这是const类型的特例，平时不会出现   
const：如果通过索引依次就找到了，const用于比较主键索引或者unique索引。 因为只能匹配一行数据，所以很快。如果将主键置于where列表中，MySQL就能将该查询转换为一个常量   
eq_ref：唯一性索引扫描，对于每个索引键，表中只有一条记录与之匹配。常见于主键或唯一索引扫描   
ref：非唯一性索引扫描，返回匹配某个单独值的所有行。本质上也是一种索引访问，它返回所有匹配 某个单独值的行，然而它可能会找到多个符合条件的行，所以它应该属于查找和扫描的混合体   
range：只检索给定范围的行，使用一个索引来选择行。key列显示使用了哪个索引，一般就是在你的where语句中出现between、<、>、in等的查询，这种范围扫描索引比全表扫描要好，因为只需要开始于缩印的某一点，而结束于另一点，不用扫描全部索引   
index：Full Index Scan ，index与ALL的区别为index类型只遍历索引树，这通常比ALL快，因为索引文件通常比数据文件小。 （也就是说虽然ALL和index都是读全表， 但index是从索引中读取的，而ALL是从硬盘读取的）   
all：Full Table Scan，遍历全表获得匹配的行   

参考地址：https://blog.csdn.net/DrDanger/article/details/79092808


## 三、字段类型和编码

1. mysql返回字符串长度：CHARACTER_LENGTH方法(CHAR_LENGTH一样的)返回的是字符数，LENGTH函数返回的是字节数，一个汉字三个字节
2. varvhar等字段建立索引长度计算语句：select count(distinct left(test,5))/count(*) from table;   越趋近1越好
3. mysql的utf8最大是3个字节不支持emoji表情符号，必须只用utf8mb4。需要在mysql配置文件中配置客户端字符集为utf8mb4。**jdbc的连接串不支持配置characterEncoding=utf8mb4，最好的办法是在连接池中指定初始化sql，例如：hikari连接池，其他连接池类似spring.datasource.hikari.connection-init-sql=set names utf8mb4。否则需要每次执行sql前都先执行set names utf8mb4**。    
4. msyql排序规则(一般使用_bin和_genera_ci)：
    1. utf8_genera_ci不区分大小写，ci为case insensitive的缩写，即大小写不敏感，
    2. utf8_general_cs区分大小写，cs为case sensitive的缩写，即大小写敏感，但是目前MySQL版本中已经不支持类似于***_genera_cs的排序规则，直接使用utf8_bin替代。
    3. utf8_bin将字符串中的每一个字符用二进制数据存储，区分大小写。
        > 那么，同样是区分大小写，utf8_general_cs和utf8_bin有什么区别？
        > cs为case sensitive的缩写，即大小写敏感；bin的意思是二进制，也就是二进制编码比较。
        > utf8_general_cs排序规则下，即便是区分了大小写，但是某些西欧的字符和拉丁字符是不区分的，比如ä=a，但是有时并不需要ä=a，所以才有utf8_bin
        > utf8_bin的特点在于使用字符的二进制的编码进行运算，任何不同的二进制编码都是不同的，因此在utf8_bin排序规则下：ä<>a
5. sql yog中初始连接指定编码类型使用连接配置的初始化命令 
![初始连接编码配置](http://cdn.jiangmiantex.com/sqlyog%E5%AD%97%E7%AC%A6%E7%BC%96%E7%A0%81%E9%85%8D%E7%BD%AE.png)

## 四、SQL语句总结

**常用的但容易忘的**：    

1. 如果有主键或者唯一键冲突则不插入：insert ignore into    
2. 如果有主键或者唯一键冲突则更新,注意这个会影响自增的增量：INSERT INTO `room_remarks`(room_id,room_remarks) VALUE(1,"sdf") ON DUPLICATE KEY UPDATE room_remarks="234"     
3. 如果有就用新的替代，values如果不包含自增列，自增列的值会变化： REPLACE INTO `room_remarks`(room_id,room_remarks) VALUE(1,"sdf")      
4. 备份表：CREATE TABLE user_info SELECT * FROM user_info    
5. 复制表结构：CREATE TABLE user_v2 LIKE user 
6. 从查询语句中导入：INSERT INTO user_v2 SELECT * FROM user或者INSERT INTO user_v2(id,num) SELECT id,num FROM user   
7. 连表更新：UPDATE user a, room b SET a.num=a.num+1 WHERE a.room_id=b.id     
8. 连表删除：DELETE user FROM user,black WHERE user.id=black.id 

**锁相关**(作为了解，很少用)

1. 共享锁：  select id from tb_test where id = 1 lock **in share mode**;      
2. 排它锁： select id from tb_test where id = 1 **for update** 


**优化时用到**：   

1. 强制使用某个索引： select * from table force index(idx_user) limit 2;    
2. 禁止使用某个索引： select * from table ignore index(idx_user) limit 2;   
3. 禁用缓存(在测试时去除缓存的影响)： select SQL_NO_CACHE from  table limit 2;   


**查看状态**
1. 查看字符集  SHOW VARIABLES LIKE 'character_set%';
2. 查看排序规则  SHOW VARIABLES LIKE 'collation%';

**SQL编写注意**
1. where语句的解析顺序是从右到左，条件尽量放where不要放having
2. 采用<font id="yanchi" href="#" color="red">延迟关联</font>(deferred join)技术优化超多分页场景，比如limit 10000,10,延迟关联可以避免回表 
1. distinct语句非常损耗性能，可以通过group by来优化
2. 连表尽量不要超过三个表


## 五、踩坑

1. 如果有自增列，truncate语句会把自增列的基数重置为0，有些场景用自增列作为业务上的id需要十分重视
2. 聚合函数会自动滤空，比如a列的类型是int且全部是NULL，则SUM(a)返回的是NULL而不是0
3. mysql判断null相等不能用“a=null”,这个结果永远为UnKnown，where和having中,UnKnown永远被视为false，check约束中，UnKnown就会视为true来处理。所以要用“a is null”处理

## 六、千万大表在线修改

mysql在表数据量很大的时候，如果修改表结构会导致锁表，业务请求被阻塞。mysql在5.6之后引入了在线更新，但是在某些情况下还是会锁表，所以一般都采用pt工具( Percona Toolkit)    

如对表添加索引：
```
pt-online-schema-change --user='root' --host='localhost' --ask-pass --alter "add index idx_user_id(room_id,create_time)" D=fission_show_room_v2,t=room_favorite_info --execute
```

## 七、慢查询日志

有时候如果线上请求超时，应该去关注下慢查询日志，慢查询的分析很简单，先找到慢查询日志文件的位置，然后利用mysqldumpslow去分析。查询慢查询日志信息可以直接通过执行sql命令查看相关变量，常用的sql如下：

```sql
-- 查看慢查询配置
-- slow_query_log  慢查询日志是否开启
-- slow_query_log_file 的值是记录的慢查询日志到文件中
-- long_query_time 指定了慢查询的阈值
-- log_queries_not_using_indexes 是否记录所有没有利用索引的查询
SHOW VARIABLES LIKE '%quer%';

-- 查看慢查询是日志还是表的形式
SHOW VARIABLES LIKE 'log_output'

-- 查看慢查询的数量
SHOW GLOBAL STATUS LIKE 'slow_queries';
```

mysqldumpslow的工具十分简单，我主要用到的是参数如下：

-t:限制输出的行数，我一般取前十条就够了
-s:根据什么来排序默认是平均查询时间at，我还经常用到c查询次数，因为查询次数很频繁但是时间不高也是有必要优化的，还有t查询时间，查看那个语句特别卡。
-v:输出详细信息

例子：<code>mysqldumpslow -v -s t -t 10  mysql_slow.log.2018-11-20-0500</code>

## 八、查看sql进程和杀死进程

如果你执行了一个sql的操作，但是迟迟没有返回，你可以通过查询进程列表看看他的实际执行状况，如果该sql十分耗时，为了避免影响线上可以用kill命令杀死进程，通过查看进程列表也能直观的看下当前sql的执行状态，如果当前数据库负载很高，在进程列表可能会出现，大量的进程夯住，执行时间很长。命令如下：

```sql
--查看进程列表
SHOW PROCESSLIST;
--杀死某个进程
kill 183665
```

如果你使用的sqlyog，那么也有图形化的页面，在菜单栏-工具-显示-进程列表。在进程列表页面可以右键杀死进程。如下所示：    

![查看进程列表](http://cdn.jiangmiantex.com/%E8%BF%9B%E7%A8%8B%E5%88%97%E8%A1%A8.png)

![杀死进程](http://cdn.jiangmiantex.com/%E6%9D%80%E6%AD%BB%E8%BF%9B%E7%A8%8B.png)

## 九、一些数据库性能的思考

在对公司慢查询日志做优化的时候，很多时候可能是忘了建索引，像这种问题很容易解决，加个索引就行了。但是有两种情况就不是简单能加索引能解决了：
1. 业务代码循环读数据库： 考虑这样一个场景，获取用户粉丝列表信息  加入分页是十个  其实像这样的sql是十分简单的，通过连表查询性能也很高，但是有时候，很多开发采用了取出一串id，然后循环读每个id的信息，这样如果id很多对数据库的压力是很大的，而且性能也很低
2. 统计sql：很多时候，业务上都会有排行榜这种，发现公司有很多地方直接采用数据库做计算，在对一些大表的做聚合运算的时候，经常超过五秒，这些sql一般很长而且很难优化， 像这种场景，如果业务允许（比如一致性要求不高或者是隔一段时间才统计的），可以专门在从库里面做统计。另外我建议还是采用redis缓存来处理这种业务
3. 超大分页: 在慢查询日志中发现了一些超大分页的慢查询如<code> limit  40000,1000</code>，因为mysql的分页是在server层做的，可以采用[延迟关联](#yanchi)在减少回表。但是看了相关的业务代码正常的业务逻辑是不会出现这样的请求的，所以很有可能是有恶意用户在刷接口，所以最好在开发的时候也对接口加上校验拦截这些恶意请求。
