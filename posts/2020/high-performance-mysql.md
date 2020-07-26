---
layout: default
title: 读书笔记《高性能MySQL》
---

# 读书笔记《高性能MySQL》

### 第1章　MySQL架构与历史
#### 1.3.1　隔离级别
>REPEATABLE READ（可重复读）解决了脏读的问题。该级别保证了在同一个事务中多次读取同样记录的结果是一致的。但是理论上，可重复读隔离级别还是无法解决另外一个幻读（PhantomRead）的问题。所谓幻读，指的是当某个事务在读取某个范围内的记录时，另外一个事务又在该范围内插入了新的记录，当之前的事务再次读取该范围的记录时，会产生幻行（PhantomRow）。InnoDB和XtraDB存储引擎通过多版本并发控制（MVCC）解决了幻读的问题。  
SERIALIZABLE（可串行化）是最高的隔离级别。它通过强制事务串行执行，避免了前面说的幻读的问题。简单来说，SERIALIZABLE会在读取的每一行数据上都加锁，所以可能导致大量的超时和锁争用的问题。

~我的理解，可串行化这个隔离级别，会在读取的每一行数据上都加共享锁，而非排它锁。可重复读是通过间隙锁解决了幻读的问题。


#### 1.3.3　事务日志
>事务日志可以帮助提高事务的效率。使用事务日志，存储引擎在修改表的数据时只需要修改其内存拷贝，再把该修改行为记录到并持久在硬盘上的事务日志中，而不用每次都将修改的数据本身持久到磁盘。事务日志采用的是追加的方式，因此写日志的操作是磁盘上一小块区域内的顺序I/O，而不像随机I/O需要在磁盘的多个地方移动磁头，所以采用事务日志的方式相对来说要快得多。事务日志持久以后，内存中被修改的数据在后台可以慢慢地刷回到磁盘。目前大多数存储引擎都是这样实现的，我们通常称之为预写式日志（Write-Ahead Logging），修改数据需要写两次磁盘。  
如果数据的修改已经记录到事务日志并持久化，但数据本身还没有写回磁盘，此时系统崩溃，存储引擎在重启时能够自动恢复这部分修改的数据。

~让我更好地理解为什么要先写入事务日志，一切都是为了性能。

#### 1.4　多版本并发控制
>InnoDB的MVCC，是通过在每行记录后面保存两个隐藏的列来实现的。这两个列，一个保存了行的创建时间，一个保存行的过期时间（或删除时间）。当然存储的并不是实际的时间值，而是系统版本号（system version number）。每开始一个新的事务，系统版本号都会自动递增。事务开始时刻的系统版本号会作为事务的版本号，用来和查询到的每行记录的版本号进行比较。下面看一下在REPEATABLE READ隔离级别下，MVCC具体是如何操作的。  
SELECT，InnoDB会根据以下两个条件检查每行记录：  
InnoDB只查找版本早于当前事务版本的数据行（也就是，行的系统版本号小于或等于事务的系统版本号），这样可以确保事务读取的行，要么是在事务开始前已经存在的，要么是事务自身插入或者修改过的。  
行的删除版本要么未定义，要么大于当前事务版本号。这可以确保事务读取到的行，在事务开始之前未被删除。  
只有符合上述两个条件的记录，才能返回作为查询结果。  
INSERT，InnoDB为新插入的每一行保存当前系统版本号作为行版本号。  
DELETE，InnoDB为删除的每一行保存当前系统版本号作为行删除标识。  
UPDATE，InnoDB为插入一行新记录，保存当前系统版本号作为行版本号，同时保存当前系统版本号到原来的行作为行删除标识。  

~多版本控制中，SELECT,UPDATE的行为比较特别，SELECT还是比较好理解，返回的记录的行的版本号必须小于或等于当前事务版本号，另外删除版本号要么为空，要么大于当前事务的版本号。对于UPDATE，这个有点违反我的直觉，也就是先插入，后删除。这样删除事务之前开始的其他事务就能够查询修改之前的记录。


#### 1.5.5　选择合适的引擎
>如何选择存储引擎，可以简单地归纳为一句话：“除非需要用到某些InnoDB不具备的特性，并且没有其他办法可以替代，否则都应该优先选择InnoDB引擎”。例如，如果要用到全文索引，建议优先考虑InnoDB加上Sphinx的组合，而不是使用支持全文索引的MyISAM。当然，如果不需要用到InnoDB的特性，同时其他引擎的特性能够更好地满足需求，也可以考虑一下其他存储引擎。  
除非万不得已，否则建议不要混合使用多种存储引擎，否则可能带来一系列复杂的问题，以及一些潜在的bug和边界问题。存储引擎层和服务器层的交互已经比较复杂，更不用说混合多个存储引擎了。至少，混合存储对一致性备份和服务器参数配置都带来了一些困难。  

~两个问题，一是如何选择存储引擎？二是为什么不要混合使用多种存储引擎。


#### 日志型应用
>如果需要对记录的日志做分析报表，则事情就会变得有趣了。生成报表的SQL很有可能会导致插入效率明显降低，这时候该怎么办？  
一种解决方法，是利用MySQL内置的复制方案将数据复制一份到备库，然后在备库上执行比较消耗时间和CPU的查询。这样主库只用于高效的插入工作，而备库上执行的查询也无须担心影响到日志的插入性能。当然也可以在系统负载较低的时候执行报表查询操作，但应用在不断变化，如果依赖这个策略可能以后会导致问题。  
另外一种方法，在日志记录表的名字中包含年和月的信息，比如web_logs_2012_01或者web_logs_2012_jan。这样可以在已经没有插入操作的历史表上做频繁的查询操作，而不会干扰到最新的当前表上的插入操作。  

~其实这两种方案就是将数据插入操作与统计查询操作分开，第一个方案就是放在主从两个服务上进行，另外一种方案放在同一个库的不同表上进行。

#### 1.5.6　转换表的引擎
>第三种转换的技术综合了第一种方法的高效和第二种方法的安全。不需要导出整个表的数据，而是先创建一个新的存储引擎的表，然后利用INSERT…SELECT语法来导数据。  
数据量不大的话，这样做工作得很好。如果数据量很大，则可以考虑做分批处理，针对每一段数据执行事务提交操作，以避免大事务产生过多的undo。假设有主键字段id，重复运行以下语句（最小值x和最大值y进行相应的替换）将数据导入到新表。  

~可以采用的方法有多种，这样方式比较简单并实用，记录在这里，提供一个线索，以后遇到这样的场景能够想起来。

### 第2章　MySQL基准测试

#### 2.1　为什么需要基准测试
>为什么基准测试很重要？因为基准测试是唯一方便有效的、可以学习系统在给定的工作负载下会发生什么的方法。基准测试可以观察系统在不同压力下的行为，评估系统的容量，掌握哪些是重要的变化，或者观察系统如何处理不同的数据。基准测试可以在系统实际负载之外创造一些虚构场景进行测试。

~为什么基准测试很重要？


### 第3章　服务器性能剖析
#### 3.1　性能优化简介
>我们将性能定义为完成某件任务所需要的时间度量，换句话说，性能即响应时间，这是一个非常重要的原则。我们通过任务和时间而不是资源来测量性能。数据库服务器的目的是执行SQL语句，所以它关注的任务是查询或者语句，如SELECT、UPDATE、DELETE等。数据库服务器的性能用查询的响应时间来度量，单位是每个查询花费的时间。

~这个性能的定义，有别于我以前的认识。



>我们观察到，很多人在优化时，都将精力放在修改一些东西上，却很少去进行精确的测量。我们的做法完全相反，将花费非常多，甚至90％的时间来测量响应时间花在哪里。如果通过测量没有找到答案，那要么是测量的方式错了，要么是测量得不够完整。如果测量了系统中完整而且正确的数据，性能问题一般都能暴露出来，对症下药的解决方案也就比较明了。

~这就是一个提醒，我以前也是这样干的，一上来就优化，也不会测量问题在哪里，而只是猜测。


#### 3.1.x　性能优化简介
>完成一项任务所需要的时间可以分成两部分：执行时间和等待时间。如果要优化任务的执行时间，最好的办法是通过测量定位不同的子任务花费的时间，然后优化去掉一些子任务、降低子任务的执行频率或者提升子任务的效率。而优化任务的等待时间则相对要复杂一些，因为等待有可能是由其他系统间接影响导致，任务之间也可能由于争用磁盘或者CPU资源而相互影响。

~要知道这些，优化MYSQL的性能，至少要知道在应该在那些方面优化。


#### 3.2　对应用程序进行性能剖析
>大多数设计和构建过高性能应用程序的人相信，应该尽可能地测量一切可以测量的地方，并且接受这些测量带来的额外开销，这些开销应该被当成应用程序的一部分。Oracle的性能优化大师Tom Kyte曾被问到Oracle中的测量点的开销，他的回答是，测量点至少为性能优化贡献了10％。

~如何看待测量，也就是说不要担心测量带来的性能的影响，而应该去测量一切可以测量的地方。


#### 3.3.1　剖析服务器负载
>在MySQL的当前版本中，慢查询日志是开销最低、精度最高的测量查询时间的工具。如果还在担心开启慢查询日志会带来额外的I/O开销，那大可以放心。我们在I/O密集型场景做过基准测试，慢查询日志带来的开销可以忽略不计（实际上在CPU密集型场景的影响还稍微大一些）。更需要担心的是日志可能消耗大量的磁盘空间。如果长期开启慢查询日志，注意要部署日志轮转（logrotation）工具。或者不要长期启用慢查询日志，只在需要收集负载样本的期间开启即可。

