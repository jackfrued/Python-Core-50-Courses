## 第42课：深入MySQL

### 索引

索引是关系型数据库中用来提升查询性能最为重要的手段。关系型数据库中的索引就像一本书的目录，我们可以想象一下，如果要从一本书中找出某个知识点，但是这本书没有目录，这将是意见多么可怕的事情！我们估计得一篇一篇的翻下去，才能确定这个知识点到底在什么位置。创建索引虽然会带来存储空间上的开销，就像一本书的目录会占用一部分篇幅一样，但是在牺牲空间后换来的查询时间的减少也是非常显著的。

MySQL 数据库中所有数据类型的列都可以被索引。对于MySQL 8.0 版本的 InnoDB 存储引擎来说，它支持三种类型的索引，分别是 B+ 树索引、全文索引和 R 树索引。这里，我们只介绍使用得最为广泛的 B+ 树索引。使用 B+ 树的原因非常简单，因为它是目前在基于磁盘进行海量数据存储和排序上最有效率的数据结构。B+ 树是一棵[平衡树](https://zh.wikipedia.org/zh-cn/%E5%B9%B3%E8%A1%A1%E6%A0%91)，树的高度通常为3或4，但是却可以保存从百万级到十亿级的数据，而从这些数据里面查询一条数据，只需要3次或4次 I/O 操作。

B+ 树由根节点、中间节点和叶子节点构成，其中叶子节点用来保存排序后的数据。由于记录在索引上是排序过的，因此在一个叶子节点内查找数据时可以使用二分查找，这种查找方式效率非常的高。当数据很少的时候，B+ 树只有一个根节点，数据也就保存在根节点上。随着记录越来越多，B+ 树会发生分裂，根节点不再保存数据，而是提供了访问下一层节点的指针，帮助快速确定数据在哪个叶子节点上。

在创建二维表时，我们通常都会为表指定主键列，主键列上默认会创建索引，而对于 MySQL InnoDB 存储引擎来说，因为它使用的是索引组织表这种数据存储结构，所以主键上的索引就是整张表的数据，而这种索引我们也将其称之为**聚集索引**（clustered index）。很显然，一张表只能有一个聚集索引，否则表的数据岂不是要保存多次。我们自己创建的索引都是二级索引（secondary index），更常见的叫法是**非聚集索引**（non-clustered index）。通过我们自定义的非聚集索引只能定位记录的主键，在获取数据时可能需要再通过主键上的聚集索引进行查询，这种现象称为“回表”，因此通过非聚集索引检索数据通常比使用聚集索引检索数据要慢。

接下来我们通过一个简单的例子来说明索引的意义，比如我们要根据学生的姓名来查找学生，这个场景在实际开发中应该经常遇到，就跟通过商品名称查找商品是一个道理。我们可以使用 MySQL 的`explain`关键字来查看 SQL 的执行计划（数据库执行 SQL 语句的具体步骤）。

```SQL
explain select * from tb_student where stuname='林震南'\G
```

```
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_student
   partitions: NULL
         type: ALL
possible_keys: NULL
          key: NULL
      key_len: NULL
          ref: NULL
         rows: 11
     filtered: 10.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

在上面的 SQL 执行计划中，有几项值得我们关注：

1. `select_type`：查询的类型。
    - `SIMPLE`：简单 SELECT，不需要使用 UNION 操作或子查询。
    - `PRIMARY`：如果查询包含子查询，最外层的 SELECT 被标记为 PRIMARY。
    - `UNION`：UNION 操作中第二个或后面的 SELECT 语句。
    - `SUBQUERY`：子查询中的第一个 SELECT。
    - `DERIVED`：派生表的 SELECT 子查询。
2. `table`：查询对应的表。
3. `type`：MySQL 在表中找到满足条件的行的方式，也称为访问类型，包括：`ALL`（全表扫描）、`index`（索引全扫描，只遍历索引树）、`range`（索引范围扫描）、`ref`（非唯一索引扫描）、`eq_ref`（唯一索引扫描）、`const` / `system`（常量级查询）、`NULL`（不需要访问表或索引）。在所有的访问类型中，很显然 ALL 是性能最差的，它代表的全表扫描是指要扫描表中的每一行才能找到匹配的行。
4. `possible_keys`：MySQL 可以选择的索引，但是**有可能不会使用**。
5. `key`：MySQL 真正使用的索引，如果为`NULL`就表示没有使用索引。
6. `key_len`：使用的索引的长度，在不影响查询的情况下肯定是长度越短越好。
7. `rows`：执行查询需要扫描的行数，这是一个**预估值**。
8. `extra`：关于查询额外的信息。
    - `Using filesort`：MySQL 无法利用索引完成排序操作。
    - `Using index`：只使用索引的信息而不需要进一步查表来获取更多的信息。
    - `Using temporary`：MySQL 需要使用临时表来存储结果集，常用于分组和排序。
    - `Impossible where`：`where`子句会导致没有符合条件的行。
    - `Distinct`：MySQL 发现第一个匹配行后，停止为当前的行组合搜索更多的行。
    - `Using where`：查询的列未被索引覆盖，筛选条件并不是索引的前导列。

从上面的执行计划可以看出，当我们通过学生名字查询学生时实际上是进行了全表扫描，不言而喻这个查询性能肯定是非常糟糕的，尤其是在表中的行很多的时候。如果我们需要经常通过学生姓名来查询学生，那么就应该在学生姓名对应的列上创建索引，通过索引来加速查询。

```SQL
create index idx_student_name on tb_student(stuname);
```

再次查看刚才的 SQL 对应的执行计划。

```SQL
explain select * from tb_student where stuname='林震南'\G
```

```
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_student
   partitions: NULL
         type: ref
possible_keys: idx_student_name
          key: idx_student_name
      key_len: 62
          ref: const
         rows: 1
     filtered: 100.00
        Extra: NULL
1 row in set, 1 warning (0.00 sec)
```

可以注意到，在对学生姓名创建索引后，刚才的查询已经不是全表扫描而是基于索引的查询，而且扫描的行只有唯一的一行，这显然大大的提升了查询的性能。MySQL 中还允许创建前缀索引，即对索引字段的前N个字符创建索引，这样的话可以减少索引占用的空间（但节省了空间很有可能会浪费时间，**时间和空间是不可调和的矛盾**），如下所示。

```SQL
create index idx_student_name_1 on tb_student(stuname(1));
```

上面的索引相当于是根据学生姓名的第一个字来创建的索引，我们再看看 SQL 执行计划。

```SQL
explain select * from tb_student where stuname='林震南'\G
```

```
*************************** 1. row ***************************
           id: 1
  select_type: SIMPLE
        table: tb_student
   partitions: NULL
         type: ref
possible_keys: idx_student_name
          key: idx_student_name
      key_len: 5
          ref: const
         rows: 2
     filtered: 100.00
        Extra: Using where
1 row in set, 1 warning (0.00 sec)
```

不知道大家是否注意到，这一次扫描的行变成了2行，因为学生表中有两个姓“林”的学生，我们只用姓名的第一个字作为索引的话，在查询时通过索引就会找到这两行。

如果要删除索引，可以使用下面的SQL。

```SQL
alter table tb_student drop index idx_student_name;
```

或者

```SQL
drop index idx_student_name on tb_student;
```

在创建索引时，我们还可以使用复合索引、函数索引（MySQL 5.7 开始支持），用好复合索引实现**索引覆盖**可以减少不必要的排序和回表操作，这样就会让查询的性能成倍的提升，有兴趣的读者可以自行研究。

我们简单的为大家总结一下索引的设计原则：

1. **最适合**索引的列是出现在**WHERE子句**和连接子句中的列。
2. 索引列的基数越大（取值多、重复值少），索引的效果就越好。
3. 使用**前缀索引**可以减少索引占用的空间，内存中可以缓存更多的索引。
4. **索引不是越多越好**，虽然索引加速了读操作（查询），但是写操作（增、删、改）都会变得更慢，因为数据的变化会导致索引的更新，就如同书籍章节的增删需要更新目录一样。
5. 使用 InnoDB 存储引擎时，表的普通索引都会保存主键的值，所以**主键要尽可能选择较短的数据类型**，这样可以有效的减少索引占用的空间，提升索引的缓存效果。

最后，还有一点需要说明，InnoDB 使用的 B-tree 索引，数值类型的列除了等值判断时索引会生效之外，使用`>`、`<`、`>=`、`<=`、`BETWEEN...AND... `、`<>`时，索引仍然生效；对于字符串类型的列，如果使用不以通配符开头的模糊查询，索引也是起作用的，但是其他的情况会导致索引失效，这就意味着很有可能会做全表查询。

### 视图

视图是关系型数据库中将一组查询指令构成的结果集组合成可查询的数据表的对象。简单的说，视图就是虚拟的表，但与数据表不同的是，数据表是一种实体结构，而视图是一种虚拟结构，你也可以将视图理解为保存在数据库中被赋予名字的 SQL 语句。

使用视图可以获得以下好处：

1. 可以将实体数据表隐藏起来，让外部程序无法得知实际的数据结构，让访问者可以使用表的组成部分而不是整个表，降低数据库被攻击的风险。
2. 在大多数的情况下视图是只读的（更新视图的操作通常都有诸多的限制），外部程序无法直接透过视图修改数据。
3. 重用 SQL 语句，将高度复杂的查询包装在视图表中，直接访问该视图即可取出需要的数据；也可以将视图视为数据表进行连接查询。
4. 视图可以返回与实体数据表不同格式的数据，在创建视图的时候可以对数据进行格式化处理。

创建视图。

```SQL
-- 创建视图
create view `vw_avg_score` 
as 
    select `stu_id`, round(avg(`score`), 1) as `avg_score` 
    from `tb_record` group by `stu_id`;

-- 基于已有的视图创建视图
create view `vw_student_score` 
as 
    select `stu_name`, `avg_score` 
    from `tb_student` natural join `vw_avg_score`;
```

> **提示**：因为视图不包含数据，所以每次使用视图时，都必须执行查询以获得数据，如果你使用了连接查询、嵌套查询创建了较为复杂的视图，你可能会发现查询性能下降得很厉害。因此，在使用复杂的视图前，应该进行测试以确保其性能能够满足应用的需求。

使用视图。

```SQL
select * from `vw_student_score` order by `avg_score` desc;
```

```
+--------------+----------+
| stuname      | avgscore |
+--------------+----------+
| 杨过         |     95.6 |
| 任我行       |     53.5 |
| 王语嫣       |     84.3 |
| 纪嫣然       |     73.8 |
| 岳不群       |     78.0 |
| 东方不败     |     88.0 |
| 项少龙       |     92.0 |
+--------------+----------+
```

既然视图是一张虚拟的表，那么视图的中的数据可以更新吗？视图的可更新性要视具体情况而定，以下类型的视图是不能更新的：

1. 使用了聚合函数（`SUM`、`MIN`、`MAX`、`AVG`、`COUNT`等）、`DISTINCT`、`GROUP BY`、`HAVING`、`UNION`或者`UNION ALL`的视图。
2. `SELECT`中包含了子查询的视图。
3. `FROM`子句中包含了一个不能更新的视图的视图。
4. `WHERE`子句的子查询引用了`FROM`子句中的表的视图。

删除视图。

```SQL
drop view vw_student_score;
```

> **说明**：如果希望更新视图，可以先用上面的命令删除视图，也可以通过`create or replace view`来更新视图。

视图的规则和限制。

1. 视图可以嵌套，可以利用从其他视图中检索的数据来构造一个新的视图。视图也可以和表一起使用。
2. 创建视图时可以使用`order by`子句，但如果从视图中检索数据时也使用了`order by`，那么该视图中原先的`order by`会被覆盖。
3. 视图无法使用索引，也不会激发触发器（实际开发中因为性能等各方面的考虑，通常不建议使用触发器，所以我们也不对这个概念进行介绍）的执行。

### 函数

MySQL 中的函数跟 Python 中的函数太多的差异，因为函数都是用来封装功能上相对独立且会被重复使用的代码的。如果非要找出一些差别来，那么 MySQL 中的函数是可以执行 SQL 语句的。下面的例子，我们通过自定义函数实现了截断超长字符串的功能。

```SQL
delimiter $$

create function truncate_string(
    content varchar(10000),
    max_length int unsigned
) returns varchar(10000) no sql
begin
    declare result varchar(10000) default content;
    if char_length(content) > max_length then
        set result = left(content, max_length);
        set result = concat(result, '……');
    end if;
    return result;
end $$

delimiter ;
```

> **说明1**：函数声明后面的`no sql`是声明函数体并没有使用 SQL 语句；如果函数体中需要通过 SQL 读取数据，需要声明为`reads sql data`。
>
> **说明2**：定义函数前后的`delimiter`命令是为了修改定界符，因为函数体中的语句都是用`;`表示结束，如果不重新定义定界符，那么遇到的`;`的时候代码就会被截断执行，显然这不是我们想要的效果。

在查询中调用自定义函数。

```SQL
select truncate_string('和我在成都的街头走一走，直到所有的灯都熄灭了也不停留', 10) as short_string;
```

```
+--------------------------------------+
| short_string                         |
+--------------------------------------+
| 和我在成都的街头走一……                 |
+--------------------------------------+
```

### 过程

过程（又称存储过程）是事先编译好存储在数据库中的一组 SQL 的集合，调用过程可以简化应用程序开发人员的工作，减少与数据库服务器之间的通信，对于提升数据操作的性能也是有帮助的。其实迄今为止，我们使用的 SQL 语句都是针对一个或多个表的单条语句，但在实际开发中经常会遇到某个操作需要多条 SQL 语句才能完成的情况。例如，电商网站在受理用户订单时，需要做以下一系列的处理。 

1. 通过查询来核对库存中是否有对应的物品以及库存是否充足。
2. 如果库存有物品，需要锁定库存以确保这些物品不再卖给别人， 并且要减少可用的物品数量以反映正确的库存量。
3. 如果库存不足，可能需要进一步与供应商进行交互或者至少产生一条系统提示消息。 
4. 不管受理订单是否成功，都需要产生流水记录，而且需要给对应的用户产生一条通知信息。 

我们可以通过过程将复杂的操作封装起来，这样不仅有助于保证数据的一致性，而且将来如果业务发生了变动，只需要调整和修改过程即可。对于调用过程的用户来说，过程并没有暴露数据表的细节，而且执行过程比一条条的执行一组 SQL 要快得多。

下面的过程实现了查询某门课程的最高分、最低分和平均分。

```SQL
drop procedure if exists sp_score_stat;

delimiter $$

create procedure sp_score_stat(
	courseId int, 
	out maxScore decimal(4,1), 
	out minScore decimal(4,1),
	out avgScore decimal(4,1)
)
begin
	select max(score) into maxScore from tb_record where cou_id=courseId;
	select min(score) into minScore from tb_record where cou_id=courseId;
	select avg(score) into avgScore from tb_record where cou_id=courseId;
end $$

delimiter ;
```

> **说明**：在定义过程时，因为可能需要书写多条 SQL，而分隔这些 SQL 需要使用分号作为分隔符，如果这个时候，仍然用分号表示整段代码结束，那么定义过程的 SQL 就会出现错误，所以上面我们用`delimiter $$`将整段代码结束的标记定义为`$$`，那么代码中的分号将不再表示整段代码的结束，整段代码只会在遇到`end $$`时才会执行。在定义完过程后，通过`delimiter ;`将结束符重新改回成分号（恢复现场）。

上面定义的过程有四个参数，其中第一个参数是输入参数，代表课程的编号，后面的参数都是输出参数，因为过程不能定义返回值，只能通过输出参数将执行结果带出，定义输出参数的关键字是`out`，默认情况下参数都是输入参数。

调用过程。

```SQL
call sp_score_stat(1111, @a, @b, @c);
```

获取输出参数的值。

```SQL
select @a as 最高分, @b as 最低分, @c as 平均分;
```

删除过程。

```SQL
drop procedure sp_score_stat;
```

在过程中，我们可以定义变量、条件，可以使用分支和循环语句，可以通过游标操作查询结果，还可以使用事件调度器，这些内容我们暂时不在此处进行介绍。虽然我们说了很多过程的好处，但是在实际开发中，如果频繁的使用过程并将大量复杂的运算放到过程中，会给据库服务器造成巨大的压力，而数据库往往都是性能瓶颈所在，使用过程无疑是雪上加霜的操作。所以，对于互联网产品开发，我们一般建议让数据库只做好存储，复杂的运算和处理交给应用服务器上的程序去完成，如果应用服务器变得不堪重负了，我们可以比较容易的部署多台应用服务器来分摊这些压力。

如果大家对上面讲到的视图、函数、过程包括我们没有讲到的触发器这些知识有兴趣，建议大家阅读 MySQL 的入门读物[《MySQL必知必会》](https://item.jd.com/12818982.html)进行一般性了解即可，因为这些知识点在大家将来的工作中未必用得上，学了也可能仅仅是为了应付面试而已。

### MySQL 新特性

#### JSON类型

很多开发者在使用关系型数据库做数据持久化的时候，常常感到结构化的存储缺乏灵活性，因为必须事先设计好所有的列以及对应的数据类型。在业务发展和变化的过程中，如果需要修改表结构，这绝对是比较麻烦和难受的事情。从 MySQL 5.7 版本开始，MySQL引入了对 JSON 数据类型的支持（MySQL 8.0 解决了 JSON 的日志性能瓶颈问题），用好 JSON 类型，其实就是打破了关系型数据库和非关系型数据库之间的界限，为数据持久化操作带来了更多的便捷。

JSON 类型主要分为 JSON 对象和 JSON数组两种，如下所示。

1. JSON 对象

```JSON
{"name": "骆昊", "tel": "13122335566", "QQ": "957658"}
```

2. JSON 数组

```JSON
[1, 2, 3]
```

```JSON
[{"name": "骆昊", "tel": "13122335566"}, {"name": "王大锤", "QQ": "123456"}]
```

哪些地方需要用到JSON类型呢？举一个简单的例子，现在很多产品的用户登录都支持多种方式，例如手机号、微信、QQ、新浪微博等，但是一般情况下我们又不会要求用户提供所有的这些信息，那么用传统的设计方式，就需要设计多个列来对应多种登录方式，可能还需要允许这些列存在空值，这显然不是很好的选择；另一方面，如果产品又增加了一种登录方式，那么就必然要修改之前的表结构，这就更让人痛苦了。但是，有了 JSON 类型，刚才的问题就迎刃而解了，我们可以做出如下所示的设计。

```SQL
create table `tb_test`
(
`user_id` bigint unsigned,
`login_info` json,
primary key (`user_id`)
) engine=innodb;

insert into `tb_test` values 
    (1, '{"tel": "13122335566", "QQ": "654321", "wechat": "jackfrued"}'),
    (2, '{"tel": "13599876543", "weibo": "wangdachui123"}');
```

如果要查询用户的手机和微信号，可以用如下所示的 SQL 语句。

```SQL
select 
    `user_id`,
    json_unquote(json_extract(`login_info`, '$.tel')) as 手机号,
    json_unquote(json_extract(`login_info`, '$.wechat')) as 微信 
from `tb_test`;
```

```
+---------+-------------+-----------+
| user_id | 手机号      | 微信       |
+---------+-------------+-----------+
|       1 | 13122335566 | jackfrued |
|       2 | 13599876543 | NULL      |
+---------+-------------+-----------+
```

因为支持 JSON 类型，MySQL 也提供了配套的处理 JSON 数据的函数，就像上面用到的`json_extract`和`json_unquote`。当然，上面的 SQL 还有更为便捷的写法，如下所示。

```SQL
select 
	`user_id`,
    `login_info` ->> '$.tel' as 手机号,
    `login_info` ->> '$.wechat' as 微信
from `tb_test`;
```

再举个例子，如果我们的产品要实现用户画像功能（给用户打标签），然后基于用户画像给用户推荐平台的服务或消费品之类的东西，我们也可以使用 JSON 类型来保存用户画像数据，示意代码如下所示。

创建画像标签表。

```SQL
create table `tb_tags`
(
`tag_id` int unsigned not null comment '标签ID',
`tag_name` varchar(20) not null comment '标签名',
primary key (`tag_id`)
) engine=innodb;

insert into `tb_tags` (`tag_id`, `tag_name`) 
values
    (1, '70后'),
    (2, '80后'),
    (3, '90后'),
    (4, '00后'),
    (5, '爱运动'),
    (6, '高学历'),
    (7, '小资'),
    (8, '有房'),
    (9, '有车'),
    (10, '爱看电影'),
    (11, '爱网购'),
    (12, '常点外卖');
```

为用户打标签。

```SQL
create table `tb_users_tags`
(
`user_id` bigint unsigned not null comment '用户ID',
`user_tags` json not null comment '用户标签'
) engine=innodb;

insert into `tb_users_tags` values 
    (1, '[2, 6, 8, 10]'),
    (2, '[3, 10, 12]'),
    (3, '[3, 8, 9, 11]');
```

接下来，我们通过一组查询来了解 JSON 类型的巧妙之处。

1. 查询爱看电影（有`10`这个标签）的用户ID。

    ```SQL
    select * from `tb_users` where 10 member of (user_tags->'$');
    ```

2. 查询爱看电影（有`10`这个标签）的80后（有`2`这个标签）用户ID。

    ```
    select * from `tb_users` where json_contains(user_tags->'$', '[2, 10]');

3. 查询爱看电影或80后或90后的用户ID。

    ```SQL
    select `user_id` from `tb_users_tags` where json_overlaps(user_tags->'$', '[2, 3, 10]');
    ```

> **说明**：上面的查询用到了`member of`谓词和两个 JSON 函数，`json_contains`可以检查 JSON 数组是否包含了指定的元素，而`json_overlaps`可以检查 JSON 数组是否与指定的数组有重叠部分。

#### 窗口函数

MySQL 从8.0开始支持窗口函数，大多数商业数据库和一些开源数据库早已提供了对窗口函数的支持，有的也将其称之为 OLAP（联机分析和处理）函数，听名字就知道跟统计和分析相关。为了帮助大家理解窗口函数，我们先说说窗口的概念。

窗口可以理解为记录的集合，窗口函数也就是在满足某种条件的记录集合上执行的特殊函数，对于每条记录都要在此窗口内执行函数。窗口函数和我们上面讲到的聚合函数比较容易混淆，二者的区别主要在于聚合函数是将多条记录聚合为一条记录，窗口函数是每条记录都会执行，执行后记录条数不会变。窗口函数不仅仅是几个函数，它是一套完整的语法，函数只是该语法的一部分，基本语法如下所示：

```SQL
<窗口函数> over (partition by <用于分组的列名> order by <用户排序的列名>)
```

上面语法中，窗口函数的位置可以放以下两种函数：

1. 专用窗口函数，包括：`lead`、`lag`、`first_value`、`last_value`、`rank`、`dense_rank`和`row_number`等。
2. 聚合函数，包括：`sum`、`avg`、`max`、`min`和`count`等。

下面为大家举几个使用窗口函数的简单例子，我们先用如下所示的 SQL 建库建表。

```SQL
-- 创建名为hrs的数据库并指定默认的字符集
create database `hrs` default charset utf8mb4;

-- 切换到hrs数据库
use `hrs`;

-- 创建部门表
create table `tb_dept`
(
`dno` int not null comment '编号',
`dname` varchar(10) not null comment '名称',
`dloc` varchar(20) not null comment '所在地',
primary key (`dno`)
);

-- 插入4个部门
insert into `tb_dept` values 
    (10, '会计部', '北京'),
    (20, '研发部', '成都'),
    (30, '销售部', '重庆'),
    (40, '运维部', '深圳');

-- 创建员工表
create table `tb_emp`
(
`eno` int not null comment '员工编号',
`ename` varchar(20) not null comment '员工姓名',
`job` varchar(20) not null comment '员工职位',
`mgr` int comment '主管编号',
`sal` int not null comment '员工月薪',
`comm` int comment '每月补贴',
`dno` int not null comment '所在部门编号',
primary key (`eno`),
constraint `fk_emp_mgr` foreign key (`mgr`) references tb_emp (`eno`),
constraint `fk_emp_dno` foreign key (`dno`) references tb_dept (`dno`)
);

-- 插入14个员工
insert into `tb_emp` values 
    (7800, '张三丰', '总裁', null, 9000, 1200, 20),
    (2056, '乔峰', '分析师', 7800, 5000, 1500, 20),
    (3088, '李莫愁', '设计师', 2056, 3500, 800, 20),
    (3211, '张无忌', '程序员', 2056, 3200, null, 20),
    (3233, '丘处机', '程序员', 2056, 3400, null, 20),
    (3251, '张翠山', '程序员', 2056, 4000, null, 20),
    (5566, '宋远桥', '会计师', 7800, 4000, 1000, 10),
    (5234, '郭靖', '出纳', 5566, 2000, null, 10),
    (3344, '黄蓉', '销售主管', 7800, 3000, 800, 30),
    (1359, '胡一刀', '销售员', 3344, 1800, 200, 30),
    (4466, '苗人凤', '销售员', 3344, 2500, null, 30),
    (3244, '欧阳锋', '程序员', 3088, 3200, null, 20),
    (3577, '杨过', '会计', 5566, 2200, null, 10),
    (3588, '朱九真', '会计', 5566, 2500, null, 10);
```

例子1：查询按月薪从高到低排在第4到第6名的员工的姓名和月薪。

```SQL
select * from (
	select 
		`ename`, `sal`,
		row_number() over (order by `sal` desc) as `rank`
	from `tb_emp`
) `temp` where `rank` between 4 and 6;
```

> **说明**：上面使用的函数`row_number()`可以为每条记录生成一个行号，在实际工作中可以根据需要将其替换为`rank()`或`dense_rank()`函数，三者的区别可以参考官方文档或阅读[《通俗易懂的学会：SQL窗口函数》](https://zhuanlan.zhihu.com/p/92654574)进行了解。在MySQL 8以前的版本，我们可以通过下面的方式来完成类似的操作。
>
> ```SQL
> select `rank`, `ename`, `sal` from (
>        select @a:=@a+1 as `rank`, `ename`, `sal` 
>        from `tb_emp`, (select @a:=0) as t1 order by `sal` desc
> ) as `temp` where `rank` between 4 and 6;
> ```

例子2：查询每个部门月薪最高的两名的员工的姓名和部门名称。

```SQL
select `ename`, `sal`, `dname` 
from (
    select 
        `ename`, `sal`, `dno`,
        rank() over (partition by `dno` order by `sal` desc) as `rank`
    from `tb_emp`
) as `temp` natural join `tb_dept` where `rank`<=2;
```

> 说明：在MySQL 8以前的版本，我们可以通过下面的方式来完成类似的操作。
>
> ```SQL
> select `ename`, `sal`, `dname` from `tb_emp` as `t1` 
natural join `tb_dept` 
where (
        select count(*) from `tb_emp` as `t2` 
        where `t1`.`dno`=`t2`.`dno` and `t2`.`sal`>`t1`.`sal` 
)<2 order by `dno` asc, `sal` desc;
> ```

###  其他内容

#### 范式理论

范式理论是设计关系型数据库中二维表的指导思想。

1. 第一范式：数据表的每个列的值域都是由原子值组成的，不能够再分割。
2. 第二范式：数据表里的所有数据都要和该数据表的键（主键与候选键）有完全依赖关系。
3. 第三范式：所有非键属性都只和候选键有相关性，也就是说非键属性之间应该是独立无关的。

> **说明**：实际工作中，出于效率的考虑，我们在设计表时很有可能做出反范式设计，即故意降低方式级别，增加冗余数据来获得更好的操作性能。

#### 数据完整性

1. 实体完整性 - 每个实体都是独一无二的

   - 主键（`primary key`） / 唯一约束（`unique`）
2. 引用完整性（参照完整性）- 关系中不允许引用不存在的实体

   - 外键（`foreign key`）
3. 域（domain）完整性 - 数据是有效的
   - 数据类型及长度

   - 非空约束（`not null`）

   - 默认值约束（`default`）

   - 检查约束（`check`）

     > **说明**：在 MySQL 8.x 以前，检查约束并不起作用。

#### 数据一致性

1. 事务：一系列对数据库进行读/写的操作，这些操作要么全都成功，要么全都失败。

2. 事务的 ACID 特性
   - 原子性：事务作为一个整体被执行，包含在其中的对数据库的操作要么全部被执行，要么都不执行
   - 一致性：事务应确保数据库的状态从一个一致状态转变为另一个一致状态
   - 隔离性：多个事务并发执行时，一个事务的执行不应影响其他事务的执行
   - 持久性：已被提交的事务对数据库的修改应该永久保存在数据库中

3. MySQL 中的事务操作

   - 开启事务环境

     ```SQL
     start transaction
     ```

   - 提交事务

     ```SQL
     commit
     ```

   - 回滚事务

     ```SQL
     rollback
     ```

4. 查看事务隔离级别

    ```SQL
    show variables like 'transaction_isolation';
    ```

    ```
    +-----------------------+-----------------+
    | Variable_name         | Value           |
    +-----------------------+-----------------+
    | transaction_isolation | REPEATABLE-READ |
    +-----------------------+-----------------+
    ```

    可以看出，MySQL 默认的事务隔离级别是`REPEATABLE-READ`。

5. 修改（当前会话）事务隔离级别

    ```SQL
    set session transaction isolation level read committed;
    ```

    重新查看事务隔离级别，结果如下所示。

    ```
    +-----------------------+----------------+
    | Variable_name         | Value          |
    +-----------------------+----------------+
    | transaction_isolation | READ-COMMITTED |
    +-----------------------+----------------+
    ```

关系型数据库的事务是一个很大的话题，因为当存在多个并发事务访问数据时，就有可能出现三类读数据的问题（脏读、不可重复读、幻读）和两类更新数据的问题（第一类丢失更新、第二类丢失更新）。想了解这五类问题的，可以阅读我发布在 CSDN 网站上的[《Java面试题全集（上）》](https://blog.csdn.net/jackfrued/article/details/44921941)一文的第80题。为了避免这些问题，关系型数据库底层是有对应的锁机制的，按锁定对象不同可以分为表级锁和行级锁，按并发事务锁定关系可以分为共享锁和独占锁。然而直接使用锁是非常麻烦的，为此数据库为用户提供了自动锁机制，只要用户指定适当的事务隔离级别，数据库就会通过分析 SQL 语句，然后为事务访问的资源加上合适的锁。此外，数据库还会维护这些锁通过各种手段提高系统的性能，这些对用户来说都是透明的。想了解 MySQL 事务和锁的细节知识，推荐大家阅读进阶读物[《高性能MySQL》](https://item.jd.com/11220393.html)，这也是数据库方面的经典书籍。

ANSI/ISO SQL 92标准定义了4个等级的事务隔离级别，如下表所示。需要说明的是，事务隔离级别和数据访问的并发性是对立的，事务隔离级别越高并发性就越差。所以要根据具体的应用来确定到底使用哪种事务隔离级别，这个地方没有万能的原则。

<img src="https://gitee.com/jackfrued/mypic/raw/master/20211121225327.png" style="zoom:50%;">

### 总结

关于 MySQL 的知识肯定远远不止上面列出的这些，比如 MySQL 性能调优、MySQL 运维相关工具、MySQL 数据的备份和恢复、监控 MySQL 服务、部署高可用架构等，这一系列的问题在这里都没有办法逐一展开来讨论，那就留到有需要的时候再进行讲解吧，各位读者也可以自行探索。
