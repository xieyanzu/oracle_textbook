## 本章简介
&emsp;&emsp;本章作为数据库教材中的最后一章，主要讲解数据库的备份与恢复、SQL优化等知识。从开发的角度来讲，备份与恢复可以作为一个简单的了解内容，而SQL优化要靠读者在平日的学习、工作中不断积累，本章会从宏观方面进行介绍。

 

 

 

## 8.1  数据库备份与恢复概述

 

&emsp;&emsp;硬件、数据文件、操作系统都可能损坏或崩溃，程序员或数据维护人员也不可能永远不犯错误。因此，数据库的备份与恢复就显得格外重要。

&emsp;&emsp;备份，就是把数据库复制到转储设备的过程。恢复，就是当发生故障后，利用已备份的文件，重新建立一个完整的数据库。根据出现故障的原因，恢复分为实例恢复和介质恢复两种类型。

&emsp;&emsp;实例恢复：当Oracle实例出现意外后（如意外掉电，后台进程故障等），Oracle自动进行的恢复。

&emsp;&emsp;介质恢复：当存放数据库的介质出现故障时所做的恢复。例如，当一个文件、一个文件的一部分或一个磁盘不能读或不能写时，所进行的恢复操作。

&emsp;&emsp;DBA的主要职责之一，就是备份数据库和在数据库发生故障时高效、安全地恢复数据库。

&emsp;&emsp;备份的方法有逻辑备份、物理备份（冷备份、热备份）等多种方式；恢复的方法也有完全恢复、不完全恢复和RMAN恢复等。

### 8.1.1  逻辑备份  

&emsp;&emsp;逻辑备份是指将数据库中的用户对象导出到一个二进制文件中，需要使用导入/导出工具IMPDP/EXPDP或IMP/EXP。由于逻辑备份具有平台无关性，因此逻辑备份常作为数据迁移的主要手段。

&emsp;&emsp;本节以IMP/EXP工具为例，讲解导入及导出的具体方法。首先强调，IMP或EXP工具是直接在CMD中通过同名的IMP或EXP命令调用的，不需要登录数据库。

- 导出工具EXP

&emsp;&emsp;使用EXP执行导出操作有以下三种方式。

&emsp;&emsp;（1）表方式。导出指定的表，语法如下：

 
```
EXP  数据库用户名/密码@IP地址/实例名  file:/文件名.dmp  log=/日志文件名.log  

tables=表名1,表名2,…,表名n
```
 

&emsp;&emsp;例如，若需要导出scott用户的emp和dept表，就可以执行以下命令：

 
```
EXP  scott/tiger@127.0.0.1/orcl  file:/back/bk.dmp  log=d:/back/logfile.log  tables=emp,dept
```
 

&emsp;&emsp;这样就可以将本机中scott用户的emp表和dept表中的数据保存在f:/back/bk.dmp文件中，并且将日志内容保存在同目录中的logfile.log文件里。

&emsp;&emsp;（2）用户方式。导出指定用户的所有对象和数据，语法如下：

 
```
EXP  数据库用户名/密码@IP地址/实例名  file:/文件名.dmp   log=/日志文件名.log
```
 

&emsp;&emsp;（3）全库方式。导出数据库中的所有对象和数据，此操作需要使用DBA的角色执行，语法如下：

 
```
EXP  管理员用户名/密码@IP地址/实例名   file:/文件名.dmp   log=/日志文件名.log  full=y
```
 

- 导入工具IMP

&emsp;&emsp;执行了导出操作以后，就可以使用IMP工具执行导入操作。与导出操作相对应，导入操作也有以下三种方式。

&emsp;&emsp;（1）表方式。导入指定的表，语法如下：

 
```
IMP  数据库用户名A/密码@IP地址/实例名   file:/文件名.dmp   log=/日志文件名.log  

tables=表名1,表名2,…,表名n   fromuser=数据库用户名B  touser=数据库用户名A  

commit=y  ignore=y  
```
 

&emsp;&emsp;以上命令，就可以将用户B中用tables指定的表导入用户A中。

&emsp;&emsp;（2）用户方式。导入指定用户的所有对象和数据，语法如下：

 
```
IMP  数据库用户名A/密码@IP地址/实例名   file:/文件名.dmp   log=/日志文件名.log  

   fromuser=数据库用户名B  touser=数据库用户名A   commit=y   ignore=y  
```
 

