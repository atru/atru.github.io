---
layout: post
title:  "Извлекаем схему таблицы &ndash; сложный вариант"
captitle:  "Извлекаем схему таблицы &ndash; сложный вариант"
date:   2014-02-18 12:53:45
categories: MSSQL
tags: sql
lang: ru
---

#Типы данных#

Создадим базу данных, которая будет выполнять своего рода 'системную' функцию для нашего сервера. В ней мы создадим таблицу и заполним её всем, что вернет нам `exec sys.sp_datatype_info`.

{% highlight sql %}

USE [SYSDB]
GO
SET ANSI_NULLS ON
GO
SET QUOTED_IDENTIFIER ON
GO
SET ANSI_PADDING ON
GO
CREATE TABLE [dbo].[_datatype_info](
	[TYPE_NAME] [nvarchar](128) NULL,
	[DATA_TYPE] [smallint] NULL,
	[PRECISION] [int] NULL,
	[LITERAL_PREFIX] [varchar](32) NULL,
	[LITERAL_SUFFIX] [varchar](32) NULL,
	[CREATE_PARAMS] [varchar](32) NULL,
	[NULLABLE] [smallint] NULL,
	[CASE_SENSITIVE] [smallint] NULL,
	[SEARCHABLE] [smallint] NOT NULL,
	[UNSIGNED_ATTRIBUTE] [smallint] NULL,
	[MONEY] [smallint] NOT NULL,
	[AUTO_INCREMENT] [smallint] NULL,
	[LOCAL_TYPE_NAME] [nvarchar](128) NULL,
	[MINIMUM_SCALE] [smallint] NULL,
	[MAXIMUM_SCALE] [smallint] NULL,
	[SQL_DATA_TYPE] [smallint] NULL,
	[SQL_DATETIME_SUB] [smallint] NULL,
	[NUM_PREC_RADIX] [int] NULL,
	[INTERVAL_PRECISION] [smallint] NULL,
	[USERTYPE] [smallint] NULL
) ON [PRIMARY]

GO
SET ANSI_PADDING OFF
GO

INSERT INTO SYSDB.._datatype_info
exec sys.sp_datatype_info

{% endhighlight %}

Выполнив эти действия, мы имеем все системные типы данных MS SQL, сохранённые в таблице. Вскоре мы ими воспользуемся.

Хранить эту информацию в отдельной таблице полезно по двум главным причинам:

  * Таблицы быстрее возвращают результат, нежели представления или процедуры
  * В отличие от процедуры, таблицу можно присоединять к запросу (на самом деле, можно использовать `OPENROWSET`, но он трудный для чтения и написания, непрост в администрировании: требует отдельное соединение, и пр.)

#Отображение стобцов#

Как только мы выполним следующее:

{% highlight sql %}
SELECT 'AdventureWorks2008' db, C.object_id, C.name, Tp.name AS type, C.column_id, C.is_nullable, C.is_identity,
				(CASE when ST.CREATE_PARAMS is null then '' WHEN CHARINDEX(',',ST.CREATE_PARAMS)>0 then CAST(Tp.precision AS varchar(10)) ELSE CAST(C.max_length AS varchar(10)) END) AS length, 
				(CASE WHEN CHARINDEX(',',ST.CREATE_PARAMS)>0 then CAST(C.scale AS varchar(10)) ELSE '' END) AS precision,
				(CASE WHEN C.precision > 0 THEN C.precision WHEN C.precision = 0 AND C.max_length > 0 THEN C.max_length ELSE Tp.max_length END) AS StrLen,
				ST.LITERAL_PREFIX COLLATE Cyrillic_General_CI_AS AS LITERAL_PREFIX, ST.LITERAL_SUFFIX COLLATE Cyrillic_General_CI_AS AS LITERAL_SUFFIX,
				(CASE WHEN t2.is_primary_key=1 THEN 'Y' END) AS primary_key
			FROM AdventureWorks2008.sys.columns AS C JOIN AdventureWorks2008.sys.types AS Tp ON(C.system_type_id = Tp.system_type_id) 
					JOIN SYSDB.._datatype_info AS ST ON(Tp.name = ST.TYPE_NAME)
					left join AdventureWorks2008.sys.index_columns t1 on(t1.column_id = C.column_id and t1.object_id = C.object_id)
					left JOIN AdventureWorks2008.sys.indexes t2 on(t1.object_id = t2.object_id AND t1.index_id = t2.index_id)
{% endhighlight %}

нам вернётся набор данных, перечисляющий схемы всех таблиц нашей базы.

<img width="100%" alt="View columns" src="/img/2014/view-column.png" style="cursor:pointer" onclick="window.open('/img/2014/view-column.png','_blank');return;" ></img>

#Есть уловка#

Как видно, в результирущем наборе есть поле `object_id`. Это уникальный идентификатор таблицы (`SELECT OBJECT_ID('AdventureWorks2008.SalesLT.Customer')`) *в пределах* базы данных. Так что если вы вдруг решите (а я уже решил) сохранить схемы таблиц из всех баз данных в одной таблице `SYSDB.._table_columns`, нужно помнить, что поле  `object_id` не идентифицирует таблицу в пределах сервера. Для этого нужно использовать столбец `db`.

Также, столбец с типом `sysname` и пользовательские типы (user-defined type) могут кое-что подпортить, поэтому есть смысл отбрасывать результаты с `sysname`.

#Почему просто не `exec sp_help 'AdventureWorks2008.SalesLT.Customer'`?#

  1. Потому что результат удобно читать, но неудобно извлекать (особенно второй резалт-сет).
  1. Потому что вскоре всё это станет статичной таблицей (быстрее, чем процедура).
  1. Потому что она не возвращает достаточное количество информации для моих целей. Сложный вариант, как-никак.
