---
title: sql-optimization
date: 2018-04-19 16:37:27
tags: 
      - sql
      - database
---

**一、问题的提出**

在应用系统开发初期，由于开发数据库数据比较少，对于查询SQL语句，复杂视图的的编写等体会不出SQL语句各种写法的性能优劣，但是如果将应用系统提交实际应用后，随着数据库中数据的增加，系统的响应速度就成为目前系统需要解决的最主要的问题之一。系统优化中一个很重要的方面就是SQL语句的优化。对于海量数据，劣质SQL语句和优质SQL语句之间的速度差别可以达到上百倍，可见对于一个系统不是简单地能实现其功能就可，而是要写出高质量的SQL语句，提高系统的可用性。
<!-- more -->

**在多数情况下，Oracle使用索引来更快地遍历表，优化器主要根据定义的索引来提高性能**。但是，如果在SQL语句的where子句中写的SQL代码不合理，就会**造成优化器删去索引而使用全表扫描**，一般就这种SQL语句就是所谓的劣质SQL语句。在编写SQL语句时我们应清楚优化器根据何种原则来删除索引，这有助于写出高性能的SQL语句。

**二、SQL语句编写注意问题**

下面就某些SQL语句的where子句编写中需要注意的问题作详细介绍。在这些where子句中，即使某些列存在索引，但是由于编写了劣质的SQL，系统在运行该SQL语句时也不能使用该索引，而同样使用全表扫描，这就造成了响应速度的极大降低。

**1. 操作符优化**

**(a) IN 操作符**

用IN写出来的SQL的优点是比较容易写及清晰易懂，这比较适合现代软件开发的风格。但是用IN的SQL性能总是比较低的，从Oracle执行的步骤来分析用IN的SQL与不用IN的SQL有以下区别：

ORACLE试图将其转换成多个表的连接，如果转换不成功则先执行IN里面的子查询，再查询外层的表记录，如果转换成功则直接采用多个表的连接方式查询。由此可见用IN的SQL至少多了一个转换的过程。一般的SQL都可以转换成功，但对于含有分组统计等方面的SQL就不能转换了。

**推荐方案：**在业务密集的SQL当中尽量不采用IN操作符，用EXISTS 方案代替。

**(b) NOT IN操作符**

此操作是强列不推荐使用的，因为它不能应用表的索引。

**推荐方案：**用NOT EXISTS 方案代替

**(c) IS NULL 或IS NOT NULL操作**（判断字段是否为空）

判断字段是否为空一般是不会应用索引的，因为索引是不索引空值的。不能用null作索引，任何包含null值的列都将不会被包含在索引中。即使索引有多列这样的情况下，只要这些列中有一列含有null，该列就会从索引中排除。也就是说如果某列存在空值，即使对该列建索引也不会提高性能。任何在where子句中使用is null或is not null的语句优化器是不允许使用索引的。

**推荐方案**：用其它相同功能的操作运算代替，如：a is not null 改为 a>0 或a>’’等。不允许字段为空，而用一个缺省值代替空值，如申请中状态字段不允许为空，缺省为申请。

**(d) > 及 < 操作符（大于或小于操作符）**

大于或小于操作符一般情况下是不用调整的，因为它有索引就会采用索引查找，但有的情况下可以对它进行优化，如一个表有100万记录，一个数值型字段A，30万记录的A=0，30万记录的A=1，39万记录的A=2，1万记录的A=3。那么执行A>2与A>=3的效果就有很大的区别了，因为A>2时ORACLE会先找出为2的记录索引再进行比较，而A>=3时ORACLE则直接找到=3的记录索引。

**(e) LIKE操作符**

LIKE操作符可以应用通配符查询，里面的通配符组合可能达到几乎是任意的查询，但是如果用得不好则会产生性能上的问题，如LIKE ‘%5400%’ 这种查询不会引用索引，而LIKE ‘X5400%’则会引用范围索引。

一个实际例子：用YW_YHJBQK表中营业编号后面的户标识号可来查询营业编号 YY_BH LIKE ‘%5400%’ 这个条件会产生全表扫描，如果改成YY_BH LIKE ’X5400%’ OR YY_BH LIKE ’B5400%’ 则会利用YY_BH的索引进行两个范围的查询，性能肯定大大提高。

