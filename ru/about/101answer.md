---
title: Квалификационный тест 101media.ru
layout: page
lang: ru
lang_view: false
---


HTML и CSS
=====================

1. [Вёрстка 1][html1]
1. [Вёрстка 2][html2]
1. Почему javascript в `<a>` - плохо

JavaScript
=====================

1. [Array.prototype.find()][javascript1]
1. [Первый вариант][javascript2]. [Второй вариант][javascript3]

PHP
=====================

1. *Написать результат и описать последовательность выполнения кода интерпретатором.*

  ```php
  <?php
  $i = 5;
  echo −−$i + $i++;
  ?>
  ```
  Результат: 8.

  Команда `--$i` выполнится перед `echo`, получится `echo 4 + 4; //8` . Команда `$i++` выполнится после, но никто об этом не узнает.

1. *Написать результат и состояние массива на каждом шаге выполнения программы.*

  ```php
  <?php
  $arr = array(1, 3, 5, 7); //Array ( [0] => 1 [1] => 3 [2] => 5 [3] => 7)
  echo array_pop($arr); //Array ( [0] => 1 [1] => 3 [2] => 5), выведется текст "7"
  echo array_shift($arr); //Array ( [0] => 3 [1] => 5), выведется текст "1"
  echo $arr[0]; //выведется текст "3"
  echo $arr[2]; //ничто не выведется
  ?>
  ```

1.  *Каков будет результат работы этой программы. Напишите последовательность вызова конструкторов.*

  ```php
  <?php
  abstract class Foo_A {
    public $var;
    public function __construct()
    {
      $this->var = 10;  //var = 10
    }
  }
  class Foo_B extends Foo_A {
    public function __construct()
    {
      parent::__construct();  //var = 10
      $this->var = 20;  //var = 20
    }
  }
  class Foo_C extends Foo_B {
    public function __construct()
    {
      $this->var = 30;  //var = 30
      parent::__construct();   //var = 10, потом var = 20
    }
  }

  $Foo = new Foo_C();
  echo $Foo->var; // 20
?>
```

1. *Что такое «интерфейс» в ООП. Зачем они нужны?*

  Интерфейс &mdash; это т.н. чистый абстрактный класс, содержащий только абстрактные методы. Определённый класс может наследовать исключительно один абстрактный класс («являться»), однако реализовывать несколько интерфейсов («уметь делать»). Это «умение делать» может распространяться на различные по своей природе объекты: человек и машина могут быть каждый по&ndash;своему быть «сравниваемы» с другими людьми/машинами, «сортируемы», уметь «считать свой КПД» (человек &mdash; по выполненной работе, машина по пройденному пути) и т.п. При этом человек унаследовал класс «Млекопитающее», а машина &mdash; «СредствоПередвижения», которые никогда не слышали про сравниваемость, сортируемость, КПД.

  Интерфейсы нужны для универсализации в системе (полиморфизм), уменьшения связности компонентов системы, в юнит-тестировании.


SQL
=====================

1. *Запрос, который выведет список должностей, на которых числится более 10 сотрудников.*

  ```sql
  SELECT post
      FROM employers
      GROUP BY post
      HAVING COUNT(employee)>10
  ```

1. *Рассказать, как вы будете хранить и получать связанные N:N сущности и почему. Приведите примеры запросов.*

  Связанные многие&ndash;ко&ndash;многим сущности имеют каждая свой суррогатный ключ:

  `A: A_KEY (PK)`,

  `B: B_KEY (PK)`.

  Связь осуществляется дополнительной сущностью, т.н. таблицей смежности, где идут пары ключей

  `C: A_KEY(FK) B_KEY(FK)`

  Пример запроса:

    ```sql
    SELECT A.*, B.*
        FROM A
            JOIN C ON (A.A_KEY = C.A_KEY)
            JOIN B ON (B.B_KEY = C.B_KEY)
        WHERE B.EMP=101
    ```




[html1]: http://jsfiddle.net/n3CDs/
[html2]: http://jsfiddle.net/k3pSB/
[javascript1]: http://jsfiddle.net/ZCj3N/
[javascript2]: http://jsfiddle.net/AGA7x/
[javascript3]: http://jsfiddle.net/4S6GZ/



[usb3000]: http://www.r-technology.ru/products/automation/adc/usb3000.php
[окно]: http://dspsystem.narod.ru/add/win/win.html
[konsom]: http://konsom.ru
[rank]: http://anton.shevchuk.name/project-management/developers-rank/