&emsp;&emsp;（3）全库方式。导入数据库中的所有对象和数据，此操作需要使用DBA的角色执行，语法如下：

​       
```
IMP  数据库用户名A/密码@IP地址/实例名   file:/文件名.dmp   log=/日志文件名.log 

full=y  ignore=y   destroy=y
```
 

&emsp;&emsp;其中destroy=y表示的是，如果在导入时发现已经存在的数据，则将其删除后重新导入。

&emsp;&emsp;另一种导入/导出方式IMPDP/EXPDP，是从Oracle10g起引入的新方式，称为数据泵（Data Pump）。简单地讲，IMPDP/EXPDP采用了并行地进行导入/导出的方式，比IMP/EXP速度更快。

### 8.1.2  冷备份及恢复  

&emsp;&emsp;我们已经知道，物理备份可以分为冷备份和热备份。冷备份又称脱机备份，是指在关闭数据库后进行的备份；而热备份也称为联机备份，是在数据库运行的同时进行备份，但热备份存在着难以维护等风险。实际应用中，为了确保备份的可靠性，大多采用冷备份。

&emsp;&emsp;冷备份实质就是数据库相关文件的复制：在数据库关闭之后，将所有的关键性文件（数据文件、控制文件、联机文件等）复制到另外的位置。因此，冷备份最重要的一点，就是要知道这些文件的存放位置，而这些位置是可以通过超级管理员SYS账户进行查询的。

&emsp;&emsp;冷备份的具体步骤如下。

- 使用SYS登录数据库

 
```
SQLPLUS sys/change_on_install AS SYSDBA
```
 

- 查询所有控制文件的位置

 
```
SELECT * FROM v$controlfile;
```
 

- 查询所有日志文件的位置

 
```
SELECT * FROM v$logfile;
```
 

- 查询所有数据文件的位置

 
```
SELECT * FROM v$datafile;
```
 

- 查询pfile文件的位置

 
```
SHOW PARAMETER pfile;
```
 

- 记录以上所有文件的位置，之后关闭数据库实例

 
```
SHUTDOWN immediate
```
 

- 根据记录的位置，将以上所有文件复制到备份目录中

- 重新启动数据库实例

​       
```
STARTUP
```
 

&emsp;&emsp;冷备份完成以后，如果要进行恢复操作，只需要先关闭数据库实例，然后将之前备份的文件再复制回原目录中，最后重启数据库实例。

### 8.1.3  RMAN  

&emsp;&emsp;RMAN（Recovery Manager，恢复管理器）是一种用于备份和恢复数据的Oracle工具。

&emsp;&emsp;RMAN能够备份整个数据库或某些数据库部件（如表空间、数据文件、控制文件、归档文件、Spfile参数文件等）。RMAN也允许进行增量数据块级别的备份，但增量RMAN备份是时间和空间有效的，因为它们只备份自上次备份以来有变化的那些数据块。而且，通过RMAN提供的接口，可以很好地与第三方的备份与恢复软件（如VERITAS等）对接，从而提供更强大的备份与恢复功能。此外，RMAN还提供了其他更多功能，如数据库的克隆等。   

&emsp;&emsp;使用RMAN有Nocatalog和Catalog两种备份及恢复方式。

&emsp;&emsp;Nocatalog方式：用控制文件作为catalog秦桂岭1(目录)，每一次备份都要往控制文件里面写很多备份信息，因此控制文件里面会有越来越多的备份信息。

&emsp;&emsp;Catalog方式：首先要创建备份目录并建立恢复目录，即将数据库的备份信息写到恢复目录里面。

&emsp;&emsp;本节以Nocatalog方式为例，介绍使用RMAN进行备份与恢复的基本步骤。

- 使用Nocatalog方式备份数据

&emsp;&emsp;（1）查看是否开启归档模式

&emsp;&emsp;以SYSDBA角色登录，并执行以下命令：

 
```
ARCHIVELOG LIST ;
```
 

&emsp;&emsp;如果发现数据库处于非归档模式，则需要通过以下步骤改为归档模式。

&emsp;&emsp;① 关闭数据库（确保仍然处于SYSDBA角色）

 
```
SHUTDOWN immediate;
```
 

&emsp;&emsp;② 将数据库启动到MOUNT模式

​       
```
START mount;
```
 

&emsp;&emsp;③ 调整数据库的归档模式

 
```
ALTER DATABASE archivelog;
```
 

