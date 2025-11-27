---
title: SQLServer锁基本概念
author: nhsoft.lsd
date: 2022-08-21
categories: [SQLServer]
tags: [SQLServer,数据库]
pin: false
---

1.监控结果中blocked列的值为阻塞spid的阻塞头session_id，waitresource为被阻塞的session等待的资源。当两个session相互阻塞时，即发生了死锁。

```
while 1=1
begin
select * from sys.sysprocesses where blocked<>0
waitfor delay '00:00:01'  --循环间隔时间可以自定义
end

```

```
while 1=1
Begin
SELECT
db.name DBName,
tl.request_session_id,  --被阻塞session
wt.blocking_session_id,  --阻塞头session
OBJECT_NAME(p.OBJECT_ID) BlockedObjectName,
tl.resource_type,
h1.TEXT AS RequestingText,
h2.TEXT AS BlockingText,
tl.request_mode
FROM sys.dm_tran_locks AS tl
INNER JOIN sys.databases db ON db.database_id = tl.resource_database_id
INNER JOIN sys.dm_os_waiting_tasks AS wt ON tl.lock_owner_address = wt.resource_address
INNER JOIN sys.partitions AS p ON p.hobt_id = tl.resource_associated_entity_id
INNER JOIN sys.dm_exec_connections ec1 ON ec1.session_id = tl.request_session_id
INNER JOIN sys.dm_exec_connections ec2 ON ec2.session_id = wt.blocking_session_id
CROSS APPLY sys.dm_exec_sql_text(ec1.most_recent_sql_handle) AS h1
CROSS APPLY sys.dm_exec_sql_text(ec2.most_recent_sql_handle) AS h2
waitfor delay '00:00:01' --循环间隔时间可以自定义
End
```

![weixin.png](/assets/img/nhsoft_lsd/weixin.png)

<div style="text-align: center;">公众号名称：怪味Coding</div>
<div style="text-align: center;">微信扫码关注或搜索公众号名称</div>
