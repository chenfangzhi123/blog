# mysql利用binlog进行数据恢复

最近线上误操作了一个数据，由于是直接修改的数据库，所有唯一的恢复方式就在mysql的binlog。binlog使用的是ROW模式，即受影响的每条记录都会生成一个sql。同时利用了[binlog2sql](https://github.com/danfengcao/binlog2sql)项目。
[TOC]

## binlog基本配置和格式

### binlog基本配置
binlog需要在mysql的配置文件的mysqld节点中进行配置：

```php
# 日志中的Serverid
server-id		= 1
# 日志路径
log_bin			= /var/log/mysql/mysql-bin.log
# 保存几天的日志
expire_logs_days	= 10
# 每个binlog的大小
max_binlog_size   = 1000M
#binlgo模式
binlog_format=ROW
# 默认是所有记录，可以配置哪些需要记录，哪些不记录
#binlog_do_db		= include_database_name
#binlog_ignore_db	= include_database_name
```
### 查看binlog状态
1. SHOW BINARY LOGS; 查看binlog文件
2. SHOW VARIABLES LIKE '%log_bin%'  查看日志状态
3. SHOW MASTER STATUS  查看日志文件位置

### binlog的三种格式
1.ROW    
针对行记录日志，每行修改产生一条记录。   
优点：上下文信息比较全，恢复某条误操作时可以直接在日志中查找到原文信息，对于主从复制支持好。   
缺点：输出非常大，如果是Alter语句将产生大量的记录   

格式如下：
```sql
DELETE FROM `back`.`sys_user` WHERE `deptid`=27 AND `status`=1 AND `account`='admin' AND `name`='张三' AND `phone`='18200000000' AND `roleid`='1' AND `createtime`='2016-01-29 08:49:53' AND `sex`=2 AND `email`='sn93@qq.com' AND `birthday`='2017-05-05 00:00:00' AND `avatar`='girl.gif' AND `version`=25 AND `password`='ecfadcde9305f8891bcfe5a1e28c253e' AND `salt`='8pgby' AND `id`=1 LIMIT 1; #start 4 end 796 time 2018-10-12 17:03:19
```

2.STATEMENT    
针对sql语句的，每条语句产生一条记录   
优点：产生的日志量比较小，主从版本可以不一致   
缺点：主从有些语句不能支持，像自增主键和UUID这种类型的

格式如下：
```sql
delete from `sys_role`;
```

3.MIX    
结合了两种的优点，一般情况下都采用STATEMENT模式，对于不支持的语句采用ROW模式

## 转换成sql

### mysql自带的mysqlbinlog
由于binlog是二进制的，所以需要先转换成文本文件，一般可以采用Mysql自带的mysqlbinlog转换成文本。

```shell
mysqlbinlog --no-defaults --base64-output='decode-rows' -d room -v mysql-bin.011012 > /root/binlog_2018-10-10
```

> 参数说明
> 1. --no-defaults 为了防止报错：mysqlbinlog: unknown variable 'default_character_set=utf8mb4'
> 2. --base64-output='decode-rows' 和-v一起使用， 进行base64解码 
其他有很多用来限定范围的参数，比如数据库，起始时间，起始位置等等。这些参数在查找误操作的时候非常有用。


binlog的基本块如下：
```
# at 417750
#181007  1:50:38 server id 1630000  end_log_pos 417844 CRC32 0x9fc3e3cd 	Query	thread_id=440109962	exec_time=0	error_code=0
SET TIMESTAMP=1538877038/*!*/;
BEGIN
```

1. \# at 417750    
> 指明的当前位置相对文件开始的偏移位置，这个在mysqlbinlog命令中可以作为--start-position的参数    
2. \#181007  1:50:38 server id 1630000  end_log_pos 417844 CRC32 0x9fc3e3cd 	Query	thread_id=440109962	exec_time=0	error_code=0
> 181007  1:50:38指明时间为18年10月7号1:50:38，serverid也就是你在配置文件中的配置的，end_log_pos 417844，这个块在417844结束。thread_id执行的线程id，exec_time执行时间，error_code错误码     
2. SET TIMESTAMP=1538877038/*!*/;
BEGIN
> 具体的执行语句

一行记录产生的日志如下所示
```
# at 417750
#181010  9:50:38 server id 1630000  end_log_pos 417844 CRC32 0x9fc3e3cd 	Query	thread_id=440109962	exec_time=0	error_code=0
SET TIMESTAMP=1539136238/*!*/;
BEGIN
/*!*/;
# at 417844
#181010  9:50:38 server id 1630000  end_log_pos 417930 CRC32 0xce36551b 	Table_map: `goods`.`good_info` mapped to number 129411
# at 417930
#181010  9:50:38 server id 1630000  end_log_pos 418030 CRC32 0x5827674a 	Update_rows: table id 129411 flags: STMT_END_F
### UPDATE `goods`.`good_info`
### WHERE
###   @1='2018:10:07' /* DATE meta=0 nullable=0 is_null=0 */
###   @2=9033404 /* INT meta=0 nullable=0 is_null=0 */
###   @3=1 /* INT meta=0 nullable=0 is_null=0 */
###   @4=8691108 /* INT meta=0 nullable=0 is_null=0 */
###   @5=9033404 /* INT meta=0 nullable=0 is_null=0 */
###   @6=20 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @7=1538877024 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
### SET
###   @1='2018:10:07' /* DATE meta=0 nullable=0 is_null=0 */
###   @2=9033404 /* INT meta=0 nullable=0 is_null=0 */
###   @3=1 /* INT meta=0 nullable=0 is_null=0 */
###   @4=8691108 /* INT meta=0 nullable=0 is_null=0 */
###   @5=9033404 /* INT meta=0 nullable=0 is_null=0 */
###   @6=21 /* LONGINT meta=0 nullable=0 is_null=0 */
###   @7=1538877024 /* TIMESTAMP(0) meta=0 nullable=0 is_null=0 */
# at 418030
#181010  9:50:38 server id 1630000  end_log_pos 418061 CRC32 0x468fb30e 	Xid = 212760460521
COMMIT/*!*/;
# at 418061
```

一行记录产生的日志如上所示。以SET TIMESTAMP=1539136238/\*!\*/;开始，以COMMIT/\*!\*/;结尾。我们可以根据两个at指明的位置来限定范围。   
注意一条记录开始的<span id="time" style="color:red">SET TIMESTAMP</span>之前的# at 417750和结尾的<span id="commit" style="color:red">COMMIT</span>之后的# at 418061

### 利用binlog2sql

binlog2sql官网介绍：从MySQL binlog解析出你要的SQL。根据不同选项，你可以得到原始SQL、回滚SQL、去除主键的INSERT SQL等。    

基本使用如下： 
```shell
python binlog2sql.py -hlocalhost  -P3306 -udev -p'\*' -d room -t  room_info --start-file='mysql-bin.011012' --start-position 129886892  --stop-position 130917280 > rollback.sql
```

具体的使用我就不讲解了github上讲解的十分清楚，主要看下很多用来筛选的条件，比如起止时间--start-datetime/--stop-datetime，表名限定-t，数据库限定-d，语句限定--sql-type，主要说说我遇到的一些问题。  
1. mysql的binlog模式
这里需要设置为ROW，因为ROW模式有原来的信息，如果可以直接利用binlog2sql反向生成回滚sql，如果是STATEMENT无法生成，需要利用的mysql定时备份的文件再去做回滚

## 恢复数据的具体操作

因为当时线上执行的是一条update语句，没有唯一键索引的。导致有两千多条记录被更新。语句如下： 

```sql
update room_info set status=1 where status=2;
```

1. 根据操作时间先定位对应的binlog文件
我记得当时操作的时间大概的是上午9多左右，所以去找对应的binlog文件最后修改时间大于9点并且时间最接近的一个文件。使用linux的<code>ll</code>命令查看文件的修改时间。
2. 筛选具体的数据库
因为一个mysql实例的所有binlog文件是在一个文件中的，所以我们先要去除其他不想关的数据库。利用-d参数来指明数据实例。然后在利用开始时间(--start-datetime)和结束时间(--stop-datetime)来进一步筛选

```shell
mysqlbinlog --no-defaults -v  --base64-output='decode-rows' -d room --start-datetime='2018-10-10 9:00:00' --stop-datetime='2018-10-10 10:00:00' mysql-bin.011012>temp.sql
```

3. 压缩取回文件分析

```shell
zip temp.zip temp.sql && sz temp.zip 
```

取回文件在本地用文本工具如vscode分析，里面有正则匹配，根据你改动过的特征，比如我有个房间号888888，这个不应该被修改，你就查看这个房间号的修改记录，ROW模式的语句是Where在前，set在后。利用正则<code>`room_id`=888888.\*`show_state`=1.\*AND `show_state`=2</code>很快就能匹配到。我当时的语句影响了两千多条记录，你根据找到的语句去找开始的<a href="#time" target="_self" style="color:red">SET TIMESTAMP=1539136238</a>的位置之前的at和结尾的<a href="#commit" target="_self" style="color:red">COMMIT</a>之后的at。

4. 利用binlog2sql生成回滚语句

python binlog2sql.py -hlocalhost  -P3306 -udev -p'\*' -d room -t  room_info -B --start-file='mysql-bin.011012' --start-position 129886892  --stop-position 130917280 > rollback.sql

## 另外

因为我这边是一条update影响多条的情况，如果是带唯一键的情况下，影响的只有一条记录，完全没必要这么麻烦，直接利用binlog2sql带上-d和-t参数限定数据库和表，然后利用grep来查找，直接可以得出对应的sql。mysqlbinlog少了一个限定表和限定语句的功能。比如精确到一张表的Delete语句，能减少很多的数据，能快速定位。