带通配符(%)的like语句：

同样以上面的例子来看这种情况。目前的需求是这样的，要求在职工表中查询名字中包含cliton的人。可以采用如下的查询SQL语句:

select * from employee where last_name like '%cliton%';

这里由于通配符(%)在搜寻词首出现，所以Oracle系统不使用last_name的索引。在很多情况下可能无法避免这种情况，但是一定要心中有底，通配符如此使用会降低查询速度。然而当通配符出现在字符串其他位置时，优化器就能利用索引。在下面的查询中索引得到了使用:

select * from employee where last_name like 'c%';

**(f) UNION操作符**

UNION在进行表链接后会筛选掉重复的记录，所以在表链接后会对所产生的结果集进行排序运算，删除重复的记录再返回结果。实际大部分应用中是不会产生重复的记录，最常见的是过程表与历史表UNION。如： 
select * from gc_dfys 
union 
select * from ls_jg_dfys 
这个SQL在运行时先取出两个表的结果，再用排序空间进行排序删除重复的记录，最后返回结果集，如果表数据量大的话可能会导致用磁盘进行排序。

**推荐方案：**采用UNION ALL操作符替代UNION，因为UNION ALL操作只是简单的将两个结果合并后就返回。

select * from gc_dfys 
union all 
select * from ls_jg_dfys

**(g) 联接列**

对于有联接的列，即使最后的联接值为一个静态值，优化器是不会使用索引的。我们一起来看一个例子，假定有一个职工表(employee)，对于一个职工的姓和名分成两列存放(FIRST_NAME和LAST_NAME)，现在要查询一个叫比尔.克林顿(Bill Cliton)的职工。

下面是一个采用联接查询的SQL语句：

select * from employss where first_name||''||last_name ='Beill Cliton';

上面这条语句完全可以查询出是否有Bill Cliton这个员工，但是这里需要注意，系统优化器对基于last_name创建的索引没有使用。当采用下面这种SQL语句的编写，Oracle系统就可以采用基于last_name创建的索引。

*** where first_name ='Beill' and last_name ='Cliton';

**(h) Order by语句**

ORDER BY语句决定了Oracle如何将返回的查询结果排序。Order by语句对要排序的列没有什么特别的限制，也可以将函数加入列中(象联接或者附加等)。任何在Order by语句的非索引项或者有计算表达式都将降低查询速度。

仔细检查order by语句以找出非索引项或者表达式，它们会降低性能。解决这个问题的办法就是重写order by语句以使用索引，也可以为所使用的列建立另外一个索引，同时应绝对避免在order by子句中使用表达式。

**(i) NOT**

我们在查询时经常在where子句使用一些逻辑表达式，如大于、小于、等于以及不等于等等，也可以使用and(与)、or(或)以及not(非)。NOT可用来对任何逻辑运算符号取反。下面是一个NOT子句的例子:

... where not (status ='VALID')

如果要使用NOT，则应在取反的短语前面加上括号，并在短语前面加上NOT运算符。NOT运算符包含在另外一个逻辑运算符中，这就是不等于(<>)运算符。换句话说，即使不在查询where子句中显式地加入NOT词，NOT仍在运算符中，见下例:

... where status <>'INVALID';

对这个查询，可以改写为不使用NOT：

select * from employee where salary<3000 or salary>3000;

虽然这两种查询的结果一样，但是第二种查询方案会比第一种查询方案更快些。第二种查询允许Oracle对salary列使用索引，而第一种查询则不能使用索引。

**2. SQL书写的影响**

**(a) 同一功能同一性能不同写法SQL的影响。**

如一个SQL在A程序员写的为  Select * from zl_yhjbqk

B程序员写的为 Select * from dlyx.zl_yhjbqk（带表所有者的前缀）

C程序员写的为 Select * from DLYX.ZLYHJBQK（大写表名）

D程序员写的为 Select *  from DLYX.ZLYHJBQK（中间多了空格）

