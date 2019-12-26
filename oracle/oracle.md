* 

* 查看用户信息

  * select * from all_users;
  * select * from dba_users;

* 查看表空间
  * select tablespace_name from dba_tablespaces;
  * select tablespace_name from user_tablespaces;

* 修改用户名

  * 用sysdba账号登入数据库，然后查询到要更改的用户信息：
    *  SELECT user#,name FROM user$;

  * 更改用户名并提交：
    *  UPDATE USER$ SET NAME='PORTAL' WHERE user#=88;
    *  COMMIT;
  * 强制刷新：
    * ALTER SYSTEM CHECKPOINT;
    * ALTER SYSTEM FLUSH SHARED_POOL;
  * 更新用户的密码：
    * ALTER USER PORTAL IDENTIFIED BY 123;

  ###### 重启数据库与监听器

  * 以root用户
    * cd $ORACLE_HOME/bin #进入到oracle的安装目录 
    * ./dbstart #重启服务器 
    * ./lsnrctl start #重启监听器
  * 以oracle用户
    * su - oracle
    * sqlplus /nolog  #进入sqlplus控制台
    * connect / as sysdba  #以管理员登录
    * startup  #启动数据库
    * shutdown immediate  # 停止数据库
    * exit  #退出控制台
    * lsnrct  #进入监听器控制台
    * start #启动
    * exit #退出

* 登录失败

  * insufficient privileges

    * sqlnet.ora ==>加入SQLNET.AUTHENTICATION_SERVICES=(NTS)

  * credential retrieval

    * 开始 -> 程序 -> Oracle -> Configuration and Migration Tools ->
      Net Manager→本地→概要文件→Oracle高级安全性→验证→去掉所选方法中的 "NTS" 就可以了.
    * SQLNET.AUTHENTICATION_SERVICES= (NONE)

  * 修改密码

    * alter user system identified by system

* 查看锁表及解锁

  * **--以下几个为相关表**

    SELECT * FROM v$lock;
    SELECT * FROM v$sqlarea;
    SELECT * FROM v$session;
    SELECT * FROM v$process ;
    SELECT * FROM v$locked_object;
    SELECT * FROM all_objects;
    SELECT * FROM v$session_wait;

  * --查看被锁的表 
    select b.owner,b.object_name,a.session_id,a.locked_mode from v$locked_object a,dba_objects b where b.object_id = a.object_id;

  * --查看那个用户那个进程照成死锁
    select b.username,b.sid,b.serial#,logon_time from v$locked_object a,v$session b where a.session_id = b.sid order by b.logon_time;

  * --查看连接的进程 
    SELECT sid, serial#, username, osuser FROM v$session;

  * **--查出锁定表的sid, serial#,os_user_name, machine_name, terminal，锁的type,mode**

    SELECT s.sid, s.serial#, s.username, s.schemaname, s.osuser, s.process, s.machine,
    s.terminal, s.logon_time, l.type
    FROM v$session s, v$lock l
    WHERE s.sid = l.sid
    AND s.username IS NOT NULL
    ORDER BY sid;

    这个语句将查找到数据库中所有的DML语句产生的锁，还可以发现，
    任何DML语句其实产生了两个锁，一个是表锁，一个是行锁。

  * --杀掉进程 sid,serial#
    alter system kill session'210,11562';

* 查看表空间大小

  * SELECT a.tablespace_name                        "表空间名",
           total                                    "表空间大小",
           free                                     "表空间剩余大小",
           ( total - free )                         "表空间使用大小",
           Round(( total - free ) / total, 4) * 100 "使用率   %"
    FROM   (SELECT tablespace_name,
                   Sum(bytes) free
            FROM   DBA_FREE_SPACE
            GROUP  BY tablespace_name) a,
           (SELECT tablespace_name,
                   Sum(bytes) total
            FROM   DBA_DATA_FILES
            GROUP  BY tablespace_name) b
    WHERE  a.tablespace_name = b.tablespace_name

###### oralce误删除数据