~所以根本就不要担心慢查询的性能影响，只要注意日志文件所消耗的磁盘空间的问题。



#### 3.4　诊断间歇性问题
>下面列举了我们认为已经解决的一些间歇性数据库性能问题的实际案例：  
1、应用通过curl从一个运行得很慢的外部服务来获取汇率报价的数据。  
2、memcached缓存中的一些重要条目过期，导致大量请求落到MySQL以重新生成缓存条目。
3、DNS查询偶尔会有超时现象。  
4、可能是由于互斥锁争用，或者内部删除查询缓存的算法效率太低的缘故，MySQL的查询缓存有时候会导致服务有短暂的停顿。  
5、当并发度超过某个阈值时，InnoDB的扩展性限制导致查询计划的优化需要很长的时间。   

~这些案例，我可能也会遇到。


### 第4章　Schema与数据类型优化
#### 4.1　选择优化的数据类型
>一般情况下，应该尽量使用可以正确存储数据的最小数据类型。更小的数据类型通常更快，因为它们占用更少的磁盘、内存和CPU缓存，并且处理时需要的CPU周期也更少。  
很多表都包含可为NULL（空值）的列，即使应用程序并不需要保存NULL也是如此，这是因为可为NULL是列的默认属性。通常情况下最好指定列为NOT NULL，除非真的需要存储NULL值。  
如果查询中包含可为NULL的列，对MySQL来说更难优化，因为可为NULL的列使得索引、索引统计和值比较都更复杂。可为NULL的列会使用更多的存储空间，在MySQL里也需要特殊处理。  

~这是设计数据字典应该遵守的两个规则，能够一定程度提升性能或简化程序的逻辑。

#### 4.1.1　整数类型
>你的选择决定MySQL是怎么在内存和磁盘中保存数据的。然而，整数计算一般使用64位的BIGINT整数，即使在32位环境也是如此。（一些聚合函数是例外，它们使用DECIMAL或DOUBLE进行计算）。  
MySQL可以为整数类型指定宽度，例如INT（11），对大多数应用这是没有意义的：它不会限制值的合法范围，只是规定了MySQL的一些交互工具（例如MySQL命令行客户端）用来显示字符的个数。对于存储和计算来说，INT（1）和INT（20）是相同的。  

~类型TINYINT，SMALLINT，MEDIUMINT，INT，BIGINT分别使用8，16，24，32，64位存储空间，并不能通过指定宽度而改变存储空间的大小。


#### 4.1.2　实数类型
>浮点和DECIMAL类型都可以指定精度。对于DECIMAL列，可以指定小数点前后所允许的最大位数。这会影响列的空间消耗。MySQL5.0和更高版本将数字打包保存到一个二进制字符串中（每4个字节存9个数字）。例如，DECIMAL（18,9）小数点两边将各存储9个数字，一共使用9个字节：小数点前的数字用4个字节，小数点后的数字用4个字节，小数点本身占1个字节。  
因为需要额外的空间和计算开销，所以应该尽量只在对小数进行精确计算时才使用DECIMAL——例如存储财务数据。但在数据量比较大的时候，可以考虑使用BIGINT代替DECIMAL，将需要存储的货币单位根据小数的位数乘以相应的倍数即可。假设要存储财务数据精确到万分之一分，则可以把所有金额乘以一百万，然后将结果存储在BIGINT里，这样可以同时避免浮点存储计算不精确和DECIMAL精确计算代价高的问题。

~‘在数据量比较大的时候，可以考虑使用BITINT代替DECIMAL’，这里有一个前提，就是在数据量比较大的情况，因为BITINT一般使用64位，也就是8个字节。如果使用BITING表示比较小的数，那么使用DECIMAL表示此数字可能所占用的空间更小一些，所以需要注意。

#### 4.1.3　字符串类型
>这些情况下使用VARCHAR是合适的：字符串列的最大长度比平均长度大很多；列的更新很少，所以碎片不是问题；使用了像UTF-8这样复杂的字符集，每个字符都使用不同的字节数进行存储。

~知道在什么情况下使用VARCHAR很重要，因为使用VARCHAR类型的机会比较多。



>CHAR适合存储很短的字符串，或者所有值都接近同一个长度。例如，CHAR非常适合存储密码的MD5值，因为这是一个定长的值。对于经常变更的数据，CHAR也比VARCHAR更好，因为定长的CHAR类型不容易产生碎片。对于非常短的列，CHAR比VARCHAR在存储空间上也更有效率。例如用CHAR（1）来存储只有Y和N的值，如果采用单字节字符集(5)只需要一个字节，但是VARCHAR（1）却需要两个字节，因为还有一个记录长度的额外字节。

~其中一些观点，比如对于经常变更的数据，CHAR更好，因为不易产生碎片，这悖于我之前的认识，但是仔细想想也是合理。


>更长的列会消耗更多的内存，因为MySQL通常会分配固定大小的内存块来保存内部值。尤其是使用内存临时表进行排序或操作时会特别糟糕。在利用磁盘临时表进行排序时也同样糟糕。

~这个要知道，当MYSQL使用内存临时表进行排序或操作，相对于数据表来说会占用更大的内存空间，比如某列的类型为VARCHAR(100)，但是存储的值都是10个字符上下。那么在使用临时表排序或其他某些操作时，MYSQL将会分配100字符的大小的空间来存储这个10个左右的字符。


#### 使用枚举（ENUM）代替字符串类型
>枚举列按照枚举定义顺序排序，一种绕过这种限制的方式是按照需要的顺序来定义枚举列。另外也可以在查询中使用FIELD()函数显式地指定排序顺序，但这会导致MySQL无法利用索引消除排序。  
枚举最不好的地方是，字符串列表是固定的，添加或删除字符串必须使用ALTER TABLE。因此，对于一系列未来可能会改变的字符串，使用枚举不是一个好主意，除非能接受只在列表末尾添加元素，这样在MySQL5.1中就可以不用重建整个表来完成修改。

~了解枚举类型的优缺点，才能更好的使用此种类型。


#### TIMESTAMP
>TIMESTAMP也有DATETIME没有的特殊属性。默认情况下，如果插入时没有指定第一个TIMESTAMP列的值，MySQL则设置这个列的值为当前时间。在修改一行记录时，MySQL默认也会更新第一个TIMESTAMP列的值（除非在UPDATE语句中明确指定了值）。你可以配置任何TIMESTAMP列的插入和更新行为。最后，TIMESTAMP列默认为NOT NULL，这也和其他的数据类型不一样。  
除了特殊行为之外，通常也应该尽量使用TIMESTAMP，因为它比DATETIME空间效率更高。

~因为TIMESTAMP只需4个字节的存储空间，而DATETIME需要8字节的存储空间，这样空间利用效率更好。再加上TIMESTAMP还有一些其他的特征，所以一般采用TIMESTAMP类型。



>如果需要存储比秒更小粒度的日期和时间值怎么办？MySQL目前没有提供合适的数据类型，但是可以使用自己的存储格式：可以使用BIGINT类型存储微秒级别的时间截，或者使用DOUBLE存储秒之后的小数部分。这两种方式都可以，或者也可以使用MariaDB替代MySQL。

~记录在这里，以后遇到此类场景，可以参考使用。

#### 4.1.6　选择标识符（identifier）
>一旦选定了一种类型，要确保在所有关联表中都使用同样的类型。类型之间需要精确匹配，包括像UNSIGNED这样的属性。混用不同数据类型可能导致性能问题，即使没有性能影响，在比较操作时隐式类型转换也可能导致很难发现的错误。  
整数通常是标识列最好的选择，因为它们很快并且可以使用AUTO_INCREMENT。

~标识符最好选用整数，如果不选择整数，也要确保选择的类型尽量简单，且与外键的类型以及属性保持一致，防止可能的性能问题以及隐式的类型转换。


#### 4.1.7　特殊类型数据
>另一个例子是一个IPv4地址。人们经常使用VARCHAR（15）列来存储IP地址。然而，它们实际上是32位无符号整数，不是字符串。用小数点将地址分成四段的表示方法只是为了让人们阅读容易。所以应该用无符号整数存储IP地址。MySQL提供INET_ATON()和INET_NTOA()函数在这两种表示方法之间转换。  
```sql
>SELECT INET_ATON('10.0.81.142');
>SELECT INET_NTOA(167793038);
```

~如何高效存储IP地址的方法。


#### 4.2　MySQLschema设计中的陷阱
>MySQL的存储引擎API工作时需要在服务器层和存储引擎层之间通过行缓冲格式拷贝数据，然后在服务器层将缓冲内容解码成各个列。从行缓冲中将编码过的列转换成行数据结构的操作代价是非常高的。MyISAM的定长行结构实际上与服务器层的行结构正好匹配，所以不需要转换。然而，MyISAM的变长行结构和InnoDB的行结构则总是需要转换。转换的代价依赖于列的数量。

~所以说不要为了方便而直接使用*，将一直不需要的列都查询出来，这样严重性能。


