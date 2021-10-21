## Mysql中Exists和In的使用

### Exists的使用
> exists对外表用loop逐条查询，每次查询都会查看exists的条件语句，当 exists里的条件语句能够返回记录行时(无论记录行是的多少，只要能返回)，条件就为真，返回当前loop到的这条记录，反之如果exists里的条 件语句不能返回记录行，则当前loop到的这条记录被丢弃，exists的条件就像一个bool条件，当能返回结果集则为true，不能返回结果集则为 false

如下：

	select * from user where exists (select 1);

对user表的记录逐条取出，由于子条件中的`select 1`永远能返回记录行，那么user表的所有记录都将被加入结果集，所以与 `select * from user` 是一样的

又如下

	select * from user where exists (select * from user where userId = 0);

可以知道对user表进行loop时，检查条件语句`(select * from user where userId = 0)`,由于userId永远不为0，所以条件语句永远返回空集，条件永远为false，那么user表的所有记录都将被丢弃

`not exists`与`exists`相反，也就是当`exists`条件有结果集返回时，loop到的记录将被丢弃，否则将loop到的记录加入结果集

> 总的来说，如果A表有n条记录，那么exists查询就是将这n条记录逐条取出，然后判断n遍exists条件 

 

 
### In的使用
> in查询相当于多个or条件的叠加，这个比较好理解，

比如下面的查询

	select * from user where userId in (1, 2, 3);

等效于

	select * from user where userId = 1 or userId = 2 or userId = 3;

`not in`与`in`相反，如下

	select * from user where userId not in (1, 2, 3);

等效于

	select * from user where userId != 1 and userId != 2 and userId != 3;

> 总的来说，`in`查询就是先将子查询条件的记录全都查出来，假设结果集为B，共有m条记录，然后在将子查询条件的结果集分解成m个，再进行m次查询

 

**值得一提的是** 

`in`查询的子条件返回结果必须只有一个字段

例如

	select * from user where userId in (select id from B);

而不能是

	select * from user where userId in (select id, age from B);

而`exists`就没有这个限制

 
### Exists与In的比较


#### Exists解析
下面来考虑exists和in的性能

考虑如下SQL语句

	1. select * from A where exists (select * from B where B.id = A.id);

	2. select * from A where A.id in (select id from B);

 

`查询1.`可以转化以下伪代码，便于理解

	for ($i = 0; $i < count(A); $i++) {

	　　$a = get_record(A, $i); #从A表逐条获取记录

	　　if (B.id = $a[id]) #如果子条件成立

	　　　　$result[] = $a;

	}

	return $result;

__大概就是这么个意思，其实可以看到,`查询1`主要是用到了B表的索引，A表如何对查询的效率影响应该不大__

 
#### In解析
假设B表的所有id为1,2,3,`查询2`可以转换为

	select * from A where A.id = 1 or A.id = 2 or A.id = 3;

__这个好理解了，这里主要是用到了A的索引，B表如何对查询影响不大__

 

**下面再看`not exists` 和 `not in`**

	1. select * from A where not exists (select * from B where B.id = A.id);

	2. select * from A where A.id not in (select id from B);

看`查询1，`还是和上面一样，用了B的索引

而对于`查询2，`可以转化成如下语句

	select * from A where A.id != 1 and A.id != 2 and A.id != 3;

可以知道`not in`是个范围查询，这种`!=`的范围查询无法使用任何索引,等于说A表的每条记录，都要在B表里遍历一次，查看B表里是否存在这条记录

> 故`not exists`比`not in`效率高

 
### 总结
> mysql中的`in`语句是把外表和内表作`hash` 连接，而`exists`语句是对外表作`loop循环`，每次`loop循环`再对内表进行查询。一直大家都认为exists比in语句的效率要高，这种说法其实是不准确的。这个是要区分环境的。
 

- 如果查询的两个表大小相当，那么用`in`和`exists`差别不大。 
- 如果两个表中一个较小，一个是大表，则子查询表大的用`exists`，子查询表小的用`in`

例如：表A（小表），表B（大表）
 
	
	select * from A where cc in (select cc from B) 效率低，用到了A表上cc列的索引；
	 
	select * from A where exists(select cc from B where cc=A.cc) 效率高，用到了B表上cc列的索引。
	

相反的
 
	select * from B where cc in (select cc from A) 效率高，用到了B表上cc列的索引；
	 
	select * from B where exists(select cc from A where cc=B.cc) 效率低，用到了A表上cc列的索引。
 
 
`not in` 和`not exists` 如果查询语句使用了`not in` 那么内外表都进行全表扫描，没有用到索引；而`not extsts` 的子查询依然能用到表上的索引。所以无论那个表大，用`not exists`都比`not in`要快。 

**in 与 = 的区别** 

	select name from student where name in ('zhang','wang','li','zhao'); 
与
	
	select name from student where name='zhang' or name='li' or name='wang' or name='zhao' 

的结果是相同的。