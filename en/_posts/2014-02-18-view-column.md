---
layout: post
title:  "Viewing table schema &ndash; the hard way"
captitle:  "Viewing Table Schema &ndash; the Hard Way"
date:   2014-02-18 12:53:45
categories: MSSQL
tags: sql
lang: en
---

#Data Types#

Let's create a database which will serve as a 'system' database for the server. Create a table there, and fill it with everything that `exec sys.sp_datatype_info` has returned.

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

By doing this, we have stored all datatypes of the MS SQL databases. We shall make use of them pretty soon.

Having data types in a separate table brings two major advantages:

  * Tables return data faster than procedures and views
  * Unlike a procedure, you can join a table (in fact, you can, but OPENROWSET is slow, hard to remember and difficult to maintain: it requires a new connection and so on)


#Viewing Columns#

Once we try to execute the following:

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

we get a result-set with all table definitions within a database.

<img width="100%" alt="View columns" src="/img/2014/view-column.png" style="cursor:pointer" onclick="window.open('/img/2014/view-column.png','_blank');return;"></img>

#There's a catch#

As you can see, there is an `object_id` field in the result set. It is the unique if of a table `SELECT OBJECT_ID('AdventureWorks2008.SalesLT.Customer')` *within* the database. So, if you ever decide (which I will do) to put all databases into a single table `SYSDB.._table_columns`, bear in mind that `object_id` does not identify the table within a server. You also need to filter the `db` column.

Also, column type `sysname` and user-defined types may mess things up, so it is worth filtering the `sysname` results in the query.

#Why not `exec sp_help 'AdventureWorks2008.SalesLT.Customer'`?#

  1. Because it is nice to read, but hard to get use of.
  1. Because it will soon become a static table, which is faster than a procedure.
  1. Because it does not give all the needed information for my purposes. The hard way it is.

What Next?
==========

[This][copyrow].

[copyrow]: {{ site.url }}/{{ page.lang }}/2014/04/03/copy-row.html