#### 4.5　加快ALTER TABLE操作的速度？
>MySQL的ALTER TABLE操作的性能对大表来说是个大问题。MySQL执行大部分修改表结构操作的方法是用新的结构创建一个空表，从旧表中查出所有数据插入新表，然后删除旧表。  
大部分ALTER TABLE操作将导致MySQL服务中断。我们会展示一些在DDL操作时有用的技巧，但这是针对一些特殊的场景而言的。对常见的场景，能使用的技巧只有两种：一种是先在一台不提供服务的机器上执行ALTER TABLE操作，然后和提供服务的主库进行切换；另外一种技巧是“影子拷贝”。影子拷贝的技巧是用要求的表结构创建一张和源表无关的新表，然后通过重命名和删表操作交换两张表。  

~对于ALTER TABLE操作，上面提供了两个方法，对于方法一，操作起来可能要复杂一些，假设你在备库上执行ALTER TABLE操作，当次操作执行完毕之后，你需要停止主库的写入服务，并等待备库完成数据的同步，之后再完成主备库的切换。而方法二，有一些工具帮助来实现，如果自己手动去做，也将面对一些问题，如何完整的复制数据，因为旧表也会有新的数据插入和记录的更新。

#### 4.5.1　只修改.frm文件
>基本的技术是为想要的表结构创建一个新的.frm文件，然后用它替换掉已经存在的那张表的.frm文件，像下面这样：  
1，创建一张有相同结构的空表，并进行所需要的修改（例如增加ENUM常量）。  
2，执行FLUSH TABLES WITH READ LOCK。这将会关闭所有正在使用的表，并且禁止任何表被打开。  
3，交换.frm文件。  
4，执行UNLOCK TABLES来释放第2步的读锁。  

~这种方法比较简单，但是需要承担一定的风险，这并不是文档化的方法，需要着这样做之前先对数据进行备份，另外此类表结构的修改需要兼容于现有的数据，最好应用到线上环境之前先在开发环境测试一下。

### 第5章　创建高性能的索引
#### 5.1.1　索引的类型
>创建自定义哈希索引。如果存储引擎不支持哈希索引，则可以模拟像InnoDB一样创建哈希索引，这可以享受一些哈希索引的便利，例如只需要很小的索引就可以为超长的键创建索引。  
思路很简单：在B-Tree基础上创建一个伪哈希索引。这和真正的哈希索引不是一回事，因为还是使用B-Tree进行查找，但是它使用哈希值而不是键本身进行索引查找。你需要做的就是在查询的WHERE子句中手动指定使用哈希函数。  
如果数据表非常大，CRC32()会出现大量的哈希冲突，则可以考虑自己实现一个简单的64位哈希函数。这个自定义函数要返回整数，而不是字符串。一个简单的办法可以使用MD5()函数返回值的一部分来作为自定义哈希函数。  
要避免冲突问题，必须在WHERE条件中带入哈希值和对应列值。  
```shell
mysql>SELECT id FROM url WHERE url_ crc= CRC32(" http:// www. mysql. com") 
-> AND url=" http:// www. mysql. com";
```

~InnoDB上创建哈希索引的方法


#### 5.2　索引的优点#
>如果表的数量特别多，可以建立一个元数据信息表，用来查询需要用到的某些特性。例如执行那些需要聚合多个应用分布在多个表的数据的查询，则需要记录“哪个用户的信息存储在哪个表中”的元数据，这样在查询时就可以直接忽略那些不包含指定用户信息的表。对于大型系统，这是一个常用的技巧。事实上，Infobright就是使用类似的实现。对于TB级别的数据，定位单条记录的意义不大，所以经常会使用块级别元数据技术来替代索引。

~我的理解是这样的，将一些元数据存放在元数据信息表中，那么什么是元数据呢，就是数据分布信息，比如数据存放在那个库，那个表中这样的信息。这样在查询数据时，就可以直接到对应的库，表中查找。


#### 5.3.2　前缀索引和索引选择性
>前缀索引是一种能使索引更小、更快的有效办法，但另一方面也有其缺点：MySQL无法使用前缀索引做ORDER BY和GROUP BY，也无法使用前缀索引做覆盖扫描。

~只要仔细想想，这两个缺点是前缀索引所固有的。


#### 5.3.4　选择合适的索引列顺序
>当不需要考虑排序和分组时，将选择性最高的列放在前面通常是很好的。这时候索引的作用只是用于优化WHERE条件的查找。在这种情况下，这样设计的索引确实能够最快地过滤出需要的行，对于在WHERE子句中只使用了索引部分前缀列的查询来说选择性也更高。然而，性能不只是依赖于所有索引列的选择性（整体基数），也和查询条件的具体值有关，也就是和值的分布有关。

~影响性能的因素有很多，索引列的选择性只是其中之一。


#### 5.3.5　聚簇索引
>对于高并发工作负载，在InnoDB中按主键顺序插入可能会造成明显的争用。主键的上界会成为“热点”。因为所有的插入都发生在这里，所以并发插入可能导致间隙锁竞争。另一个热点可能是AUTO_INCREMENT锁机制；如果遇到这个问题，则可能需要考虑重新设计表或者应用，或者更改innodb_autoinc_lock_mode配置。如果你的服务器版本还不支持innodb_autoinc_lock_mode参数，可以升级到新版本的InnoDB，可能对这种场景会工作得更好。

~这是特别需要注意的。



#### 5.3.7　使用索引扫描来做排序
>MySQL有两种方式可以生成有序的结果：通过排序操作；或者按索引顺序扫描。如果EXPLAIN出来的type列的值为“index”，则说明MySQL使用了索引扫描来做排序（不要和Extra列的“Using index”搞混淆了）。  
只有当索引的列顺序和ORDER BY子句的顺序完全一致，并且所有列的排序方向（倒序或正序）都一样时，MySQL才能够使用索引来对结果做排序。如果查询需要关联多张表，则只有当ORDER BY子句引用的字段全部为第一个表时，才能使用索引做排序。ORDER BY子句和查找型查询的限制是一样的：需要满足索引的最左前缀的要求；否则，MySQL都需要执行排序操作，而无法利用索引排序。

~使用索引排序的注意项。

#### 5.3.9　冗余和重复索引
>还有一种情况是将一个索引扩展为（A，ID），其中ID是主键，对于InnoDB来说主键列已经包含在二级索引中了，所以这也是冗余的。

~这个是我没有想到的。


#### 5.3.11　索引和锁
>换句话说，底层存储引擎的操作是“从索引的开头开始获取满足条件actor_id<5的记录”，服务器并没有告诉InnoDB可以过滤第1行的WHERE条件。注意到EXPLAIN的Extra列出现了“Using where”，这表示MySQL服务器将存储引擎返回行以后再应用WHERE过滤条件。  
```shell
mysql> EXPLAIN SELECT actor_id FROM sakila.actor
-> WHERE actor_id < 5 AND actor_id <> 1 FOR UPDATE;
+----+-------------+-------+-------+---------+--------------------------+
| id | select_type | table | type | key | Extra |
+----+-------------+-------+-------+---------+--------------------------+
| 1 | SIMPLE | actor | range | PRIMARY | Using where; Using index |
+----+-------------+-------+-------+---------+--------------------------+
```

~那么是不是可以这样理解呢，当Extra出现‘Using where; Using index’，是不是就表示存储引擎利用索引获取数据后，然后MYSQL服务器再对返回的数据进行筛选呢，我想是这样的。




#### 5.4.1　支持多种过滤条件
>这个“诀窍”就是：如果某个查询不限制性别，那么可以通过在查询条件中新增AND SEX IN（'m','f'）来让MySQL选择该索引。这样写并不会过滤任何行，和没有这个条件时返回的结果相同。但是必须加上这个列的条件，MySQL才能够匹配索引的最左前缀。这个“诀窍”在这类场景中非常有效，但如果列有太多不同的值，就会让IN()列表太长，这样做就不行了。  
这里提到可以在索引中加入更多的列，并通过IN()的方式覆盖那些不在WHERE子句中的列。但这种技巧也不能滥用，否则可能会带来麻烦。因为每额外增加一个IN()条件，优化器需要做的组合都将以指数形式增加，最终可能会极大地降低查询性能。

~如何更好的利用索引（更好的利用INNODB中的最左前缀索引）。


>无论如何创建索引，这种查询都是个严重的问题。因为随着偏移量的增加，MySQL需要花费大量的时间来扫描需要丢弃的数据。反范式化、预先计算和缓存可能是解决这类查询的仅有策略。一个更好的办法是限制用户能够翻页的数量，实际上这对用户体验的影响不大，因为用户很少会真正在乎搜索结果的第10000页。  
优化这类索引的另一个比较好的策略是使用延迟关联，通过使用覆盖索引查询返回需要的主键，再根据这些主键关联原表获得需要的行。这可以减少MySQL扫描那些需要丢弃的行数。下面这个查询显示了如何高效地使用（sex，rating）索引进行排序和分页：
```shell
mysql> SELECT < cols> FROM profiles INNER JOIN ( 
-> SELECT < primary key cols> FROM profiles 
-> WHERE x. sex=' M' ORDER BY rating LIMIT 100000, 10 
-> ) AS x USING(< primary key cols>);
```

~如何解决分页中的大量翻页的问题，这里提供了一些方法。有意思有两个，一个是限制大量翻页，二是通过先查询页面记录的ID值，然后再通过ID值获取页面的记录。