- delete误删除的解决方法

  - 利用oracle提供的闪回方法，如果在删除数据后还没做大量的操作（只要保证被删除数据的块没被覆写），就可以利用闪回方式直接找回删除的数据
    具体步骤为：

    *确定删除数据的时间（在删除数据之前的时间就行，不过最好是删除数据的时间点）

    *用以下语句找出删除的数据：select * from 表名 as of timestamp to_timestamp('删除时间点','yyyy-mm-dd hh24:mi:ss')

    *把删除的数据重新插入原表：

         insert into 表名 (select * from 表名 as of timestamp to_timestamp('删除时间点','yyyy-mm-dd hh24:mi:ss'));注意要保证主键不重复。

    如果表结构没有发生改变，还可以直接使用闪回整个表的方式来恢复数据。

    具体步骤为：

    表闪回要求用户必须要有flash any table权限

     

     --开启行移动功能 
     ·alter table 表名 enable row movement

     --恢复表数据
     ·flashback table 表名 to timestamp to_timestamp(删除时间点','yyyy-mm-dd hh24:mi:ss')

     --关闭行移动功能 ( 千万别忘记 )

     ·alter table 表名 disable row movement

- drop误删除的解决方法

  - 原理：由于oracle在删除表时，没有直接清空表所占的块,oracle把这些已删除的表的信息放到了一个虚拟容器“回收站”中，而只是对该表的数据块做了可以被覆写的标志，所以在块未被重新使用前还可以恢复。

    具体步骤：

    *查询这个“回收站”或者查询user_table视图来查找已被删除的表:

     · select table_name,dropped from user_tables

     · select object_name,original_name,type,droptime from user_recyclebin

    在以上信息中，表名都是被重命名过的，字段table_name或者object_name就是删除后在回收站中的存放表名

    *如果还能记住表名，则可以用下面语句直接恢复：

      flashback table 原表名 to before drop

     如果记不住了，也可以直接使用回收站的表名进行恢复，然后再重命名，参照以下语句：

      flashback table "回收站中的表名(如：Bin$DSbdfd4rdfdfdfegdfsf==$0)" to before drop rename to 新表名

    oracle的闪回功能除了以上基本功能外，还可以闪回整个数据库：

    使用数据库闪回功能，可以使数据库回到过去某一状态, 语法如下：

    SQL>alter database flashback on
    SQL>flashback database to scn SCNNO;
    SQL>flashback database to timestamp to_timestamp('2007-2-12 12:00:00','yyyy-mm-dd hh24:mi:ss');

- 完全删除表

  - oracle提供以上机制保证了安全操作，但同时也代来了另外一个问题，就是空间占用，由于以上机制的运行，使用drop一个表或者delete数据后，空间不会自

    动回收，对于一些确定不使用的表，删除时要同时回收空间，可以有以下2种方式：

      1、采用truncate方式进行截断。（但不能进行数据回恢复了）

      2、在drop时加上purge选项：drop table 表名 purge

         该选项还有以下用途：

      也可以通过删除recyclebin区域来永久性删除表 ,原始删除表drop table emp cascade constraints
       purge table emp;
       删除当前用户的回收站:
        purge recyclebin;
       删除全体用户在回收站的数据:
       purge dba_recyclebin

* 锁表解锁

	* 查询被锁的表

		* select t2.username,
			       t2.sid,
			       t2.serial#,
			       t3.object_name,
			       t2.OSUSER,
			       t2.MACHINE,
			       t2.PROGRAM,
			       t2.LOGON_TIME,
			       t2.COMMAND,
			       t2.LOCKWAIT,
			       t2.SADDR,
			       t2.PADDR,
			       t2.TADDR,
			       t2.SQL_ADDRESS,
			       t1.LOCKED_MODE
			  from v$locked_object t1, v$session t2, dba_objects t3
			 where t1.session_id = t2.sid
			   and t1.object_id = t3.object_id
			 order by t2.logon_time;

	* 使用拥有管理员权限用户执行

		* alter system kill session 'sid,seial#';

###### 汉字排序

*  **ORDER BY NLSSORT(NAME, 'NLS_SORT=SCHINESE_PINYIN_M');**
	* **SCHINESE_RADICAL_M** 按照部首（第一顺序）、笔划（第二顺序）排序 
	* **SCHINESE_STROKE_M** 按照笔划（第一顺序）、部首（第二顺序）排序 
	* **SCHINESE_PINYIN_M** 按照拼音排序，系统的默认排序方式为拼音排序

###### 删除用户和表空间

* drop user jsszcollect cascade
* drop tablespace jsszcollect including contents and datafiles

###### 查询表空间

* select * from user_tablespaces;
* select * from dba_data_files;

###### 创建表空间

* create tablespace jssz  
	logging  
	datafile '/home/oracle/oracle/product/11.2.0/db_1/dbs/jssz.dbf' 
	size 50m  
	autoextend on  
	next 50m maxsize 20480m  
	extent management local; 