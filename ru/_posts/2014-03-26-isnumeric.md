---
layout: post
title:  "Действительно ли ISNUMERIC() только о numeric?"
captitle:  "Действительно ли ISNUMERIC() только о numeric?"
date: 2014-03-26 11:38:28
categories: MSSQL
tags: sql clr
lang: ru
---

Плохой дизайн
=====================

Если вы разрабатываете движок для баз данных и предоставляете системный тип `numeric`, а также набор других  числовых типов (`decimal`, `float`, `real`, `int`, `bigint`, `smallint`), вы ожидаете, что разработчики будут пользоваться функцией `ISNUMERIC()` для проверки некоторого текста (например `-0.001`) на соответствие «числу» (numeric). Логично. Но потом вы добавляете, что типы `money` и `smallmoney` также являются «числовыми», и даже оставляете об этом заметку на MSDN.

>ISNUMERIC возвращает «1» для некоторых символов, которые не являются числами (например, плюс (+), минус (-) и такие символы валют, как знак доллара ($). [MSDN][msdn]


Проверим монстра в действии

<center><img title="Fuzzy logic" src="/img/2014/isnumeric_01.png" /></center>

Очевидно, что `ISNUMERIC('-$00,0.2,,,0')` возвращает `1`. У меня <u title="Devdrama ведь">нет слов</u>.

Давайте изобретём велосипед
=====================

Порой вам может понадобиться <strike>нормальный</strike> более предсказуемый способ проверки текста на число. Вот тогда можно прибегнуть к [CLR][clr].

```c#
public static SqlBoolean fnIsNumeric(SqlString field, string sqltype)
{
	var result = new SqlBoolean(0);  // default False
	string errorMessage = string.Empty;

	// Determine base type and any decimal precision parameters
	if(sqltype==null)
		return result;
	var type = sqltype.ToString().Trim().ToLower();
	var typePrecision = string.Empty;
	if(type.Contains("(")) {
		typePrecision = type.Substring(type.IndexOf("(") + 1).Replace(")", "").Trim();
		type = type.Substring(0, type.IndexOf("(") );
	}

	try
	{
		switch (type)
		{
			case "bigint":
				var sqlBigInt = new SqlInt64();
				sqlBigInt = field.ToSqlInt64();
				if (sqlBigInt.IsNull == false) result = true;
				break;

			case "bit":
				if(field.Value.Contains("+") || field.Value.Contains("-")) {
					result = false;
				} else {
					var sqlBit = new SqlByte();
					sqlBit = field.ToSqlByte();
					if (sqlBit.IsNull == false && (sqlBit == 0 || sqlBit == 1)) result = true;
				}
				break;

			case "decimal":
			case "numeric":
				// Support format decimal(x,0) or decimal(x,y) where true only if number fits in precision x,y
				// Precision = maximum number of digits to the left of the decimal point
				// If decimal(x,y) supplied, maximum precision = x - y
				var sqlDecimal = new SqlDecimal();
				sqlDecimal = field.ToSqlDecimal();
				if(sqlDecimal.IsNull == false)
				{
					result = true;
					if (typePrecision.Length > 0)
					{
						var parameters = typePrecision.Split(",".ToCharArray());
						if (parameters.Length > 0)
						{
							int precision = 0;
							int.TryParse(parameters[0], out precision);
							if (precision > 0)
							{
								if (parameters.Length > 1)
								{
									int scale = 0;
									int.TryParse(parameters[1], out scale);
									precision = precision - scale;
								}
								var x = " " + sqlDecimal.Value.ToString().Replace("-","") + ".";
								string decPrecisionDigitCount = x.Substring(0,x.IndexOf(".")).Trim();
								if(decPrecisionDigitCount.Length > precision) result = false;
							}
						}
					}
				}
				break;

			case "float":
				var sqlFloat = new SqlDouble();
				sqlFloat = field.ToSqlDouble();
				if (sqlFloat.IsNull == false) result = true;
				break;

			case "int":
				var sqlInt = new SqlInt32();
				sqlInt = field.ToSqlInt32();
				if (sqlInt.IsNull == false) result = true;
				break;

			case "money":
				var sqlMoney = new SqlMoney();
				sqlMoney = field.ToSqlMoney();
				if (sqlMoney.IsNull == false) result = true;
				break;

			case "real":
				var SqlSingle = new SqlSingle();
				SqlSingle = field.ToSqlSingle();
				if (SqlSingle.IsNull == false) result = true;
				break;

			case "smallint":
				var sqlSmallInt = new SqlInt16();
				sqlSmallInt = field.ToSqlInt16();
				if (sqlSmallInt.IsNull == false) result = true;
				break;

			case "smallmoney":
				var sqlSmallMoney = new SqlMoney();
				sqlSmallMoney = field.ToSqlMoney();
				if (sqlSmallMoney.IsNull == false) {
					// Ensure that it will fit in a 4-byte small money
					if (sqlSmallMoney.Value >= -214748.3648m && sqlSmallMoney.Value <= 214748.3647m) {
						result = true;
					}
				}
				break;

			case "tinyint":
				var sqlTinyInt = new SqlByte();
				sqlTinyInt = field.ToSqlByte();
				if (sqlTinyInt.IsNull == false) result = true;
				break;

			default:
				errorMessage = "Unrecognized format";
				break;
		}
	}
	catch (Exception)
	{
		if (string.IsNullOrEmpty(errorMessage) == false)
		{
			result = SqlBoolean.Null;
		}
	}
   return result;
}
```

Мы пытаемся преобразовать строку в указанный тип. Если ошибок не возникает, отвечаем «да».

Проверка
=====================

Такой скрипт

```sql
DECLARE @value table (value varchar(1000))
INSERT INTO @value SELECT '$'
INSERT INTO @value SELECT '999'
INSERT INTO @value SELECT '-999999'
INSERT INTO @value SELECT '999.999'
INSERT INTO @value SELECT '-999.999'
INSERT INTO @value SELECT '-9999999.999'
INSERT INTO @value SELECT '-0'

select value,
    SYSDB.dbo.IsNumeric2(value,'bit') as IsBoolean,
	SYSDB.dbo.IsNumeric2(value,'BigInt') as IsBigInt,
	SYSDB.dbo.IsNumeric2(value,'Int') as IsInt,
	SYSDB.dbo.IsNumeric2(value,'SmallInt') as IsSmallInt,
	SYSDB.dbo.IsNumeric2(value,'TinyInt') as IsTinyInt,
	SYSDB.dbo.IsNumeric2(value,'Decimal') as IsDecimal,
	SYSDB.dbo.IsNumeric2(value,'Numeric') as [IsNumeric],
	SYSDB.dbo.IsNumeric2(value,'Numeric(10,3)') as [IsNumeric(10,3)],
	SYSDB.dbo.IsNumeric2(value,'Numeric(10,0)') as [IsNumeric(10,0)],
	SYSDB.dbo.IsNumeric2(value,'Numeric(11,0)') as [IsNumeric(11,0)]
FROM @value

```

вернёт такой результат

<center><img title="Predictable results &mdash; as expected" src="/img/2014/isnumeric_02.png" /></center>

Функция работает ровно так, как от неё ожидают. Если хотите попробовать применить, качайте [здесь][dll].

Тьма
=====================

Вы можете предположить, что конвертация `sqlMoney = field.ToSqlMoney()` сможет интерпретировать `'-$00,0.2,,,0'` в качестве SQL `money`.

Однако, SQL [`money`][money] хранит только <strike>обычные</strike> десятичные значения. Убедимся:

<center><img title="money этот не ТОТ 'money', который можно ожидать" src="/img/2014/isnumeric_03.png" /></center>

MS SQL разбивает надежды. `SqlString.ToSqlMoney()` не признаёт знак доллара или даже точку. Возможно, логика завязана на региональные настройки логина. Но на сегодня это уже слишком.

<center><img title="точка?" src="/img/2014/isnumeric_04.png" /></center>

Тьма непроглядная
=====================

Можете почитать [SQL Server Data Type Mappings][mapping] &ndash; и увидеть, как SQL Server `money` превращается в  .NET Framework `Decimal` и имеет тип SqlDbType `Money`, однако тип DbType остаётся `Decimal`. В поисках приключений!


[msdn]:http://technet.microsoft.com/ru-ru/library/ms186272.aspx
[clr]:http://msdn.microsoft.com/library/ms254498.aspx (Introduction to SQL Server CLR Integration)
[mapping]:http://msdn.microsoft.com/en-us/library/cc716729.aspx
[dll]:https://github.com/atru/database-dll
[money]:http://technet.microsoft.com/en-us/library/ms179882.aspx