#### 5.5.1　找到并修复损坏的表#
>损坏的索引会导致查询返回错误的结果或者莫须有的主键冲突等问题，严重时甚至还会导致数据库的崩溃。如果你遇到了古怪的问题——例如一些不应该发生的错误——可以尝试运行CHECK TABLE来检查是否发生了表损坏（注意有些存储引擎不支持该命令；而有些引擎则支持以不同的选项来控制完全检查表的方式）。CHECK TABLE通常能够找出大多数的表和索引的错误。  
可以使用REPAIR TABLE命令来修复损坏的表，但同样不是所有的存储引擎都支持该命令。如果存储引擎不支持，也可通过一个不做任何操作（no-op）的ALTER操作来重建表，例如修改表的存储引擎为当前的引擎。下面是一个针对InnoDB表的例子：  
```sql
mysql> ALTER TABLE innodb_ tbl ENGINE= INNODB;
```
>如果InnoDB引擎的表出现了损坏，那么一定是发生了严重的错误，需要立刻调查一下原因。InnoDB一般不会出现损坏。InnoDB的设计保证了它并不容易被损坏。如果发生损坏，一般要么是数据库的硬件问题例如内存或者磁盘问题（有可能），要么是由于数据库管理员的错误例如在MySQL外部操作了数据文件（有可能），抑或是InnoDB本身的缺陷（不太可能）。常见的类似错误通常是由于尝试使用rsync备份InnoDB导致的。不存在什么查询能够让InnoDB表损坏，也不用担心暗处有“陷阱”。如果某条查询导致InnoDB数据的损坏，那一定是遇到了bug，而不是查询的问题。

~CHECK TABLE命令和ALTER的替代写法，需要知道这个东西，这样当发生了数据表错误的时候，还知道通过这个命令去解决问题。


#### 5.5.2　更新索引统计信息
>InnoDB会在表首次打开，或者执行ANALYZE TABLE，抑或表的大小发生非常大的变化（大小变化超过十六分之一或者新插入了20亿行都会触发）的时候计算索引的统计信息。  
一旦关闭索引统计信息的自动更新，那么就需要周期性地使用ANALYZE TABLE来手动更新。否则，索引统计信息就会永远不变。如果数据分布发生大的变化，可能会出现一些很糟糕的执行计划。

~优化器需要用到索引统计信息，一是构建索引统计信息需要花费时间，二是过期的索引统计信息会导致一些查询的性能问题，所以在优化时，需要兼具这两者。


#### 5.5.3　减少索引和数据的碎片
>可以通过执行OPTIMIZE TABLE或者导出再导入的方式来重新整理数据。这对多数存储引擎都是有效的。对于一些存储引擎如MyISAM，可以通过排序算法重建索引的方式来消除碎片。老版本的InnoDB没有什么消除碎片化的方法。不过最新版本InnoDB新增了“在线”添加和删除索引的功能，可以通过先删除，然后再重新创建索引的方式来消除索引的碎片化。  
对于那些不支持OPTIMIZE TABLE的存储引擎，可以通过一个不做任何操作（no-op）的ALTER TABLE操作来重建表。只需要将表的存储引擎修改为当前的引擎即可：
```shell
mysql>ALTER TABLE <table> ENGINE = <engine>;
```

~减少索引和数据的碎片的方法，需要有所了解。


#### 5.6　总结#
>在选择索引和编写利用这些索引的查询时，有如下三个原则始终需要记住：  
单行访问是很慢的。特别是在机械硬盘存储中（SSD的随机I/O要快很多，不过这一点仍然成立）。如果服务器从存储中读取一个数据块只是为了获取其中一行，那么就浪费了很多工作。最好读取的块中能包含尽可能多所需要的行。使用索引可以创建位置引用以提升效率。  
按顺序访问范围数据是很快的，这有两个原因。第一，顺序I/O不需要多次磁盘寻道，所以比随机I/O要快很多（特别是对机械硬盘）。第二，如果服务器能够按需要顺序读取数据，那么就不再需要额外的排序操作，并且GROUP BY查询也无须再做排序和将行按组进行聚合计算了。  
索引覆盖查询是很快的。如果一个索引包含了查询需要的所有列，那么存储引擎就不需要再回表查找行。这避免了大量的单行访问，而上面的第1点已经写明单行访问是很慢的。  
总的来说，编写查询语句时应该尽可能选择合适的索引以避免单行查找、尽可能地使用数据原生顺序从而避免额外的排序操作，并尽可能使用索引覆盖查询。

~索引优化的一个总的指导原则。其中说‘单行访问是很慢的’让有我点意外，其实想想也是如此。

### 第6章　查询性能优化
#### 6.1　为什么查询速度会慢
>通常来说，查询的生命周期大致可以按照顺序来看：从客户端到服务器，然后在服务器上进行解析，生成执行计划，执行，并返回结果给客户端。其中“执行”可以认为是整个生命周期中最重要的阶段，这其中包括了大量为了检索数据到存储引擎的调用以及调用后的数据处理，包括排序、分组等。  
在完成这些任务的时候，查询需要在不同的地方花费时间，包括网络，CPU计算，生成统计信息和执行计划、锁等待（互斥等待）等操作，尤其是向底层存储引擎检索数据的调用操作，这些调用需要在内存操作、CPU操作和内存不足时导致的I/O操作上消耗时间。根据存储引擎不同，可能还会产生大量的上下文切换以及系统调用。

~我觉得有必要了解一些。


#### 6.2　慢查询基础：优化数据访问
>查询性能低下最基本的原因是访问的数据太多。某些查询可能不可避免地需要筛选大量数据，但这并不常见。大部分性能低下的查询都可以通过减少访问的数据量的方式进行优化。对于低效的查询，我们发现通过下面两个步骤来分析总是很有效：  
1，确认应用程序是否在检索大量超过需要的数据。这通常意味着访问了太多的行，但有时候也可能是访问了太多的列。  
2，确认MySQL服务器层是否在分析大量超过需要的数据行。

~这位慢查询优化提供了一个方向。



#### 6.3.2　切分查询
>有时候对于一个大查询我们需要“分而治之”，将大查询切分成小查询，每个查询功能完全一样，只完成一小部分，每次只返回一小部分查询结果。  
删除旧的数据就是一个很好的例子。定期地清除大量数据时，如果用一个大的语句一次性完成的话，则可能需要一次锁住很多数据、占满整个事务日志、耗尽系统资源、阻塞很多小的但重要的查询。将一个大的DELETE语句切分成多个较小的查询可以尽可能小地影响MySQL性能，同时还可以减少MySQL复制的延迟。

~切分查询，并不是提升性能，而是防止大的查询占用过多的资源，因而阻塞其他的重要的查询或操作。



#### 6.4　查询执行的基础#----
>我们可以看到当向MySQL发送一个请求的时候，MySQL到底做了些什么：  
1，客户端发送一条查询给服务器。  
2，服务器先检查查询缓存，如果命中了缓存，则立刻返回存储在缓存中的结果。否则进入下一阶段。  
3，服务器端进行SQL解析、预处理，再由优化器生成对应的执行计划。  
4，MySQL根据优化器生成的执行计划，调用存储引擎的API来执行查询。  
5，将结果返回给客户端。

~可能最为重要的就是三四步。


#### 6.4.1　MySQL客户端/服务器通信协议
>一般来说，不需要去理解MySQL通信协议的内部实现细节，只需要大致理解通信协议是如何工作的。MySQL客户端和服务器之间的通信协议是“半双工”的，这意味着，在任何一个时刻，要么是由服务器向客户端发送数据，要么是由客户端向服务器发送数据，这两个动作不能同时发生。所以，我们无法也无须将一个消息切成小块独立来发送。  
这种协议让MySQL通信简单快速，但是也从很多地方限制了MySQL。一个明显的限制是，这意味着没法进行流量控制。一旦一端开始发送消息，另一端要接收完整个消息才能响应它。

~我认为需要对此有所了解。


#### 6.4.3　查询优化处理#
>在很多数据库系统中，IN()完全等同于多个OR条件的子句，因为这两者是完全等价的。在MySQL中这点是不成立的，MySQL将IN()列表中的数据先进行排序，然后通过二分查找的方式来确定列表中的值是否满足条件，这是一个O（logn）复杂度的操作，等价地转换成OR查询的复杂度为O（n），对于IN()列表中有大量取值的时候，MySQL的处理速度将会更快。

~我需要知道。


#### 嵌套关联#？
>当前MySQL关联执行的策略很简单：MySQL对任何关联都执行嵌套循环关联操作，即MySQL先在一个表中循环取出单条数据，然后再嵌套循环到下一个表中寻找匹配的行，依次下去，直到找到所有表中匹配的行为止。然后根据各个表匹配的行，返回查询中需要的各个列。MySQL会尝试在最后一个关联表中找到所有匹配的行，如果最后一个联表无法找到更多的行以后，MySQL返回到上一层次关联表，看是否能够找到更多的匹配记录，依此类推迭代执行。

~这个很重要，也即是MySql如何执行关联查询。


>MySQL有如下两种排序算法：  
>
两次传输排序（旧版本使用）  
读取行指针和需要排序的字段，对其进行排序，然后再根据排序结果读取所需要的数据行。  
这需要进行两次数据传输，即需要从数据表中读取两次数据，第二次读取数据的时候，因为是读取排序列进行排序后的所有记录，这会产生大量的随机I/O，所以两次数据传输的成本非常高。  
>
单次传输排序（新版本使用）  
先读取查询所需要的所有列，然后再根据给定列进行排序，最后直接返回排序结果。这个算法只在MySQL4.1和后续更新的版本才引入。因为不再需要从数据表中读取两次数据，对于I/O密集型的应用，这样做的效率高了很多。另外，相比两次传输排序，这个算法只需要一次顺序I/O读取所有的数据，而无须任何的随机I/O。缺点是，如果需要返回的列非常多、非常大，会额外占用大量的空间，而这些列对排序操作本身来说是没有任何作用的。因为单条排序记录很大，所以可能会有更多的排序块需要合并。  
很难说哪个算法效率更高，两种算法都有各自最好和最糟的场景。当查询需要所有列的总长度不超过参数max_length_for_sort_data时，MySQL使用“单次传输排序”，可以通过调整这个参数来影响MySQL排序算法的选择。关于这个细节，可以参考第8章“文件排序优化”。