&emsp;&emsp;④ 打开数据库

 
```
ALTER DATABASE OPEN;
```
 

&emsp;&emsp;⑤ 确认数据库已处于归档模式

​       
```
ARCHIVELOG LIST ;
```
 

&emsp;&emsp;（2）设置备份时间

&emsp;&emsp;通过参数control_file_record_keep_time设置备份信息的保存时间（单位：天），超时后就自动清除掉以前的备份信息：

 
```
ALTER SYSTEM SET control_file_record_keep_time = 天数 SCOPE=BOTH ;
```
 

&emsp;&emsp;可以通过以下语句查看是否设置成功：

 
```
SHOW PARAMETER CONTROL ;
```
 

- 使用Nocatalog方式恢复数据

&emsp;&emsp;（1）建立Oracle运行环境（包括init或sp文件）

&emsp;&emsp;（2）将备份的控制文件复制到init文件指定的位置

&emsp;&emsp;（3）进入MOUNT模式

&emsp;&emsp;（4）使用RMAN，恢复datafile文件

&emsp;&emsp;（5）通过以下语句，打开resetlogs文件：

​       
```
ALTER DATABASE OPEN resetlogs;
```
 

&emsp;&emsp;实际工作中，会有专业的人员来操作RMAN，开发人员简单了解即可。




## 8.2  上机任务


#### 目标：完成本章8.1节的任务。

 


时间：60分钟。

 


形式：每个学员独立完成，小组组长检查。 

 


工具：CMD命令，SQL Plus命令

 


注意：需要以不同的用户身份访问数据库，完成相关权限的操作。

 

 

 

 

 



## 8.3  SQL优化

&emsp;&emsp;随着数据库中数据的增加，系统的响应速度就成为目前系统需要解决的最主要问题之一。系统优化中一个很重要的方面就是SQL语句的优化。对于大量数据，劣质SQL语句和优质SQL语句之间的速度差别可以达到上百倍。因此，对于一个系统，不是简单地能实现其功能就可以，而是要写出高质量的SQL语句，提高系统的可用性。

&emsp;&emsp;在多数情况下，Oracle使用索引来更快地遍历表，优化器主要根据定义的索引来提高性能。例如，假设在SQL语句的WHERE子句中写的SQL代码不合理，就会造成优化器忽略索引而使用全表扫描，类似这样的语句就是所谓的劣质SQL语句。因此，在编写SQL语句时应清楚优化器根据何种原则来使用索引，这有助于写出高性能的SQL语句。

### 8.3.1  不要让Oracle做得太多  

&emsp;&emsp;Oracle做得越多，性能损耗就越大。编写SQL时，应该尽力避免这种损耗，常见方法有以下五种。

- 避免复杂的多表关联

&emsp;&emsp;先来看以下代码：

 
```
SELECT …

FROM user_files uf, df_money_files dm, cw_charge_record cc

WHERE uf.user_no = dm.user_no

AND dm.user_no = cc.user_no

AND …

AND NOT EXISTS(SELECT …)
```
 

&emsp;&emsp;像这样烦琐的SQL语句是很难优化的，因此在编写SQL时，要避免复杂的多表关联。

- 避免使用“*”

&emsp;&emsp;当使用SELECT语句查询所有列时，使用SELECT *是一个方便的方法，但实际上这是一个非常低效的方法。因为Oracle在解析的过程中，会将“*”依次转换成所有的列名，这个工作是通过查询数据字典完成的，这意味着将耗费更多的时间。因此建议查询时只提取要使用的列，并且使用别名，使用别名能够加快解析速度。

- 避免使用耗费资源的操作。

&emsp;&emsp;带有DISTINCT、UNION、MINUS、INTERSECT、ORDER BY的SQL语句，会启动SQL引擎去执行耗费资源的排序功能。其中， DISTINCT需要一次排序操作, 而其他的至少需要执行两次排序。

&emsp;&emsp;例如，有一个UNION查询，并且连接的每个查询都带有GROUP BY子句，而GROUP BY会触发嵌入排序（NESTED SORT）。这样，每个查询都需要执行一次排序，然后执行UNION时又会再执行唯一排序（SORT UNIQUE），而且唯一排序只能在前面的嵌入排序结束后才能开始执行，因此嵌入排序的深度会大大影响查询的效率。

