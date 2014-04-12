---
layout: post
title:  "Угадываем схему по запросу"
captitle:  "Угадываем схему по запросу"
date: 2014-03-12 16:53:10
categories: MSSQL
tags: sql clr
lang: ru
---

Это решение возникло внезапно, во время вынужденного переезда с MS SQL 2008R2 на MS SQL 2012.

Что имеем
=====================

Когда система НСИ построена на реальных таблицах (которые пользователь может физически изменять с помощью предоставляемых ему системных механизмов), поиск по классификатору и использование разных представлений для данных в каждой таблице требуют применение динамических запросов. Это неизбежно. Но при попытке `INSERT INTO #table FROM OPENQUERY(LINKED_SERVER,'SELECT <...>')` сервер MS SQL 2012 может нам намекнуть, что он хотел бы более явно знать, что мы хотим вернуть. Например, `WITH RESULT SETS (раз-два)`. И это сбивает с толку, поскольку этот запрос динамический и служит для извлечения данных из любой таблицы, какую только пожелает пользователь &mdash; мы никогда не знаем, какой набор столбцов вернётся. Перестраивание парадигмы всей системы не обсуждалось, так что решением стало создание статических таблиц с тем же набором столбцов, а потом работа с ними.

Угадывание
=====================

Это CLR-функция, которая возвращает угаданную схему всякий раз, когда мы предоставляем ей запрос.

{% highlight c# %}
public class tableDefinitionClass
{
	[SqlFunction(DataAccess= DataAccessKind.Read)]
	public static SqlString getSchemaByQuery(SqlString query)
	{
		try
		{
			string schema="";
			string sql=query.ToString();

			SqlConnection conn = new SqlConnection("context connection=true");
			conn.Open();
			SqlCommand command = null;
			SqlDataReader reader = null;
			while(sql.IndexOf("'{") > 0)
				sql = sql.Substring(0,sql.IndexOf("'{") + 1) + sql.Substring(sql.IndexOf("}'") + 1, sql.Length - sql.IndexOf("}'"));
			while(sql.IndexOf("{") > 0)
				sql = sql.Substring(0,sql.IndexOf("{"))
					+ (sql.Substring(sql.IndexOf("{"),sql.IndexOf("}")-sql.IndexOf("{")).Contains("|") ? " 1=0 " : "''")
					+ sql.Substring(sql.IndexOf("}") + 1, sql.Length - sql.IndexOf("}") - 1);
			try
			{
				command = new SqlCommand(String.Format(sql),conn);
				reader = command.ExecuteReader();
			}
			catch
			{
				conn.Close();
				return null;
			}

			DataTable td = reader.GetSchemaTable();
			foreach (DataRow myField in td.Rows)
			{
				string ColumnName="";
				string ColumnSize="";
				string NumericPrecision="";
				string NumericScale="";
				string DataTypeName="";
			    foreach (DataColumn myProperty in td.Columns)
			    {
					switch (myProperty.ColumnName.ToString())
					{
							case "ColumnName":			{ColumnName = myField[myProperty].ToString();break;}
							case "ColumnSize":			{ColumnSize = myField[myProperty].ToString();break;}
							case "NumericPrecision":	{NumericPrecision = myField[myProperty].ToString();break;}
							case "NumericScale":		{NumericScale = myField[myProperty].ToString();break;}
							case "DataTypeName":		{DataTypeName = myField[myProperty].ToString();break;}
					}
			    }
				schema += ColumnName + " " + DataTypeName;
				if(	DataTypeName == "binary"
				   || DataTypeName == "char"
				   || DataTypeName == "nchar"
				   || DataTypeName == "nvarchar"
				   || DataTypeName == "varchar"
				   || DataTypeName == "varbinary"
				  )
					schema += "(" + (ColumnSize=="2147483647"?"max":ColumnSize) + ")";
				else if(DataTypeName == "datetime2"
				   || DataTypeName == "datetimeoffset"
				   || DataTypeName == "time"
				  )
					schema += "(" + NumericScale + ")";
				else if(DataTypeName == "decimal"
				   || DataTypeName == "numeric"
				  )
					schema += "(" + NumericPrecision + "," + NumericScale + ")";
				schema += ",";
			}
			if(schema.Length > 1)
				schema = schema.Substring(0, schema.Length - 1);
			return schema;
		}
		catch
		{
			return null;
		}
	}
}
{% endhighlight %}

Запрос
=====================

```sql
SELECT SYSDB.dbo._getSchemaByQuery('SELECT * FROM AdventureWorks2008.SalesLT.Customer')
```

Результат
=====================

<center><img alt="View columns" src="/img/2014/getschemabyquery.png" href="/img/2014/getschemabyquery.png" style="cursor:pointer" onclick="window.open('/img/2014/getschemabyquery.png','_blank');return;" /></center>

Примечания
=====================

1. Вопрос к нам: какое соединение использовать нашей функции для выполнения запроса? Передать в саму функцию какой-то ConnectionString?

    Ответ нам: `SqlConnection conn = new SqlConnection("context connection=true");`

1. Неважно, в контексте какой базы данных мы работаем &mdash; он будет утрачен. Поэтому, попросив

    ```sql
    USE AdventureWorks2008
    GO
    SELECT SYSDB.dbo._guessSchemaByQuery('SELECT * FROM Address')
    ```
получим в ответ `NULL`. Если мы хотим, чтобы этот запрос заработал, нужно указывать БД и схему явно:

    ```sql
    SELECT SYSDB.dbo._guessSchemaByQuery('SELECT * FROM AdventureWorks2008.SalesLT.Address')
    ```

1. Функция, которая подключается к серверу, должна иметь флажок `[SqlFunction(DataAccess=DataAccessKind.Read)]`. В противной случае, могут быть проблемы с подключением из-за безопасности.

Попробовать
=====================

Если вы хотите применить эту функцию, проследуйте [сюда][dll].

На эту тему
=====================

1. [sp\_describe\_first\_result\_set](http://technet.microsoft.com/en-us/library/ff878602.aspx)
1. [SQL Server 2012 - WITH RESULT SETS](http://www.allaboutmssql.com/2012/10/sql-server-2012-with-result-sets.html)
1. [MSDN How to use "WITH RESULT SETS" clause in SQL 2012 for dynamic column names.](http://social.msdn.microsoft.com/Forums/en-US/4e98380a-df92-49a0-97d8-4908307573c4/how-to-use-with-result-sets-clause-in-sql-2012-for-dynamic-column-names?forum=transactsql)

[dll]:https://github.com/atru/database-dll