~理解这些东西，就是理解Mysql的底层原理，有必要。


### 第7章　MySQL高级特性
#### 7.1.1　分区表的原理
>分区表由多个相关的底层表实现，这些底层表也是由句柄对象（Handler object）表示，所以我们也可以直接访问各个分区。存储引擎管理分区的各个底层表和管理普通表一样（所有的底层表都必须使用相同的存储引擎），分区表的索引只是在各个底层表上各自加上一个完全相同的索引。从存储引擎的角度来看，底层表和一个普通表没有任何不同，存储引擎也无须知道这是一个普通表还是一个分区表的一部分。

~这就是分区表的原理，可以理解为分区表持有多个相关底层表的句柄对象吗。


#### 7.1.2　分区表的类型
>PARTITION分区子句中可以使用各种函数。但有一个要求，表达式返回的值要是一个确定的整数，且不能是一个常数。这里我们使用函数YEAR()，也可以使用任何其他的函数，如TO_DAYS()。  
MySQL还支持键值、哈希和列表分区，这其中有些还支持子分区，不过我们在生产环境中很少见到。在MySQL5.5中，还可以使用RANGE COLUMNS类型的分区，这样即使是基于时间的分区也无须再将其转化成一个整数，

~分区的类型，有必要有所了解。


#### 我们还看到的一些其他的分区技术包括：  
>1、根据键值进行分区，来减少InnoDB的互斥量竞争。  
2、使用数学模函数来进行分区，然后将数据轮询放入不同的分区。例如，可以对日期做模7的运算，或者更简单地使用返回周几的函数，如果只想保留最近几天的数据，这样分区很方便。  
3、假设表有一个自增的主键列id，希望根据时间将最近的热点数据集中存放。那么必须将时间戳包含在主键当中才行，而这和主键本身的意义相矛盾。这种情况下也可以使用这样的分区表达式来实现相同的目的：HASH（id DIV 1000000），这将为100万数据建立一个分区。这样一方面实现了当初的分区目的，另一方面比起使用时间范围分区还避免了一个问题，就是当超过一定阈值时，如果使用时间范围分区就必须新增分区。

~可以作为以后分区的参考。


#### 7.1.5　查询优化
>对于访问分区表来说，很重要的一点是要在WHERE条件中带入分区列，有时候即使看似多余的也要带上，这样就可以让优化器能够过滤掉无须访问的分区。如果没有这些条件，MySQL就需要让对应存储引擎访问这个表的所有分区，如果表非常大的话，就可能会非常慢。

~使用分区表，查询的改变。



### 第8章　优化服务器设置
#### 8.1　MySQL配置的工作原理
>首先应该知道的是MySQL从哪里获得配置信息：命令行参数和配置文件。在类UNIX系统中，配置文件的位置一般在/etc/my.cnf或者/etc/mysql/my.cnf。如果使用操作系统的启动脚本，这通常是唯一指定配置设置的地方。如果手动启动MySQL，例如在测试安装时，也可以在命令行指定设置。实际上，服务器会读取配置文件的内容，删除所有注释和换行，然后和命令行选项一起处理。

~当个提醒，不要忘记了配置来自哪里。


#### 8.1.2　设置变量的副作用
>MySQL在启动的时候，一次性分配并且初始化这块内存。如果修改这个query_cache_size 变量（即使设置为与当前一样的值），MySQL会立刻删除所有缓存的查询，重新分配这片缓存到指定大小，并且重新初始化内存。这可能花费较长的时间，在完成初始化之前服务器都无法提供服务，因为MySQL是逐个清理缓存的查询，不是一次性全部删掉。

~设置‘query_cache_size’变量的副作用。现在的Mysql版本中是不是这样处理，还得去查阅相关文档。


#### 8.1.3　入门
>应该始终通过监控来确认生产环境中变量的修改，是提高还是降低了服务器的整体性能。  
在开始改变配置之前，应该优化查询和schema，至少先做明显要做的事情，例如添加索引。如果先深入调整配置，然后修改了查询语句和schema，也许需要回头再次评估配置。请记住，除非硬件、工作负载和数据是完全静态的，否则都可能需要重新检查配置文件。

~好做法。


#### 8.1.4　通过基准测试迭代优化
>最好的办法是一次改变一个或两个变量，每次一点点，每次更改后运行基准测试，确保运行足够长的时间来确认性能是否稳定。有时结果可能会令你感到惊讶，可能把一个变量调大了一点，观察到性能提升，然后再调大一点，却发现性能大幅下降。如果变更后性能有隐患，可能是某些资源用得太多了，例如，为缓冲区分配太多内存、频繁地申请和释放内存。

~好的做法，可以用到以后的工作中。


#### 8.2　什么不该做
>首先，不要根据一些“比率”来调优。一个经典的按“比率”调优的经验法则是，键缓存的命中率应该高于某个百分比，如果命中率过低，则应该增加缓存的大小。这是非常错误的意见。无论别人怎么跟你说，缓存命中率跟缓存是否过大或过小没有关系。首先，命中率取决于工作负载——某些工作负载就是无法缓存的，不管缓存有多大——其次，缓存命中没有什么意义，我们将在后面解释原因。有时当缓存太小时，命中率比较低，增加缓存的大小确实可以提高命中率。然而，这只是个偶然情况，并不表示这与性能或适当的缓存大小有任何关系。

~纠正了我以前的错误看法和做法。


#### 8.3　创建MySQL配置文件#
>InnoDB在大多数情况下如果要运行得很好，配置大小合适的缓冲池（BufferPool）和日志文件（LogFile）是必须的。默认值都太小了。其他所有的InnoDB设置都是可选的。  
我们建议，当配置内存缓冲区的时候，宁可谨慎，而不是把它们配置得过大。如果把缓冲池配置得比它可以设的值少了20%，很可能只会对性能产生小的影响，也许就只影响几个百分点。如果设置得大了20%，则可能会造成更严重的问题：内存交换、磁盘抖动，甚至内存耗尽和硬件死机。
这里需要解释的一个选项是open_files_limit。在典型的Linux系统上我们把它设置得尽可能大。现代操作系统中打开文件句柄开销都很小。如果这个参数不够大，将会碰到经典的24号错误，“打开的文件太多（too many open files）”。

~以前不熟悉的内容，需要摘录在这里。


#### 8.4　配置内存使用
>按下面的步骤来配置内存：  
1、确定可以使用的内存上限。  
2、确定每个连接MySQL需要使用多少内存，例如排序缓冲和临时表。  
3、确定操作系统需要多少内存才够用。包括同一台机器上其他程序使用的内存，如定时任务。
4、把剩下的内存全部给MySQL的缓存，例如InnoDB的缓冲池，这样做很有意义。  

~以后就知道如何配置内存了。


#### 8.4.2　每个连接需要的内存
>相对于计算最坏情况下的开销，更好的办法是观察服务器在真实的工作压力下使用了多少内存，可以在进程的虚拟内存大小那里看到。在许多类UNIX系统里，可以观察top命令中的VIRT列，或者ps命令中的VSZ列的值。

~这个很重要，因为排除了每个连接的内存开销，才能更好的设置Innodb的缓存池的大小。这个需要自己去实验实验，看看如何获取这个值。


#### 8.4.3　为操作系统保留内存
>至少应该为操作系统保留1GB～2GB的内存——如果机器内存更多就再多预留一些。我们建议2GB或总内存的5%作为基准，以较大者为准。为了安全再额外增加一些预留，并且如果机器上还在运行内存密集型任务（如备份），则可以再多增加一些预留。

~操作系统内存预留的指导。


#### 8.4.5　InnoDB缓冲池（BufferPool）
>如果大部分都是InnoDB表，InnoDB缓冲池或许比其他任何东西更需要内存。InnoDB缓冲池并不仅仅缓存索引：它还会缓存行的数据、自适应哈希索引、插入缓冲（Insert Buffer）、锁，以及其他内部数据结构。InnoDB还使用缓冲池来帮助延迟写入，这样就能合并多个写入操作，然后一起顺序地写回。

~知道InnoDB严重依赖缓冲池就行了。


#### 8.4.6　MyISAM键缓存（KeyCaches）
>在决定键缓存需要分配多少内存之前，先去了解MyISAM索引实际上占用多少磁盘空间是很有帮助的。肯定不需要把键缓冲设置得比需要缓存的索引数据还大。  
查询INFORMATION_SCHEMA表的INDEX_LENGTH字段，把它们的值相加，就可以得到索引存储占用的空间：  
```sql
SELECT SUM(INDEX_LENGTH) FROM INFORMATION_SCHEMA.TABLES WHERE ENGINE='MYISAM';
```
>如果是类UNIX系统，也可以使用下面的命令：
```shell
$du-sch`find/path/to/mysql/data/directory/-name"*.MYI"`
```
>应该把键缓存设置得多大？不要超过索引的总大小，或者不超过为操作系统缓存保留总内存的25%～50%，以更小的为准。