以上四个SQL在ORACLE分析整理之后产生的结果及执行的时间是一样的，但是从ORACLE共享内存SGA的原理，可以得出ORACLE对每个SQL 都会对其进行一次分析，并且占用共享内存，如果将SQL的字符串及格式写得完全相同，则ORACLE只会分析一次，共享内存也只会留下一次的分析结果，这不仅可以减少分析SQL的时间，而且可以减少共享内存重复的信息，ORACLE也可以准确统计SQL的执行频率。

**(b) WHERE后面的条件顺序影响**

WHERE子句后面的条件顺序对大数据量表的查询会产生直接的影响。如： 
Select * from zl_yhjbqk where dy_dj = '1KV以下' and xh_bz=1 
Select * from zl_yhjbqk where xh_bz=1 and dy_dj = '1KV以下' 
以上两个SQL中dy_dj（电压等级）及xh_bz（销户标志）两个字段都没进行索引，所以执行的时候都是全表扫描，第一条SQL的dy_dj = '1KV以下'条件在记录集内比率为99%，而xh_bz=1的比率只为0.5%，在进行第一条SQL的时候99%条记录都进行dy_dj及xh_bz的比较，而在进行第二条SQL的时候0.5%条记录都进行dy_dj及xh_bz的比较，以此可以得出第二条SQL的CPU占用率明显比第一条低。

**(c) 查询表顺序的影响**

在FROM后面的表中的列表顺序会对SQL执行性能影响，在没有索引及ORACLE没有对表进行统计分析的情况下，ORACLE会按表出现的顺序进行链接，由此可见表的顺序不对时会产生十分耗服物器资源的数据交叉。（注：如果对表进行了统计分析，ORACLE会自动先进小表的链接，再进行大表的链接）

**3. SQL语句索引的利用**

**(a) 对条件字段的一些优化**

**采用函数处理的字段不能利用索引，**如：

substr(hbs_bh,1,4)=’5400’，优化处理：hbs_bh like ‘5400%’

trunc(sk_rq)=trunc(sysdate)， 优化处理：sk_rq>=trunc(sysdate) and sk_rq<trunc(sysdate+1)

进行了显式或隐式的运算的字段不能进行索引，如：ss_df+20>50，优化处理：ss_df>30

‘X’ || hbs_bh>’X5400021452’，优化处理：hbs_bh>’5400021542’

sk_rq+5=sysdate，优化处理：sk_rq=sysdate-5

hbs_bh=5401002554，优化处理：hbs_bh=’ 5401002554’，注：此条件对hbs_bh 进行隐式的to_number转换，因为hbs_bh字段是字符型。

**条件内包括了多个本表的字段运算时不能进行索引**，如：

ys_df>cx_df，无法进行优化 
qc_bh || kh_bh=’5400250000’，优化处理：qc_bh=’5400’ and kh_bh=’250000’

**4. 更多方面SQL优化资料分享**

**（1） 选择最有效率的表名顺序(只在基于规则的优化器中有效)：**

ORACLE 的解析器按照从右到左的顺序处理FROM子句中的表名，FROM子句中写在最后的表(基础表 driving table)将被最先处理，在FROM子句中包含多个表的情况下,你必须选择记录条数最少的表作为基础表。如果有3个以上的表连接查询, 那就需要选择交叉表(intersection table)作为基础表, 交叉表是指那个被其他表所引用的表.

**（2） WHERE子句中的连接顺序：**

ORACLE采用自下而上的顺序解析WHERE子句,根据这个原理,表之间的连接必须写在其他WHERE条件之前, 那些可以过滤掉最大数量记录的条件必须写在WHERE子句的末尾.

**（3） SELECT子句中避免使用 ‘ \* ‘：**

ORACLE在解析的过程中, 会将'*' 依次转换成所有的列名, 这个工作是通过查询数据字典完成的, 这意味着将耗费更多的时间。

**（4） 减少访问数据库的次数：**

ORACLE在内部执行了许多工作: 解析SQL语句, 估算索引的利用率, 绑定变量 , 读数据块等。

（5） 在SQL*Plus , SQL*Forms和Pro*C中重新设置ARRAYSIZE参数, 可以增加每次数据库访问的检索数据量 ,建议值为200。

**（6） 使用DECODE函数来减少处理时间：**

使用DECODE函数可以避免重复扫描相同记录或重复连接相同的表.

**（7） 整合简单,无关联的数据库访问：**

