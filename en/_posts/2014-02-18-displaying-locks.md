---
layout: post
title:  "Displaying locks in MS SQL"
captitle:  "Displaying Locks in MS SQL"
date:   2014-02-18 00:24:10
categories: MSSQL
tags: cheatsheet sql
lang: en
---

Writing bulk procedures for MES demands working with transactions, which is sometimes painful: a single mistake may lock the whole database.

This handy snippet of code displays connection logins, the locked database, the host and the session id. Then you kill those sessions. Like, `kill 67`. And then start digging.

{% highlight sql %}
SELECT DISTINCT request_mode,
      name AS database_name,
      session_id,
      host_name,
      login_time,
      login_name,
      reads,
      writes
	FROM sys.dm_exec_sessions
	LEFT OUTER JOIN sys.dm_tran_locks 
		ON sys.dm_exec_sessions.session_id = sys.dm_tran_locks.request_session_id
	INNER JOIN sys.databases
		ON sys.dm_tran_locks.resource_database_id = sys.databases.database_id
	WHERE resource_type <> 'DATABASE'
	AND request_mode LIKE '%X%' --EXCULSIVE LOCK
	--AND name ='YourDatabaseNameHere'
 ORDER BY name
{% endhighlight %}

You can also play with the `request_mode` and display any other type of connection.