~MyISAM键缓存设置依据。


#### 8.4.6　缓存未命中的次数
>从经验上来说，每秒缓存未命中的次数要更有用。假定有一个独立的磁盘，每秒可以做100个随机读。每秒5次缓存未命中可能不会导致I/O繁忙，但是每秒80次缓存未命中则可能出现问题。可以使用下面的公式来计算这个值：  
Key_reads/Uptime  
通过间隔10～100秒来计算这段时间内缓存未命中次数的增量值，可以获得当前性能的情况。下面的命令可以每10秒钟获取一次状态值的变化量：  
```shell
$ mysqladmin extended- status -r -i 10 | grey Key_ reads
```

~这个需要自己去操作操作。


>最后，即使没有任何MyISAM表，依然需要将key_buffer_size设置为较小的值，例如32M。MySQL服务器有时会在内部使用MyISAM表，例如GROUP BY语句可能会使用MyISAM做临时表。

~这个要记得，不要因为没有使用MyISAM表，就不设置键缓冲的大小。


#### MySQL键缓存块大小（Key Block Size）
>在MySQL5.1以及更新版本中，可以设置MyISAM的索引块大小跟操作系统一样，以避免写时读取。myisam_block_size变量控制着索引块大小。也可以指定每个索引的块大小，在CREATE TABLE或者CREATE INDEX语句中使用KEY_BLOCK_SIZE选项即可，但是因为同一个表的所有索引都保存在同一个文件中，因此该表所有索引的块大小都需要大于或者等于操作系统的块大小，才能避免由于边界对齐导致的写时读取。

~这个我大概理解了，比如操作系统的块大小是4K，而MyISAM的块大小为1K，那么当MyISAM修改后，需要将数据写入到磁盘时，操作系统还需要先从磁盘中读取这4K数据，然后修改其中的1K数据，因为变更数据只能在内存进行呀。如果操作系统与MyISAM的块大小相同，则直接将修改后的块保存到磁盘即可。


#### 8.4.7　线程缓存
>thread_cache_size变量指定了MySQL可以保持在缓存中的线程数。一般不需要配置这个值，除非服务器会有很多连接请求。要检查线程缓存是否足够大，可以查看Threads_created状态变量。如果我们观察到有每秒创建的新线程数少于10个的时候，通常应该尝试保持线程缓存足够大，但是实际上经常也可能看到每秒少于1个新线程的情况。  
一个好的办法是观察Threads_connected变量并且尝试设置thread_cache_size足够大以便能处理业务压力正常的波动。例如，若Threads_connected通常保持在100～120，则可以设置缓存大小为20。如果它保持在500～700，200的线程缓存应该足够大了。

~如何设置线程缓存的大小，并提供参考指标。


#### 8.4.8　表缓存（Table Cache）
>在MySQL5.1版本中，表缓存分离成两部分：一个是打开表的缓存，一个是表定义缓存（通过table_open_cache和table_defnition_cache变量来配置）。其结果是，表定义（解析.frm文件的结果）从其他资源中分离出来了，例如表描述符。打开的表依然是每个线程、每个表用的，但是表定义是全局的，可以被所有连接有效地共享。通常可以把table_definition_cache设置得足够高，以缓存所有的表定义。除非有上万张表，否则这可能是最简单的方法。

~这是一个MyISAM的配置项。


>如果遇到MySQL无法打开更多文件的错误（可以使用perror工具来检查错误号代表的含义），那么可能需要增加MySQL允许打开文件的数量。这可以通过在my.cnf文件中设置open_files_limit服务器变量来实现。

~以后遇到同样的问题，就有了解决办法。


#### 8.4.9　InnoDB数据字典（Data Dictionary）#
>另一个性能问题是第一次打开表时会计算统计信息，这需要很多I/O操作，所以代价很高。相比MyISAM，InnoDB没有将统计信息持久化，而是在每次打开表时重新计算，在打开之后，每隔一段过期时间或者遇到触发事件（改变表的内容或者查询INFORMATION_SCHEMA表，等等），也会重新计算统计信息。如果有很多表，服务器可能会花费数个小时来启动并完全预热，在这个时候服务器可能花费更多的时间在等待I/O操作，而不是做其他事。可以在PerconaServer（在MySQL5.6中也可以，但是叫做innodb_analyze_is_persistent）中打开innodb_use_sys_stats_table选项来持久化存储统计信息到磁盘，以解决这个问题。

~每次打开表时，系统都会计算统计信息，如果数据表比较多时，将会耗费不少的时间，可以通过设置相关参数持久化统计信息到磁盘。


>InnoDB打开文件和MyISAM的方式不一样，MyISAM用表缓存来持有打开表的文件描述符，而InnoDB在打开表和打开文件之间没有直接的关系。InnoDB为每个.ibd文件使用单个、全局的文件描述符。如果可以，最好把innodb_open_files的值设置得足够大以使服务器可以保持所有的.ibd文件同时打开。

~这也是一个可以优化的参数，记录在这里。


#### 8.5　配置MySQL的I/O行为
#### 8.5.1　InnoDB I/O配置#
>InnoDB使用一个后台线程智能地刷新这些变更到数据文件。这个线程可以批量组合写入，使得数据以接近顺序的方式写入，提高了效率。实际上，事务日志把数据文件的随机I/O转换为几乎顺序的日志文件和数据文件I/O。把刷新操作转移到后台使查询可以更快完成，并且缓和查询高峰时I/O系统的压力。  
整体的日志文件大小受控于innodb_log_file_size和innodb_log_files_in_group两个参数，这对写性能非常重要。日志文件的总大小是每个文件的大小之和。默认情况下，只有两个5MB的文件，总共10MB。对高性能工作来说这太小了。至少需要几百MB，或者甚至上GB的日志文件。  
InnoDB使用多个文件作为一组循环日志。通常不需要修改默认的日志数量，只修改每个日志文件的大小即可。要修改日志文件大小，需要完全关闭MySQL，将旧的日志文件移到其他地方保存，重新配置参数，然后重启。一定要确保MySQL干净地关闭了，或者还有日志文件可以保证需要应用到数据文件的事务记录，否则数据库就无法恢复了！当重启服务器的时候，查看MySQL的错误日志。在重启成功之后，才可以删除旧的日志文件。

~为什么需要事物日志文件？如何设置事物日志文件的大小？


>当InnoDB变更任何数据时，会写一条变更记录到内存日志缓冲区。在缓冲满的时候、事务提交的时候，或者每一秒钟，InnoDB都会刷写缓冲区的内容到磁盘日志文件——无论上述三个条件哪个先达到。如果有大事务，增加日志缓冲区（默认1MB）大小可以帮助减少I/O。变量innodb_log_buffer_size可以控制日志缓冲区的大小。  
通常不需要把日志缓冲区设置得非常大。推荐的范围是1MB～8MB，一般来说足够了，除非要写很多相当大的BLOB记录。相对于InnoDB的普通数据，日志条目是非常紧凑的。它们不是基于页的，所以不会浪费空间来一次存储整个页。InnoDB也使得日志条目尽可能地短。有时甚至会保存为函数号和C函数的参数！

~对于我来说，日志缓冲区刷新的条件很重要。


#### 日志文件大小设置#
>可以通过检查SHOW INNODB STATUS的输出中LOG部分来监控InnoDB的日志和日志缓冲区的I/O性能，通过观察Innodb_os_log_written状态变量来查看InnoDB对日志文件写出了多少数据。一个好用的经验法则是，查看10～100秒间隔的数字，然后记录峰值。可以用这个来判断日志缓冲是否设置得正好。例如，若看到峰值是每秒写100KB数据到日志，那么1MB的日志缓冲可能足够了。也可以使用这个衡量标准来决定日志文件设置多大会比较好。如果峰值是100KB/s，那么256MB的日志文件足够存储至少2560秒的日志记录。这看起来足够了。作为一个经验法则，日志文件的全部大小，应该足够容纳服务器一个小时的活动内容。

~设置日志缓冲区的大小的经验法则。


#### innodb_flush_log_at_trx_commit#
>高性能事务处理需要的最佳配置是把innodb_flush_log_at_trx_commit设置为1且把日志文件放到一个有电池保护的写缓存的RAID卷中。这兼顾了安全和速度。事实上，我们敢说任何希望能扛过高负荷工作负载的产品数据库服务器，都需要有这种类型的硬件。  
innodb_flush_log_at_trx_commit的值为1是将日志缓冲写到日志文件，并且每次事务提交都刷新到持久化存储。这是默认的（并且是最安全的）设置，该设置能保证不会丢失任何已经提交的事务，除非磁盘或者操作系统是“伪”刷新。

~innodb_flush_log_at_trx_commit的配置项以及它的取值。


