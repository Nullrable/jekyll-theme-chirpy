---
title: INSERT IGNORE 与 INSERT INTO的区别
author: nhsoft.lsd
categories: [MySQL]
tags: [MySQL,数据库]
pin: false
---

例
insert ignore表示，如果中已经存在相同的记录，则忽略当前新数据；
insert ignore into table(name) select name from table2
例
INSERT INTO有无数据都插入，如果主键则不插入
1.insert语句一次可以插入多组值，每组值用一对圆括号括起来，用逗号分隔，如下：
```
insert into `news`(title,body,time) values('www.111cn.net','body 1',now()),('title 2','body 2',now());
```

下面通过代码说明之间的区别，如下：

`create table testtb(id int not null primary key,name varchar(50),age int);`

`insert into testtb(id,name,age)values(1,"www.111Cn.net",13);`

`select * from testtb;`

`insert ignore into testtb(id,name,age)values(1,"aa",13);`

`select * from testtb;//仍是1，“bb”,13，#因为id是主键，出现主键重复但使用了ignore则错误被忽略`

