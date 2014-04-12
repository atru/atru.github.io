---
layout: post
title:  "Конкатенация строк при группировке"
captitle:  "Конкатенация строк при группировке"
date:   2014-04-10 17:05:01
categories: MSSQL
tags: sql clr
lang: ru
---

Это ещё одна функция, которая может значительно облегчить жизнь разработчику.

Слепим много кортежей в одну строку
===================================

Сравним два запроса:

<center><img alt="Two queries" src="/img/2014/concat_aggregate_01.png" style="cursor:pointer" onclick="window.open('/img/2014/concat_aggregate_01.png','_blank');return;"/></center>

Как видите, второй запрос агрегирует значения по правилу в `GROUP BY`.

Что нам это даёт? Посмотрим:

<center><img alt="Strings concatenated in aggregate" src="/img/2014/concat_aggregate_02.png" style="cursor:pointer" onclick="window.open('/img/2014/concat_aggregate_02.png','_blank');return;"/></center>

Отлично.

Более того, хотите составить динамический запрос по копированию данных из одной таблицы в другую? Просто перечисляйте колонки [отсюда][view_column] вкупе с любым `INSERT` и `SELECT`.

Откуда ноги растут
---------------------

Оригинальный код взят [здесь][original]. В сообщении практически разжёван механизм агрегирования в MS SQL для CLR:

* `init` новой группы;
* `accumulate` новых значений;
* `merge` с группой;
* `terminate` группу;
* `read` и `write` для сериализации.

В чём разница?
---------------------

Механизм работал не всегда. Вы ожидали, что он вернёт `a,b,c,d,e,f`, но он выдавал `a,b,cd,e,f` &mdash; по странным причинам. Это проявлялось на объёмных выборках: есть предположение, что замешана многопоточность. Решилось копированием разделителя из соседних групп.

Свойства функции:

* `IsInvariantToNulls = true`: пустые значения не влияют на результат
* `IsInvariantToDuplicates = false`: дубликаты влияют на результат
* `IsInvariantToOrder = false`: порядок влияет на результат

Если считаете функцию полезной, [добро пожаловать][dll].

[dll]: https://github.com/atru/database-dll/
[original]: http://www.mssqltips.com/sqlservertip/2022/concat-aggregates-sql-server-clr-function/
[view_column]: {{ site.url }}/{{ page.lang }}/2014/02/18/view-column.html