#### InnoDB怎样打开和刷新日志以及数据文件
>使用innodb_fush_method选项可以配置InnoDB如何跟文件系统相互作用。从名字来看，会以为只能影响InnoDB怎么写数据，实际上还影响了InnoDB怎么读数据。
fdatasync  
这在非Windows系统上是默认值：InnoDB用fsync()来刷新数据和日志文件。
InnoDB通常用fsync()代替fdatasync()，即使这个值似乎表达的是相反的意思。fdatasync()跟fsync()相似，但是只刷新文件的数据，而不包括元数据（最后修改时间，等等）。因此，fsync()会导致更多的I/O。然而InnoDB的开发者都很保守，他们发现某些场景下fdatasync()会导致数据损坏。InnoDB决定了哪些方法可以更安全地使用，有一些是编译时设置的，也有一些是运行时设置的。它使用尽可能最快的安全方法。  
O_DIRECT
InnoDB对数据文件使用O_DIRECT标记或directio()函数，这依赖于操作系统。这个设置并不影响日志文件并且不是在所有的类UNIX系统上都有效。但至少GNU/Linux、FreeBSD，以及Solaris（5.0以后的新版本）是支持的。不像O_DSYNC标记，它同时会影响读和写。  
这个设置依然使用fsync()来刷新文件到磁盘，但是会通知操作系统不要缓存数据，也不要用预读。这个选项完全关闭了操作系统缓存，并且使所有的读和写都直接通过存储设备，避免了双重缓冲。
那么下面是一些建议：如果使用类UNIX操作系统并且RAID控制器带有电池保护的写缓存，我们建议使用O_DIRECT。如果不是这样，默认值或者O_DIRECT都可能是最好的选择，具体要看应用类型。  

~如何设置innodb_fush_method参数以及相关参数的说明。



#### InnoDB表空间#
>InnoDB把数据保存在表空间内，本质上是一个由一个或多个磁盘文件组成的虚拟文件系统。InnoDB用表空间实现很多功能，并不只是存储表和索引。它还保存了回滚日志（旧版本行）、插入缓冲（Insert Buffer）、双写缓冲（Doublewrite Buffer），以及其他内部数据结构。  
配置表空间。通过innodb_data_file_path配置项可以定制表空间文件。这些文件都放在innodb_data_home_dir指定的目录下。这是一个例子：  
```properties
innodb_data_home_dir=/var/lib/mysql/
innodb_data_file_path=ibdata1:1G;ibdata2:1G;ibdata3:1G
```
>innodb_file_per_table选项让InnoDB为每张表使用一个文件，MySQL4.1和之后的版本都支持。它在数据字典存储为“表名.ibd”的数据。这使得删除一张表时回收空间简单多了，并且可以容易地分散表到不同的磁盘上。  
即使打开innodb_file_per_table选项，依然需要为回滚日志和其他系统数据创建共享表空间。没有把所有数据存在其中是明智的做法，但最好还是关闭它的自动增长，因为无法在不重新导入全部数据的情况下给共享表空间瘦身。  
什么是最终的建议？我们建议使用innodb_file_per_table并且给共享表空间设置大小范围，这样可以过得舒服点（不用处理那些空间回收的事）。

~InnoDB表空间的配置建议以及相关的配置项的说明。


>如果有个很大的回滚日志并且表空间因此增长很快，可以强制MySQL减速来使InnoDB的清理线程可以跟得上。这听起来不怎么样，但是没办法。否则，InnoDB将保持数据写入，填充磁盘直到最后磁盘空间爆满，或者表空间大于定义的上限。  
为了控制写入速度，可以设置innodb_max_purge_lag变量为一个大于0的值。这个值表示InnoDB开始延迟后面的语句更新数据之前，可以等待被清除的最大的事务数量。你必须知道工作负载以决定一个合理的值。例如，事务平均影响1KB的行，并且可以容许表空间里有100MB的未清理行，那么可以设置这个值为100000。

~控制写入速度。



#### 双写缓冲（Doublewrite Buffer）
>InnoDB用双写缓冲来避免页没写完整所导致的数据损坏。当一个磁盘写操作不能完整地完成时，不完整的页写入就可能发生，16KB的页可能只有一部分被写到磁盘上。有多种多样的原因（崩溃、Bug，等等）可能导致页没有写完整。双写缓冲在这种情况发生时可以保证数据完整性。  
双写缓冲是表空间一个特殊的保留区域，在一些连续的块中足够保存100个页。本质上是一个最近写回的页面的备份拷贝。当InnoDB从缓冲池刷新页面到磁盘时，首先把它们写（或者刷新）到双写缓冲，然后再把它们写到其所属的数据区域中。这可以保证每个页面的写入都是原子并且持久化的。  
这意味着每个页都要写两遍？是的，但是因为InnoDB写页面到双写缓冲是顺序的，并且只调用一次fsync()刷新到磁盘，所以实际上对性能的冲击是比较小的——通常只有几个百分点。更重要的是，这个策略允许日志文件更加高效。因为双写缓冲给了InnoDB一个非常牢固的保证，数据页不会损坏，InnoDB日志记录没必要包含整个页，它们更像是页面的二进制变化量。  

~双写缓冲只是保证数据的不会损坏，就是出现了数据非完整性的写入时，也能够修复数据。


#### 其他的I/O配置项
>sync_binlog选项控制MySQL怎么刷新二进制日志到磁盘。默认值是0，意味着MySQL并不刷新，由操作系统自己决定什么时候刷新缓存到持久化设备。如果这个值比0大，它指定了两次刷新到磁盘的动作之间间隔多少次二进制日志写操作（如果autocommit被设置了，每个独立的语句都是一次写，否则就是一个事务一次写）。把它设置为0和1以外的值是很罕见的。  
如果没有设置sync_binlog为1，那么崩溃以后可能导致二进制日志没有同步事务数据。这可以轻易地导致复制中断，并且使得及时恢复变得不可能。无论如何，可以把这个值设置为1来获得安全的保障。这样就会要求MySQL同步把二进制日志和事务日志这两个文件刷新到两个不同的位置。这可能需要磁盘寻道，相对来说是个很慢的操作。  
像InnoDB日志文件一样，把二进制日志放到一个带有电池保护的写缓存的RAID卷，可以极大地提升性能。事实上，写和刷新二进制日志缓存其实比InnoDB事务日志要昂贵多了，因为不像InnoDB事务日志，每次写二进制日志都会增加它们的大小。这需要每次写入文件系统都更新元信息。所以，设置sync_binlog=1可能比innodb_fush_log_at_trx_commit=1对性能的损害要大得多，尤其是网络文件系统，例如NFS。  

~可以考虑一下是否需要设置此参数值。

#### 8.5.2　MyISAM的I/O配置
>MyISAM通常每次写操作之后就把索引变更刷新磁盘。如你打算在一张表上做很多修改，那么毫无疑问，批量操作会更快一些。一种办法是用LOCK TABLES延迟写入，直到解锁这些表。这是个提升性能的很有价值的技巧，因为它使得你精确控制哪些写被延迟，以及什么时候把它们刷到磁盘。可以精确延迟那些希望延迟的语句。  
通过设置delay_key_write变量，也可以延迟索引的写入。如果这么做，修改的键缓冲块直到表被关闭才会刷新。

~如果使用MyISAM表，了解这个很有帮助。


>我们建议打开这个选项（myisam_recover，控制MyISAM怎样寻找和修复错误。），尤其是只有一些小的MyISAM表时。服务器运行着一些损坏的MyISAM表是很危险的，因为它们有时可以导致更多数据损坏，甚至服务器崩溃。然而，如果有很大的表，原子恢复是不切实际的：它导致服务器打开所有的MyISAM表时都会检查和修复，这是低效的做法。在这段时间，MySQL会阻止连接做任何工作。如果有一大堆的MyISAM表，比较好的主意还是启动后用CHECK TABLES和REPAIR TABLES命令来做，这样对服务器影响比较少。不管哪种方式，检查和修复表都是很重要的。

~对于表的检查和修复，不同的场景提供了不同的建议。


#### 8.6　配置MySQL并发
#### 8.6.1　InnoDB并发配置
>InnoDB有自己的“线程调度器”控制线程怎么进入内核访问数据，以及它们在内核中一次可以做哪些事。最基本的限制并发的方式是使用innodb_thread_concurrency变量，它会限制一次性可以有多少线程进入内核，0表示不限制。如果在旧的MySQL版本里有InnoDB并发问题，这个变量是最重要的配置之一。  
在任何架构和业务压力下，给这个变量设置个“靠谱”的值都很重要，理论上，下面的公式可以给出一个这样的值：  
并发值=CPU数量 * 磁盘数量 * 2  
但是在实践中，使用更小的值会更好一点。必须做实验来找出适合系统的最好的值。  
如果已经进入内核的线程超过了允许的数量，新的线程就无法再进入内核。InnoDB使用两段处理来尝试让线程尽可能高效地进入内核。两段策略减少了因操作系统调度引起的上下文切换。线程第一次休眠innodb_thread_sleep_delay微秒，然后再重试。如果它依然不能进入内核，则放入一个等待线程队列，让操作系统来处理。

~控制进入内核线程的数量。



#### 8.6.2　MyISAM并发配置
>理解MyISAM是怎样删除和插入行的，是非常重要的。删除操作不会重新整理整个表，它们只是把行标记为删除，在表中留下“空洞”。MyISAM倾向于在可能的时候填满这些空洞，在插入行时重新利用这些空间。如果没有空洞了，它就把新行插入表的末尾。  
通过设置concurrent_insert这个变量，可以配置MyISAM打开并发插入，可以配置为如下值：  
0，MyISAM不允许并发插入，所有插入都会对表加互斥锁。  
1，这是默认值。只要表中没有空洞，MyISAM就允许并发插入。  
2，这个值在MySQL5.0以及更新版本中有效。它强制并发插入到表的末尾，即使表中有空洞。如果没有线程从表中读取数据，MySQL将把新行放在空洞里。使用这个设置通常会使表更加碎片化。
也可以让INSERT、REPLACE、DELETE、以及UPDATE语句的优先级比SELECT语句更低，设置low_priority_updates选项就可以了。这相当于把LOW_PRIORITY修饰符应用到全局UPDATE语句。当使用MyISAM时，这是个非常重要的选项，这让SELECT语句可以获得相当好的并发度，否则一小部分获取高优先级写锁的语句就可能导致SELECT无法获取资源。  