如果你有几个简单的数据库查询语句,你可以把它们整合到一个查询中(即使它们之间没有关系) 。

**（8） 删除重复记录：**

最高效的删除重复记录方法 ( 因为使用了ROWID)例子： 
DELETE  FROM  EMP E  WHERE  E.ROWID > (SELECT MIN(X.ROWID) FROM  EMP X  WHERE  X.EMP_NO = E.EMP_NO)。

**（9） 用TRUNCATE替代DELETE：**

当删除表中的记录时,在通常情况下, 回滚段(rollback segments ) 用来存放可以被恢复的信息. 如果你没有COMMIT事务,ORACLE会将数据恢复到删除之前的状态(准确地说是恢复到执行删除命令之前的状况) 而当运用TRUNCATE时, 回滚段不再存放任何可被恢复的信息.当命令运行后,数据不能被恢复.因此很少的资源被调用,执行时间也会很短. (译者按: TRUNCATE只在删除全表适用,TRUNCATE是DDL不是DML) 。

**（10） 尽量多使用COMMIT：**

只要有可能,在程序中尽量多使用COMMIT, 这样程序的性能得到提高,需求也会因为COMMIT所释放的资源而减少，COMMIT所释放的资源: 
a. 回滚段上用于恢复数据的信息. 
b. 被程序语句获得的锁 
c. redo log buffer 中的空间 
d. ORACLE为管理上述3种资源中的内部花费

**（11） 用Where子句替换HAVING子句：**

避免使用HAVING子句, HAVING 只会在检索出所有记录之后才对结果集进行过滤. 这个处理需要排序,总计等操作. 如果能通过WHERE子句限制记录的数目,那就能减少这方面的开销. (非oracle中)on、where、having这三个都可以加条件的子句中，on是最先执行，where次之，having最后，因为on是先把不符合条件的记录过滤后才进行统计，它就可以减少中间运算要处理的数据，按理说应该速度是最快的，where也应该比having快点的，因为它过滤数据后才进行sum，在两个表联接时才用on的，所以在一个表的时候，就剩下where跟having比较了。在这单表查询统计的情况下，如果要过滤的条件没有涉及到要计算字段，那它们的结果是一样的，只是where可以使用rushmore技术，而having就不能，在速度上后者要慢如果要涉及到计算的字 段，就表示在没计算之前，这个字段的值是不确定的，根据上篇写的工作流程，where的作用时间是在计算之前就完成的，而having就是在计算后才起作 用的，所以在这种情况下，两者的结果会不同。在多表联接查询时，on比where更早起作用。系统首先根据各个表之间的联接条件，把多个表合成一个临时表 后，再由where进行过滤，然后再计算，计算完后再由having进行过滤。由此可见，要想过滤条件起到正确的作用，首先要明白这个条件应该在什么时候起作用，然后再决定放在那里。

**（12） 减少对表的查询：**

在含有子查询的SQL语句中,要特别注意减少对表的查询.例子： 
SELECT  TAB_NAME FROM TABLES WHERE (TAB_NAME,DB_VER) = ( SELECT TAB_NAME,DB_VER FROM  TAB_COLUMNS  WHERE  VERSION = 604)

**（13） 通过内部函数提高SQL效率：**

复杂的SQL往往牺牲了执行效率. 能够掌握上面的运用函数解决问题的方法在实际工作中是非常有意义的。

**（14） 使用表的别名(Alias)：**

当在SQL语句中连接多个表时, 请使用表的别名并把别名前缀于每个Column上.这样一来,就可以减少解析的时间并减少那些由Column歧义引起的语法错误。

**（15） 用EXISTS替代IN、用NOT EXISTS替代NOT IN：**

