---
layout: post
title:  "Просмотр блокирующих соединений в MS SQL"
captitle:  "Просмотр блокирующих соединений в MS SQL"
date:   2014-02-18 00:24:10
categories: MSSQL
tags: cheatsheet sql
lang: ru
---

Написание объемных процедур для автоматизации производства неизбежно влечёт за собой использование транзакций, что порой вызывает головную боль: малейшая невнимательность может заблокировать базу.

Приведённый ниже фрагмент кода отображает подключенные логины, заблокированные базы, источник подключения и идентификатор сессии. Эти подключения и нужно сбрасывать. То есть, `kill 67`. Сбросили, и копаем дальше.

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

По желанию можно поиграть с `request_mode` и посмотреть на другие типы подключений (активные, спящие).