~可以设置MyISAM在什么情况下支持并发插入，另外还可以通过减低写操作语句的优先级，来提升查询语句的并发度。


#### 8.7　基于工作负载的配置
#### 8.7.1　优化BLOB和TEXT的场景
>一个最重要的注意事项是，服务器不能在内存临时表中存储BLOB值，因此，如果一个查询涉及BLOB值，又需要使用临时表——不管它多小——它都会立即在磁盘上创建临时表。这样效率很低，尤其是对小而快的查询。临时表可能是查询中最大的开销。  
有两种办法来减轻这个不利的情况：通过SUBSTRING()函数把值转换为VARCHAR，或者让临时表更快一些。  
让临时表运行更快的最好方式是，把它们放在基于内存的文件系统（GNU/Linux上是tmpfs）。这会降低一些开销，尽管这依然比内存表慢许多。  

~我可能也会遇到此类情形，可以做个参考。


#### 变长列#
>对于很长的变长列（例如，BLOB、TEXT，以及长字符列），InnoDB存储一个768字节的前缀在行内。如果列的值比前缀长，InnoDB会在行外分配扩展存储空间来存剩下的部分。它会分配一个完整的16KB的页，像其他所有的InnoDB页面一样，每个列都有自己的页面（不同的列不会共享扩展存储空间）。InnoDB一次只为一个列分配一个页的扩展存储空间，直到使用了超过32个页以后，就会一次性分配64个页面。  
注意，我们说过InnoDB可能会分配扩展存储空间。如果总的行长（包括大字段的完整长度）比InnoDB的最大行长限制要短（比8KB小一些），InnoDB将不会分配扩展存储空间，即使大字段（Longcolumn）的长度超过了前缀长度。  
最后，当InnoDB更新存储在扩展存储空间中的大字段时，将不会在原来的位置更新。而是会在会在扩展存储空间中写一个新值到一个新的位置，并且不会删除旧的值。  

~InnoDB如何存储BLOB类型的列以及修改BLOB列。


1、如果一张表里有很多大字段，最好是把它们组合起来单独存到一个列里面，比如说用XML文档格式存储。这让所有的大字段共享一个扩展存储空间，这比每个字段用自己的页要好。
2、有时候可以把大字段用COMPRESS()压缩后再存为BLOB，或者在发送到MySQL前在应用程序中进行压缩，这可以获得显著的空间优势和性能收益。
#使用BLOB字段的建议。


#### 8.8　完成基本配置
>max_connections#  
把max_connections设置得足够高，以容纳正常可能达到的负载，并且要足够安全，能保证允许你登录和管理服务器。例如，若认为正常情况将有300或者更多连接，则可以设置为500或者更多。如果不知道将会有多少连接，500也不是一个不合理的起点。默认值是100，对大部分应用来说这都不够。  
要时时小心可能遇到连接限制的突然袭击。例如，若重新启动应用服务器，可能没有把它的连接关闭干净，同时MySQL可能没有意识到它们已经被关闭了。当应用服务器重新开始运转，并试图打开到数据库的连接，就可能由于挂起的连接还没有超时，而使新连接被拒绝。  
观察Max_used_connections状态变量随着时间的变化。这个是高水位标记，可以告诉你服务器连接是不是在某个时间点有个尖峰。如果这个值达到了max_connections，说明客户端至少被拒绝了一次，并且当它重现的时候，应该使用第3章中的技巧来抓取服务器的活动状态。

~很重要


#### 8.9　安全和稳定的设置
>expire_logs_days  
如果启用了二进制日志，应该打开这个选项，可以让服务器在指定的天数之后清理旧的二进制日志。如果不启用，最终服务器的空间会被耗尽，导致服务器卡住或崩溃。我们建议把这个选项设置得足够从两个备份之前恢复（在最近的备份失败的情况下）。即使每天都做备份，还是建议留下7～14天的二进制日志。从我们的经验来看，当遇到一些不常见的问题时，你会感谢有这一两个星期的二进制日志。例如重搭一个备机再次尝试赶上主库。应该保持足够多的二进制日志，遇到这些情况时可以给自己一些呼吸的空间。  
max_allowed_packet  
这个设置防止服务器发送太大的包，也会控制多大的包可以被接收。默认值可能太小了，但设置得太大也可能有危险。如果设置得太小，有时复制上会出问题，通常表现为备库不能接收主库发过来的复制数据。你也许需要增加这个设置到16MB或者更大。这些文档里没有，但这个选项也控制在一个用户定义的变量的最大值，所以如果需要非常大的变量，要小心——如果超过这个变量的大小，它们可能被截断或者设置为NULL。  
max_connect_errors  
如果有时网络短暂抽风了，或者应用配置出现错误，或者有另外的问题，如权限，在短暂的时间内不断地尝试连接，客户端可能被列入黑名单，然后将无法连接，直到再次刷新主机缓存。这个选项的默认设置太小了，很容易导致问题。你也许希望增加这个值，实际上，如果知道服务器可以充分抵御蛮力攻击，可以把这个值设得非常大，以有效地禁用主机黑名单。  
skip_name_resolve  
这个选项禁用了另一个网络相关和鉴权认证相关的陷阱：DNS查找。DNS是MySQL连接过程中的一个薄弱环节。当连接服务器时，默认情况下，它试图确定连接和使用的主机的主机名，作为身份验证凭据的一部分。（就是说，你的凭据是用户名，主机名、以及密码——并不只是用户名和密码）但是验证主机来源，服务器需要执行DNS的正向和反向查找。要是DNS有问题就悲剧了，在某些时间点这是必然的事。当发生这样的情况时，所有事都会堆积起来，最终导致连接超时。为了避免这种情况，我们强烈建议设置这个选项，在验证时关闭DNS查找。如果这么做，需要把基于主机名的授权改为用IP地址、通配符，或者特定主机名“localhost”，因为基于主机名的账号会被禁用。  
read_only  
这个选项禁止没有特权的用户在备库做变更，只接受从主库传输过来的变更，不接受从应用来的变更。我们强烈建议把备库设置为只读模式。  
skip_slave_start  
这个选项阻止MySQL试图自动启动复制。因为在不安全的崩溃或其他问题后，启动复制是不安全的，所以需要禁用自动启动，用户需要手动检查服务器，并确定它是安全的之后再开始复制。  
slave_net_timeout  
这个选项控制备库发现跟主库的连接已经失败并且需要重连之前等待的时间。默认值是一个小时，太长了。设置为一分钟或更短。  
sync_master_info、sync_relay_log、sync_relay_log_info  
这些选项，在MySQL5.5以及更新版本中可用，解决了复制中备库长期存在的问题：不把它们的状态文件同步到磁盘，所以服务器崩溃后可能需要人来猜测复制的位置实际上在主库是哪个位置，并且可能在中继日志（RelayLog）里有损坏。这些选项使得备库崩溃后，更容易从崩溃中恢复。这些选项默认是不打开的，因为它们会导致备库额外的fsync()操作，可能会降低性能。如果有很好的硬件，我们建议打开这些选项，如果复制中出现fsync()造成的延时问题，就应该关闭它们。

~几个非常重要的配置选项。



#### 8.10　高级InnoDB设置#
>innodb  
这个看似平淡无奇的选项实际上非常重要，如果把这个值设置为FORCE，只有在InnoDB可以启动时，服务器才会启动。如果使用InnoDB作为默认存储引擎，这一定是你期望的结果。你应该不会希望在InnoDB失败（例如因为错误的配置而导致的不可启动）的情况下启动服务器，因为写的不好的应用可能之后会连接到服务器，导致一些无法预知的损失和混乱。最好是整个服务器都失败，强制你必须查看错误日志，而不是以为服务器正常启动了。  
innodb_old_blocks_time  
InnoDB有个两段缓冲池LRU（最近最少使用）链表，设计目的是防止换出长期使用很多次的页面。像mysqldump产生的这种一次性的（大）查询，通常会读取页面到缓冲池的LRU列表，从中读取需要的行，然后移动到下一页。理论上，两段LRU链表将阻止此页取代很长一段时间内都需要用到的页面被放入“年轻（Young）”子链表，并且只在它已被浏览过多次后将其移动到“年老（Old）”子链表。但是InnoDB默认没有配置为防止这种情况，因为页内有很多行，所以从页面读取的行的多次访问，会导致它立即被转移到“年老（Old）”子链表，对那些需要长时间缓存的页面带来换出的压力。
这个变量指定一个页面从LRU链表的“年轻”部分转移到“年老”部分之前必须经过的毫秒数。默认情况下它设置为0，将它设为诸如1000毫秒（一秒）这样的小一点的值，在我们的基准测试中已被证明非常有效。

~几个非常重要的配置选项。

#### 8.11　总结
>如果使用的是InnoDB，最重要的选项是下面这两个：innodb_buffer_pool_size，innodb_log_file_size。恭喜你——你解决了我们见过的真实存在的配置问题中的绝大部分！

~恩，记得这这两个选项。
