---
layout: post
title:  "Is ISNUMERIC() really about numeric?"
captitle:  "Is ISNUMERIC() Really About Numeric?"
date: 2014-03-26 11:38:28
categories: MSSQL
tags: sql clr
lang: en
---

Bad design
=====================

If you create a database engine and provide a system type `numeric`, and a bunch of other types that are similar to this one (`decimal`, `float`, `real`, `int`, `bigint`, `smallint`), you expect software developers to use a system function `ISNUMERIC()` to validate some text (eg `-0.001`) as being numeric. Totally fine. But then you also say that `money` and `smallmoney` are also numeric types, and you leave a remark on MSDN about this.

>ISNUMERIC returns 1 for some characters that are not numbers, such as plus (+), minus (-), and valid currency symbols such as the dollar sign ($). [MSDN][msdn]

Let us try and test the beast in action

<center><img title="Fuzzy logic" src="/img/2014/isnumeric_01.png" /></center>

As you can see, `ISNUMERIC('-$00,0.2,,,0')` returns `1`. I am <u title="Devdrama it is">speechless</u>.

Reinvent the wheel, shall we?
=====================

Sometimes you may need a <strike>normal</strike> more predictable text-to-numeric validation. And that's when we use the [CLR][clr].

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

We try to convert the string to the specified type. If all goes well, we return a `yes`.

Test run
=====================

The script

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

results in

<center><img title="Predictable results &mdash; as expected" src="/img/2014/isnumeric_02.png" /></center>

The function works exactly as expected. If you find it useful, download [here][dll].

Obscurity
=====================

One could imagine, that proposed `sqlMoney = field.ToSqlMoney()` would interpret `'-$00,0.2,,,0'` as SQL `money`.

But SQL [`money`][money] stores only <strike>normal</strike> decimal values. See for yourself:

<center><img title="money is not THAT 'money' one could expect" src="/img/2014/isnumeric_03.png" /></center>

MSSQL shatters one's dreams. `SqlString.ToSqlMoney()` won't accept dollar sign or dot. The logic might depend on language settings. But that is just too much for today.

<center><img title="the dot?" src="/img/2014/isnumeric_04.png" /></center>

More obscurity
=====================

Read [SQL Server Data Type Mappings][mapping] &ndash; see how SQL Server `money` becomes .NET Framework `Decimal` and has SqlDbType `Money`, yet DbType is still `Decimal`. Adventures await!


[msdn]:http://technet.microsoft.com/en-us/library/ms186272.aspx
[clr]:http://msdn.microsoft.com/library/ms254498.aspx (Introduction to SQL Server CLR Integration)
[mapping]:http://msdn.microsoft.com/en-us/library/cc716729.aspx
[dll]:https://github.com/atru/database-dll
[money]:http://technet.microsoft.com/en-us/library/ms179882.aspx