在许多基于基础表的查询中,为了满足一个条件,往往需要对另一个表进行联接.在这种情况下, 使用EXISTS(或NOT EXISTS)通常将提高查询的效率. 在子查询中,NOT IN子句将执行一个内部的排序和合并. 无论在哪种情况下,NOT IN都是最低效的 (因为它对子查询中的表执行了一个全表遍历). 为了避免使用NOT IN ,我们可以把它改写成外连接(Outer Joins)或NOT EXISTS。 
例子： 
（高效）SELECT * FROM  EMP (基础表)  WHERE  EMPNO > 0  AND  EXISTS (SELECT ‘X'  FROM DEPT  WHERE  DEPT.DEPTNO = EMP.DEPTNO  AND  LOC = ‘MELB') 
(低效)SELECT  * FROM  EMP (基础表)  WHERE  EMPNO > 0  AND  DEPTNO IN(SELECT DEPTNO  FROM  DEPT  WHERE  LOC = ‘MELB')

**（16） 识别'低效执行'的SQL语句：**

虽然目前各种关于SQL优化的图形化工具层出不穷,但是写出自己的SQL工具来解决问题始终是一个最好的方法： 
SELECT  EXECUTIONS , DISK_READS, BUFFER_GETS, 
ROUND((BUFFER_GETS-DISK_READS)/BUFFER_GETS,2) Hit_radio, 
ROUND(DISK_READS/EXECUTIONS,2) Reads_per_run, 
SQL_TEXT 
FROM  V$SQLAREA 
WHERE  EXECUTIONS>0 
AND  BUFFER_GETS > 0 
AND  (BUFFER_GETS-DISK_READS)/BUFFER_GETS < 0.8 
ORDER BY  4 DESC;

**（17） 用索引提高效率：**

索引是表的一个概念部分,用来提高检索数据的效率，ORACLE使用了一个复杂的自平衡B-tree结构. 通常,通过索引查询数据比全表扫描要快. 当ORACLE找出执行查询和Update语句的最佳路径时, ORACLE优化器将使用索引. 同样在联结多个表时使用索引也可以提高效率. 另一个使用索引的好处是,它提供了主键(primary key)的唯一性验证.。那些LONG或LONG RAW数据类型, 你可以索引几乎所有的列. 通常, 在大型表中使用索引特别有效. 当然,你也会发现, 在扫描小表时,使用索引同样能提高效率. 虽然使用索引能得到查询效率的提高,但是我们也必须注意到它的代价. 索引需要空间来存储,也需要定期维护, 每当有记录在表中增减或索引列被修改时, 索引本身也会被修改. 这意味着每条记录的INSERT , DELETE , UPDATE将为此多付出4 , 5 次的磁盘I/O . 因为索引需要额外的存储空间和处理,那些不必要的索引反而会使查询反应时间变慢.。定期的重构索引是有必要的： 
ALTER  INDEX <INDEXNAME> REBUILD <TABLESPACENAME>

**（18） 用EXISTS替换DISTINCT：**

当提交一个包含一对多表信息(比如部门表和雇员表)的查询时,避免在SELECT子句中使用DISTINCT. 一般可以考虑用EXIST替换, EXISTS 使查询更为迅速,因为RDBMS核心模块将在子查询的条件一旦满足后,立刻返回结果. 例子： 
(低效): 
SELECT  DISTINCT  DEPT_NO,DEPT_NAME  FROM  DEPT D , EMP E WHERE  D.DEPT_NO = E.DEPT_NO 
(高效): 
SELECT  DEPT_NO,DEPT_NAME  FROM  DEPT D  WHERE  EXISTS ( SELECT ‘X'  FROM  EMP E  WHERE E.DEPT_NO = D.DEPT_NO);

（19） sql语句用大写的；因为oracle总是先解析sql语句，把小写的字母转换成大写的再执行。

（20） 在java代码中尽量少用连接符“＋”连接字符串！

（21） 避免在索引列上使用NOT，通常我们要避免在索引列上使用NOT, NOT会产生在和在索引列上使用函数相同的影响. 当ORACLE”遇到”NOT,他就会停止使用索引转而执行全表扫描。

**（22） 避免在索引列上使用计算** WHERE子句中，如果索引列是函数的一部分．优化器将不使用索引而使用全表扫描．举例: 
低效： 
SELECT … FROM  DEPT  WHERE SAL * 12 > 25000; 
高效: 
SELECT … FROM DEPT WHERE SAL > 25000/12;

**（23） 用>=替代>** 
高效: 
SELECT * FROM  EMP  WHERE  DEPTNO >=4 
低效: 
SELECT * FROM EMP WHERE DEPTNO >3 
两者的区别在于, 前者DBMS将直接跳到第一个DEPT等于4的记录而后者将首先定位到DEPTNO=3的记录并且向前扫描到第一个DEPT大于3的记录。

**（24） 用UNION替换OR (适用于索引列)**

通常情况下, 用UNION替换WHERE子句中的OR将会起到较好的效果. 对索引列使用OR将造成全表扫描. 注意, 以上规则只针对多个索引列有效. 如果有column没有被索引, 查询效率可能会因为你没有选择OR而降低. 在下面的例子中, LOC_ID 和REGION上都建有索引. 
高效: 
SELECT LOC_ID , LOC_DESC , REGION 
FROM LOCATION 
WHERE LOC_ID = 10 
UNION 
SELECT LOC_ID , LOC_DESC , REGION 
FROM LOCATION 
WHERE REGION = “MELBOURNE” 
低效: 
SELECT LOC_ID , LOC_DESC , REGION 
FROM LOCATION 
WHERE LOC_ID = 10 OR REGION = “MELBOURNE” 
如果你坚持要用OR, 那就需要返回记录最少的索引列写在最前面.

**（25） 用IN来替换OR**

这是一条简单易记的规则，但是实际的执行效果还须检验，在ORACLE8i下，两者的执行路径似乎是相同的． 
低效: 
SELECT…. FROM LOCATION WHERE LOC_ID = 10 OR LOC_ID = 20 OR LOC_ID = 30 
高效 
SELECT… FROM LOCATION WHERE LOC_IN  IN (10,20,30);

**（26） 避免在索引列上使用IS NULL和IS NOT NULL**

避免在索引中使用任何可以为空的列，ORACLE将无法使用该索引．对于单列索引，如果列包含空值，索引中将不存在此记录. 对于复合索引，如果每个列都为空，索引中同样不存在此记录. 如果至少有一个列不为空，则记录存在于索引中．举例: 如果唯一性索引建立在表的A列和B列上, 并且表中存在一条记录的A,B值为(123,null) , ORACLE将不接受下一条具有相同A,B值（123,null）的记录(插入). 然而如果所有的索引列都为空，ORACLE将认为整个键值为空而空不等于空. 因此你可以插入1000 条具有相同键值的记录,当然它们都是空! 因为空值不存在于索引列中,所以WHERE子句中对索引列进行空值比较将使ORACLE停用该索引. 
低效: (索引失效) 
SELECT … FROM  DEPARTMENT  WHERE  DEPT_CODE IS NOT NULL; 
高效: (索引有效) 
SELECT … FROM  DEPARTMENT  WHERE  DEPT_CODE >=0;

**（27） 总是使用索引的第一个列：**

如果索引是建立在多个列上, 只有在它的第一个列(leading column)被where子句引用时,优化器才会选择使用该索引. 这也是一条简单而重要的规则，当仅引用索引的第二个列时,优化器使用了全表扫描而忽略了索引。

**（28） 用UNION-ALL 替换UNION ( 如果有可能的话)：**

当SQL 语句需要UNION两个查询结果集合时,这两个结果集合会以UNION-ALL的方式被合并, 然后在输出最终结果前进行排序. 如果用UNION ALL替代UNION, 这样排序就不是必要了. 效率就会因此得到提高. 需要注意的是，UNION ALL 将重复输出两个结果集合中相同记录. 因此各位还是要从业务需求分析使用UNION ALL的可行性. UNION 将对结果集合排序,这个操作会使用到SORT_AREA_SIZE这块内存. 对于这块内存的优化也是相当重要的. 下面的SQL可以用来查询排序的消耗量 
低效： 
SELECT  ACCT_NUM, BALANCE_AMT 
FROM  DEBIT_TRANSACTIONS 
WHERE TRAN_DATE = '31-DEC-95' 
UNION 
SELECT ACCT_NUM, BALANCE_AMT 
FROM DEBIT_TRANSACTIONS 
WHERE TRAN_DATE = '31-DEC-95' 
高效: 
SELECT ACCT_NUM, BALANCE_AMT 
FROM DEBIT_TRANSACTIONS 
WHERE TRAN_DATE = '31-DEC-95' 
UNION ALL 
SELECT ACCT_NUM, BALANCE_AMT 
FROM DEBIT_TRANSACTIONS 
WHERE TRAN_DATE = '31-DEC-95'

**（29） 用WHERE替代ORDER BY：**

ORDER BY 子句只在两种严格的条件下使用索引. 
ORDER BY中所有的列必须包含在相同的索引中并保持在索引中的排列顺序. 
ORDER BY中所有的列必须定义为非空. 
WHERE子句使用的索引和ORDER BY子句中所使用的索引不能并列. 
例如: 
表DEPT包含以下列: 
DEPT_CODE PK NOT NULL 
DEPT_DESC NOT NULL 
DEPT_TYPE NULL 
低效: (索引不被使用) 
SELECT DEPT_CODE FROM  DEPT  ORDER BY  DEPT_TYPE 
高效: (使用索引) 
SELECT DEPT_CODE  FROM  DEPT  WHERE  DEPT_TYPE > 0

**（30） 避免改变索引列的类型:**

当比较不同数据类型的数据时, ORACLE自动对列进行简单的类型转换. 
假设 EMPNO是一个数值类型的索引列. 
SELECT …  FROM EMP  WHERE  EMPNO = ‘123' 
实际上,经过ORACLE类型转换, 语句转化为: 
SELECT …  FROM EMP  WHERE  EMPNO = TO_NUMBER(‘123') 
幸运的是,类型转换没有发生在索引列上,索引的用途没有被改变. 
现在,假设EMP_TYPE是一个字符类型的索引列. 
SELECT …  FROM EMP  WHERE EMP_TYPE = 123 
这个语句被ORACLE转换为: 
SELECT …  FROM EMP  WHERE TO_NUMBER(EMP_TYPE)=123 
因为内部发生的类型转换, 这个索引将不会被用到! 为了避免ORACLE对你的SQL进行隐式的类型转换, 最好把类型转换用显式表现出来. 注意当字符和数值比较时, ORACLE会优先转换数值类型到字符类型。

分析select   emp_name   form   employee   where   salary   >   3000   在此语句中若salary是Float类型的，则优化器对其进行优化为Convert(float,3000)，因为3000是个整数，我们应在编程时使用3000.0而不要等运行时让DBMS进行转化。同样字符和整型数据的转换。

**（31） 需要当心的WHERE子句:**

某些SELECT 语句中的WHERE子句不使用索引. 这里有一些例子. 
在下面的例子里, (1)‘!=' 将不使用索引. 记住, 索引只能告诉你什么存在于表中, 而不能告诉你什么不存在于表中. (2) ‘ ¦ ¦'是字符连接函数. 就象其他函数那样, 停用了索引. (3) ‘+'是数学函数. 就象其他数学函数那样, 停用了索引. (4)相同的索引列不能互相比较,这将会启用全表扫描.

（32） a. 如果检索数据量超过30%的表中记录数.使用索引将没有显著的效率提高. b. 在特定情况下, 使用索引也许会比全表扫描慢, 但这是同一个数量级上的区别. 而通常情况下,使用索引比全表扫描要块几倍乃至几千倍!

**（33） 避免使用耗费资源的操作：**

带有DISTINCT,UNION,MINUS,INTERSECT,ORDER BY的SQL语句会启动SQL引擎执行耗费资源的排序(SORT)功能. DISTINCT需要一次排序操作, 而其他的至少需要执行两次排序. 通常, 带有UNION, MINUS , INTERSECT的SQL语句都可以用其他方式重写. 如果你的数据库的SORT_AREA_SIZE调配得好, 使用UNION , MINUS, INTERSECT也是可以考虑的, 毕竟它们的可读性很强。

**（34） 优化GROUP BY：**

提高GROUP BY 语句的效率, 可以通过将不需要的记录在GROUP BY 之前过滤掉.下面两个查询返回相同结果但第二个明显就快了许多. 
低效: 
SELECT JOB , AVG(SAL) 
FROM EMP 
GROUP by JOB 
HAVING JOB = ‘PRESIDENT' 
OR JOB = ‘MANAGER' 
高效: 
SELECT JOB , AVG(SAL) 
FROM EMP 
WHERE JOB = ‘PRESIDENT' 
OR JOB = ‘MANAGER' 
GROUP by JOB