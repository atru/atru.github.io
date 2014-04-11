---
layout: post
title:  "String concatenation aggregate function"
captitle:  "String Concatenation Aggregate Function"
date:   2014-04-10 17:05:01
categories: MSSQL
tags: sql clr
lang: en
---

This is yet another handy function, which can make one's programming easier.

Concatenate Many Rows Into A Single String
==========================================

Let us compare two simple queries:

<img width="100%" alt="Two queries" src="/img/2014/concat_aggregate_01.png" style="cursor:pointer" onclick="window.open('/img/2014/concat_aggregate_01.png','_blank');return;"></img>

As you can see, the second query aggregates values by a `GROUP BY` clause.

What does it give to us? Let's see:

<img width="100%" alt="Strings concatenated in aggregate" src="/img/2014/concat_aggregate_02.png" style="cursor:pointer" onclick="window.open('/img/2014/concat_aggregate_02.png','_blank');return;"></img>

Neat.

Even more: if you want to assemble a dynamic query for copying data from one table to another? Simply enumerate to common [columns][view_column] and wrap it into `INSERT` and `SELECT`.

Where It Came From
---------------------

The original source was taken from [here][original]. The post pretty much explains the mechanism of an MS SQL aggregate for CLR:

* the `init` of a new group;
* the `accumulate` of new values;
* the `merge` with the group;
* the `terminate` of a group;
* the `read` and `write` to serialize the struct.

What Is The Difference?
-----------------------

The mechanism didn't work sometimes. You would expect it to return `a,b,c,d,e,f`, but it gave you `a,b,cd,e,f` &mdash; for some strange reasons. It occured on bulky sets: there's a suggestion it involves multithreading. The solution was to copy the delimiter from `other` groups.

Aggregate properties:

* `IsInvariantToNulls = true`: nulls don't change the result
* `IsInvariantToDuplicates = false`: duplicates change the result
* `IsInvariantToOrder = false`: order changes the result

If you find it useful, feel free to [use][dll].

[dll]: https://github.com/atru/database-dll/
[original]: http://www.mssqltips.com/sqlservertip/2022/concat-aggregates-sql-server-clr-function/
[view_column]: {{ site.url }}/{{ page.lang }}/2014/02/18/view-column.html