&emsp;&emsp;一般来讲，带有UNION、MINUS、INTERSECT的SQL语句都应该改写为其他形式。

- 用EXISTS替换DISTINCT

&emsp;&emsp;当提交一个包含一对多表信息的查询时（比如部门表和雇员表），避免在SELECT子句中使用DISTINCT。一般可以考虑用EXISTS替换，EXISTS会使查询更为迅速，因为RDBMS核心模块将在子查询的条件满足后，立刻返回结果。

&emsp;&emsp;低效：

 
```
SELECT DISTINCT DEPT_NO, DEPT_NAME

FROM DEPT D, EMP E

WHERE D.DEPT_NO = E.DEPT_NO
```
 

&emsp;&emsp;高效：

 
```
SELECT DEPT_NO, DEPT_NAME

FROM DEPT D

WHERE EXISTS (SELECT ‘X’

​                FROM EMP E

​                WHERE E.DEPT_NO = D.DEPT_NO);
```
 

- 用UNION ALL替换UNION

&emsp;&emsp;当SQL 语句使用UNION拼接两个查询结果集合时，这两个结果集合会先以UNION-ALL的方式被合并，然后在输出最终结果前进行排序。如果用UNION ALL替代UNION，这样排序就不是必要的了，效率就会因此得到提高。

&emsp;&emsp;低效：

​       
```
SELECT ACCT_NUM, BALANCE_AMT

FROM DEBIT_TRANSACTIONS

WHERE TRAN_DATE = '21-DEC-85'

UNION

SELECT ACCT_NUM, BALANCE_AMT

FROM DEBIT_TRANSACTIONS

WHERE TRAN_DATE = '21-DEC-85'
```
 

&emsp;&emsp;高效：

 
```
SELECT ACCT_NUM, BALANCE_AMT

FROM DEBIT_TRANSACTIONS

WHERE TRAN_DATE = '21-DEC-85'

UNION ALL

SELECT ACCT_NUM, BALANCE_AMT

FROM DEBIT_TRANSACTIONS

WHERE TRAN_DATE = '21-DEC-85'
```

### 8.3.2  给优化器更明确的命令  

&emsp;&emsp;发送给优化器的命令越明确，执行效率就越高。优化器是如何选择并执行命令的呢？可参考以下六点。




- 自动选择索引

&emsp;&emsp;如果表中有两个或两个以上索引，并且其中有一个唯一性索引，其他是非唯一性的，此时，Oracle将使用唯一性索引而完全忽略非唯一性索引。

&emsp;&emsp;例如：

 
```
SELECT ENAME

FROM EMP

WHERE EMPNO = 2326  

AND DEPTNO = 20 ;
```
 

&emsp;&emsp;假定只有EMPNO上的索引是唯一性的，那么该唯一性索引将用来检索记录。

 避免在索引列上使用函数或计算列

&emsp;&emsp;在WHERE子句中，如果索引列是函数的一部分或存在列的计算，优化器将不使用索引而使用全表扫描。

&emsp;&emsp;低效：

​       
```
SELECT …

FROM DEPT

WHERE SAL* 12 > 25000;
```
 

&emsp;&emsp;高效：

​       
```
SELECT …

FROM DEPT

WHERE SAL> 25000/12;
```
 

- 避免在索引列上使用NOT

&emsp;&emsp;与第（2）点相同，在索引列中使用NOT也会使得优化器进行全表扫描，即当Oracle遇到NOT时，就会停止使用索引转而执行全表扫描。

&emsp;&emsp;低效：

​       
```
SELECT …

FROM DEPT

WHERE NOT DEPT_CODE = 0;
```
 

&emsp;&emsp;高效：

 
```
SELECT …

FROM DEPT

WHERE DEPT_CODE > 0;
```
 

- 避免使用前置通配符

&emsp;&emsp;在WHERE子句中，如果索引列所对应的值的第一个字符以通配符开始，索引将被忽略。

&emsp;&emsp;例如：

​       
```
SELECT USER_NO, USER_NAME, ADDRESS

FROM USER_FILES

WHERE USER_NO LIKE  '%109204421';
```
 

&emsp;&emsp;在这种情况下，Oracle将进行全表扫描。

- 避免使用IS NULL

&emsp;&emsp;如果在WHERE子句中使用IS NULL或IS NOT NULL，那么优化器也是会忽略索引的。

- 避免出现索引列自动转换

&emsp;&emsp;当比较不同数据类型的数据时，Oracle会自动对列进行简单的类型转换。

&emsp;&emsp;如下，假设USER_NO是一个字符类型的索引列。

 
```
SELECT USER_NO,USER_NAME,ADDRESS

FROM USER_FILES

WHERE USER_NO = 109204421
```
 

&emsp;&emsp;这个语句会被Oracle转换为下面的语句：

​       
```
SELECT USER_NO,USER_NAME,ADDRESS

FROM USER_FILES

WHERE TO_NUMBER(USER_NO) = 109204421
```
 

&emsp;&emsp;内部发生了类型转换，因此这个索引将不会被用到。为了避免Oracle对所编写的SQL语句进行隐式的类型转换，最好把类型转换显式表现出来。并且注意，当字符和数值进行比较时，Oracle会优先将数值类型转换为字符类型。

### 8.3.3  减少访问次数  

&emsp;&emsp;当执行每条SQL语句时，Oracle都在内部执行了许多工作，如解析SQL语句、估算索引的利用率、绑定变量、读数据块等。由此可见，减少访问数据库的次数，就能减少Oracle的实际工作量。例如，可以使用DECODE函数避免重复扫描相同记录或重复连接相同的表，如下面的SQL语句：

​       
```
SELECT  COUNT(*)，SUM(SAL)

FROM EMP

WHERE DEPT_NO = 0020

AND ENAME LIKE ‘SMITH%’;

   SELECT COUNT(*)，SUM(SAL)

   FROM EMP

   WHERE DEPT_NO = 0030

​               AND ENAME LIKE　‘SMITH%’;
```
 

&emsp;&emsp;可以用DECODE函数高效地得到相同结果：

​       
```
SELECT COUNT(DECODE(DEPT_NO,0020,’X’,NULL)) D0020_COUNT,

​        COUNT(DECODE(DEPT_NO,0030,’X’,NULL)) D0030_COUNT,

​        SUM(DECODE(DEPT_NO,0020,SAL,NULL)) D0020_SAL,

​        SUM(DECODE(DEPT_NO,0030,SAL,NULL)) D0030_SAL

FROM EMP 

WHERE ENAME LIKE ‘SMITH%’;
```
 




### 8.3.4  细节上的影响  

&emsp;&emsp;WHERE子句中的连接顺序、ORDER BY中是否存在非索引项、用WHERE替换HAVING语句、用>=替代>等细节，都可能影响Oracle的性能。

- WHERE子句中的连接顺序

&emsp;&emsp;Oracle采用自下而上的顺序解析WHERE子句。根据这个原理, 当在WHERE子句中有多个表链接时，WHERE子句中排在最后的表应当是返回行数可能最少的表。

&emsp;&emsp;例如，假设从emp表查到的数据比较少或该表的过滤条件比较确定，能大大缩小查询范围，就需要将最精确的部分放在WHERE子句的最后：

​       
```
SELECT * FROM emp e, dept d

WHERE d.deptno>10 AND e.deptno=30 ;
```
 

&emsp;&emsp;如果dept表返回的记录数较多，那么上面的查询语句会比下面的查询语句快得多：

​       
```
SELECT * FROM emp e,dept d

WHERE e.deptno=30 AND d.deptno>10 ;
```
 

- ORDER BY子句

&emsp;&emsp;ORDER BY语句对要排序的列没有什么特别的限制，也可以将函数加入列中。但是如果在ORDER BY语句中存在非索引项或列的表达式计算，都将降低查询速度。

&emsp;&emsp;SQL优化的细节较多，除了本节介绍的这些知识以外，还需要在平日的学习工作中多总结、多积累。




## 8.4  上机任务


#### 目标：完成本章8.3节的任务。

 


时间：60分钟。

 


形式：每个学员独立完成，小组组长检查。 

 


工具：PL/SQL Dev

 

 

 

 

 

 



## 8.5  本章练习

 

1  备份有哪几种不同的方式？

 

 

2  进行冷备份的具体步骤是什么？冷备份需要注意什么？

 

 

3  使用EXP工具进行导出操作时，有哪几种不同的方式？

 

 
4  为什么要进行SQL优化？

 

 

5  UNION ALL与UNION在执行过程中，有什么区别？

 

 

6  出了本章介绍的以外，你还知道哪些SQL优化的知识？

 

 