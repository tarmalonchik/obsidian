+ [Уровни изоляции транзакций](#уровни%20изоляции%20транзакций)
+ [Индексы](#индексы)
+ [Primary key](#primary%20key)
+ [Foreign key](#foreign%20key)
+ [Unique index](#unique%20index)
+ [Составные индексы](#составные%20индексы)
+ [Explain](#explain)
+ [Vacuum](#vacuum)
+ [Pgbouncer](#Pgbouncer)
+ [Sequence](#sequence)
+ [Serial](#serial)
+ [Партицирование](#Партицирование)
+ [Locks](#locks)
+ [Transaction versions](#transaction%20versions)
+ [Оптимизация запросов]
+ [Виды JOIN](#Виды%20JOIN)
+ [Aggregate functions](#aggregate%20functions)
+ [Where vs Having](#where%20vs%20having)
+ [With As](#with%20as)
+ Триггеры vs процедуры
+ Где хранятся структуры b-tree или hash


## Уровни изоляции транзакций

## Индексы

Индексы — это распространенный способ повышения производительности базы данных. Индекс позволяет серверу базы данных находить и извлекать определенные строки гораздо быстрее, чем это было бы возможно без индекса. Но индексы также увеличивают нагрузку на систему баз данных в целом, поэтому их следует использовать разумно.

Postgres поддерживает следующие виды индексов:
+ B-tree - могут обрабатывать запросы на равенство и диапазон данных, которые можно отсортировать в определенном порядке.
	+ В частности, планировщик запросов PostgreSQL будет рассматривать возможность использования индекса B-дерева всякий раз, когда индексированный столбец участвует в сравнении с использованием одного из этих операторов `< <= = >= >`. 
	+ Конструкции, эквивалентные комбинациям этих операторов, такие как BETWEEN и IN, также могут быть реализованы с помощью поиска по индексу B-дерева. 
	+ Кроме того, условие IS NULL или IS NOT NULL для индексного столбца можно использовать с индексом B-дерева.
	+ Оптимизатор также может использовать индекс B-дерева для запросов, включающих операторы сопоставления с образцом `LIKE` и `~,` если шаблон является константой и привязан к началу строки — например, `col LIKE 'foo%'` или `col ~ '^foo'`, но не `col LIKE '%bar'`.
	+ Индексы B-дерева также можно использовать для извлечения данных в отсортированном порядке. Это не всегда быстрее, чем простое сканирование и сортировка, но часто бывает полезно.
+ Hash
	+ Хэш-индексы хранят 32-битный хэш-код, полученный на основе значения индексированного столбца. Следовательно, такие индексы могут обрабатывать только простые сравнения на равенство. Планировщик запросов будет рассматривать возможность использования хеш-индекса всякий раз, когда индексированный столбец участвует в сравнении с использованием оператора равенства:
+ GiST (Generalized Search Tree)
	+ Индексы GiST — это не отдельный вид индексов, а скорее инфраструктура, в рамках которой можно реализовать множество различных стратегий индексирования. Соответственно, конкретные операторы, с которыми можно использовать индекс GiST, различаются в зависимости от стратегии индексирования (класса операторов). Например, стандартная поставка PostgreSQL включает классы операторов GiST для нескольких двумерных геометрических типов данных, которые поддерживают индексированные запросы с использованием этих операторов: `<< &< &> >> <<| &<| |&> |>> @> <@ ~= &&`
	+ сбалансированное по высоте дерево, состоящее из узлов-страниц. Узлы состоят из индексных записей.
+ GIN (Generalized Inverted Index)
	+ Это так называемый _обратный индекс_. Он работает с типами данных, значения которых не являются атомарными, а состоят из элементов. При этом индексируются не сами значения, а отдельные элементы; каждый элемент ссылается на те значения, в которых он встречается.
	+ Основная область применения метода gin — ускорение полнотекстового поиска, поэтому логично рассматривать этот индекс более подробно именно на этом примере.
+ BRIN (Block Range Index)


## Primary key

Primary key - это первичный ключ. Он накладывает следующие ограничения:
+ автоматически создает B-tree индекс
+ не должно быть NULL значений
+ значения должны быть уникальными
+ таблица может иметь не более одного Primary key. 
+ в теории, каждая таблица должна иметь primary key, но это не обязательное условие в Postgres

## Foreign key

Foreign key - указывает, что значения в стобце (или группе столбцов) должны соответствовать значениям, появляющимся в некоторой строке другой таблицы. Мы говорим, что это поддерживает ссылочную целостность между двумя связанными таблицами.

Foreign key также может ограничивать и ссылаться на группу столбцов. Как обычно, его необходимо записать в форме табличного ограничения. Вот надуманный пример синтаксиса:

Foreign key может ссылаться на элемент родительской таблицы, только в случае, если этот элемент имеет ограничение `unique index`

```
CREATE TABLE t1 (
  a integer PRIMARY KEY,
  b integer,
  c integer,
  FOREIGN KEY (b, c) REFERENCES other_table (c1, c2)
);
```

При удалении элемента, на которую ссылается Foreign key есть несколько сценариев:
+ `RESTRICT` – если в таблице-потомке есть записи, которые ссылаются на существующий первичный ключ в таблице-потомке, то при удалении или обновлении записи с первичным ключом в таблице-предке возвратится ошибка.
+ `CASCASE` при удалении или обновлении записи в таблице-предке, которая содержит первичный ключ, автоматически удаляются или обновляются записи со ссылками на это значение в таблице-потомке.
+ `SET NULL` – при удалении или обновлении записи в таблице-предке, которая содержит первичный ключ, значения внешнего ключа в таблице-потомке устанавливаются в NULL.
+ `NO ACTION` — при удалении или обновлении записи в таблице-предке, которая содержит первичный ключ, в таблице-потомке никаких действий предприниматься не будет.
+ `SET DEFAULT` – тут понятно из названия, что при удалении или обновлении записи в таблице-предке, которая содержит первичный ключ, в таблице-потомке соответствующим записям будет выставлено значение по умолчанию.


## Unique index

В Postgres уникальными могут быть объявлены только индексы B-дерева. То есть при создании ограничения `unique index`, столбец становится также и B-tree индексом. 


## Составные индексы

Индекс может быть определен более чем для одного столбца таблицы. Например, если у вас есть таблица такого вида:

```
CREATE TABLE test2 (
  major int,
  minor int,
  name varchar
);
```

И при этом часто выполняется такой запрос: 
```
SELECT name FROM test2 WHERE major = _`constant`_ AND minor = _`constant`_;
```

Тогда можно определить составной индекс в следующем виде: 
```
CREATE INDEX test2_mm_idx ON test2 (major, minor);
```

Составные индексы поддерживают только следующие виды индексов:
+ B-tree
+ GiST
+ GIN
+ BRIN

Индексы могут иметь до 32 колонок. 

При создании составного индекса, порядок колонок имеет значение. Так первое значение имеет наивысший приоритет. 
Если создать составной индекс `firstname`+ `lastname`, тогда при поиске по колонке с запросом:
+ `where firtname='Ivan'`, Postgres просканирует индекс, и запрос будет довольно быстрым
+ `where lastname='Ivan'`, Postgres может сделать `seq scan`, то есть он пройдется по всем записям, и это будет поиск не по индексу. 


## Sequence

Sequence - это не тип. Это объект базы данных, который может создавать последовательности. 

Создание sequence: 
`create sequence 'seq_name';  

Получение значения из sequence:
`select nextval('seq_name'::regclass);`

Объект `sequence` не вызывает проблем при обращении из разных транзакций. При этом разные транзакции получают разные значения. 

## Serial

Serial - это фиктивный тип. Он создает собственный sequence объект, и привязывает его к таблице.  Данный тип был специально создан для работы с `sequence` объектами, который при генерации таблицы превращается примерно в:
```
id integer NOT NULL DEFAULT nextval('tbl_id_seq')`
```

Для того, чтобы не обращаться к sequence по имени, можно использовать функцию для получения имени `sequence`:
```
pg_get_serial_sequence(table_name, column_name)
```

Однако, из за того, что каждая транзакция получает свое уникальное значение, то при откате какой либо транзакции, в последовательности могут быть пробелы. 

`serial` не накладывает каких либо правил по уникальности поля таблицы. Для того, чтобы поле стало уникальным, необходимо создать `unique index`

Начиная **с PostgreSQL 10**, появилась возможность объявления идентифицирующего столбца(`GENERATED AS IDENTITY`), соответствующего стандарту SQL:2003
+ `GENERATED BY DEFAULT AS IDENTITY` - поведение эквивалентно `serial`
+ `GENERATED ALWAYS AS IDENTITY` - отличается от serial тем, что не позволяет вставлять какое либо значение, кроме `defaul`. Для того, чтобы выставить значение отличное от `default`, необходимо воспользоваться командой `OVERRIDING SYSTEM VALUE`. 


## Pgbouncer

Pgbouncer — это программа, управляющая пулом соединений Postgres Pro. Любое конечное приложение может подключиться к pgbouncer, как если бы это был непосредственно сервер Postgres Pro, и pgbouncer создаст подключение к реальному серверу, либо задействует одно из ранее установленных подключений.

Pgbouncer поддерживает несколько видов пуллов:
+ Пул сеансов - наиболее корректный метод. Когда клиент подключается, ему назначается одно серверное подключение на всё время, пока клиент остаётся подключённым. Когда клиент отключается, это подключение к серверу возвращается в пул. Этот метод работает по умолчанию.
+ Пул транзакций - подключение к серверу назначается клиенту только на время транзакции. Когда pgbouncer замечает, что транзакция завершена, это подключение возвращается в пул.
+ Пул операторов - наиболее агрессивный метод. Подключение к серверу будет возвращаться в пул сразу после завершения каждого запроса. Транзакции с несколькими операторами в этом режиме запрещаются, так как они не будут работать.


## Aggregate functions

Как и большинство других продуктов реляционных баз данных, PostgreSQL поддерживает агрегатные функции. Агрегатная функция вычисляет один результат из нескольких входных строк. Например, существуют агрегаты для вычисления количества, sum(), avg(), max() и min() по набору строк. Пример: 
```
SELECT max(temp_lo) FROM weather;
```

Aggregate functions также очень полезны в сочетании с предложениями GROUP BY. Например, мы можем получить количество показаний и максимальную низкую температуру, наблюдаемую в каждом городе, с помощью:
```
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city;
```
что дает нам одну выходную строку для каждого города. Каждый совокупный результат вычисляется по строкам таблицы, соответствующим этому городу. Мы можем фильтровать эти сгруппированные строки, используя HAVING:
```
SELECT city, count(*), max(temp_lo)
    FROM weather
    GROUP BY city
    HAVING max(temp_lo) < 40;
```


## Where vs Having

| Criteria                 | Where                                                | Having                                                 |
| ------------------------ | ---------------------------------------------------- | ------------------------------------------------------ |
| Purpose                  | Filters rows before aggregation                      | Filters rows after aggregation                         |
| Used With                | Works with individual rows                           | Works with grouped or aggregated data                  |
| Applicable To            | Columns in the table (individual records)            | Aggregated results (e.g., `SUM()`, `COUNT()`)          |
| Stage of Query Execution | Filters data before `GROUP BY`                       | Filters data after `GROUP BY`                          |
| Aggregate Functions      | Cannot be used with aggregate functions              | Can be used with aggregate functions                   |
| Efficiency               | Generally more efficient, as it filters data earlier | Can be less efficient, as it filters after aggregation |
| Example Usage            | WHERE column_name = value                            | HAVING aggregate_function(column_name) > value         |
| Order in Query           | Appears before `GROUP BY`                            | Appears after `GROUP BY`                               |

# With As

Основная ценность SELECT в операторе With — разбить сложные запросы на более простые части. Примером является:
```
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total_sales
    FROM orders
    GROUP BY region
), top_regions AS (
    SELECT region
    FROM regional_sales
    WHERE total_sales > (SELECT SUM(total_sales)/10 FROM regional_sales)
)
SELECT region,
       product,
       SUM(quantity) AS product_units,
       SUM(amount) AS product_sales
FROM orders
WHERE region IN (SELECT region FROM top_regions)
GROUP BY region, product;
```

Необязательный модификатор RECURSIVE превращает FOR из простого синтаксического удобства в функцию, которая выполняет функции, которые иначе невозможны в стандартном SQL. Используя RECURSIVE, запрос With может ссылаться на собственный вывод. Пример:
```
WITH RECURSIVE r AS (  
    SELECT 1 AS i, 1 AS factorial  
    UNION  
    SELECT i+1 AS i, factorial * (i+1) as factorial  
    FROM r  
    WHERE i < 10  
)  
SELECT * FROM r;
```


## Explain

`EXPLAIN` — показывает план выполнения оператора. 

Пример использования - `EXPLAIN [ ( _`option`_ [, ...] ) ] _`statement`_`

Эта команда отображает план выполнения, созданный планировщиком PostgreSQL для предоставленного оператора. План выполнения показывает, как будут сканироваться таблицы, на которые ссылается оператор — путем простого последовательного сканирования, сканирования по индексу и т. д. — и если имеется ссылка на несколько таблиц, какие алгоритмы соединения будут использоваться для объединения необходимых строк из каждой входной таблицы.

Наиболее важной частью отображения является предполагаемая стоимость выполнения инструкции, которая представляет собой предположение планировщика о том, сколько времени потребуется для выполнения инструкции (измеряется в произвольных единицах стоимости, но обычно означает выборку страниц с диска). На самом деле показаны два числа: начальная стоимость до того, как можно будет вернуть первую строку, и общая стоимость возврата всех строк. Для большинства запросов важна общая стоимость, но в таких контекстах, как подзапрос в EXISTS, планировщик выберет наименьшую начальную стоимость вместо наименьшей общей стоимости (поскольку исполнитель все равно остановится после получения одной строки). Кроме того, если вы ограничите количество возвращаемых строк с помощью предложения LIMIT, планировщик выполнит соответствующую интерполяцию между затратами конечной точки, чтобы оценить, какой план действительно является самым дешевым.

`EXPLAIN ANALYZE` обеспечивает фактическое выполнение оператора, а не только запланированное. Затем на дисплей добавляется фактическая статистика времени выполнения, включая общее время, затраченное на каждый узел плана (в миллисекундах), и общее количество строк, которые он фактически вернул. Это полезно для проверки того, близки ли оценки планировщика к реальности.


## Vacuum

`VACUUM` — сбор мусора и, при необходимости, анализ базы данных.

`VACUUM` освобождает память, занятую мертвыми данными. При нормальной работе PostgreSQL кортежи, удаленные или устаревшие в результате обновления, физически не удаляются из таблицы; они остаются присутствовать до тех пор, пока не будет выполнен ВАКУУМ. Поэтому необходимо периодически делать `VACUUM`, особенно на часто обновляемых таблицах.

Без списка table_and_columns VACUUM обрабатывает каждую таблицу и  текущей базе данных, на очистку которых у текущего пользователя есть разрешение. При использовании списка VACUUM обрабатывает только эти таблицы.

`VACUUM ANALYZE`  выполняет `VACUUM`, а затем `ANALYZE` для каждой выбранной таблицы.

Виды:
+ `VACUUM` - удаляет мусорные строки, но не возвращает место в ОС. Насколько я понял, при этом Postgres сможет переиспользовать эти данные в будущем. Но они не вернутся в ОС. 
+ `VACUUM FULL` - полностью реорганизует таблицу, возвращает место в ОС, но блокирует доступ к таблице.  В этом кейсе, Postgres реально очистит данные, и они станут свободными. Те база данных начент весить меньше. 
+ `AUTOVACUUM` - фоновый процесс, который автоматически запускает VACUUM, чтобы не приходилось делать это вручную. `AUTOVACUUM` включен по умолчанию в PosgtgreSQL.

## Виды JOIN

+ `CROSS JOIN` -  это пересечение каждого элемента первой таблицы с каждым элементом второй таблицы. Это значит, что если в таблице А - 5 записей, а в таблице Б - 10 записей, то  после `cross join`  получится 50 записей.  `cross join` не поддерживает `on` условие, в котором можно задать условие соединения. 
+ `INNER JOIN or JOIN`. Это два эквивалента одной и той же команды. Данное объединение логический эквивалент `cross join` с дополнительным условием. 
+ `LEFT JOIN` - это логический эквивалент `inner join` с добавлением строк из первой таблицы, для которых не нашлось ничего в правой таблице.  
+ `RIGHT JOIN` - это то же самое, что и `left join`, только отзеркаленное. 
+ `FULL OUTER JOIN` - это логический эквивалент `inner join` с добавлением строк из первой таблицы, для которых не нашлось ничего в правой таблице и с добавлением строк из второй таблицы, для которых не нашлось ничего в левой. Т.е `LEFT JOIN` и `RIGHT JOIN` вернут подмножества `FULL OUTER JOIN` - на

## Transaction versions

MVCC (англ. multiversion concurrency control — управление параллельным доступом посредством многоверсионности) — один из механизмов СУБД для обеспечения параллельного доступа к базам данных, заключающийся в предоставлении каждому пользователю так называемого «снимка» базы, обладающего тем свойством, что вносимые пользователем изменения невидимы другим пользователям до момента фиксации транзакции. Этот способ управления позволяет добиться того, что пишущие транзакции не блокируют читающих, и читающие транзакции не блокируют пишущих.

В отличие от большинства других систем баз данных, которые используют блокировки для управления параллелизмом, Postgres поддерживает согласованность данных, используя многоверсионную модель. Это означает, что при запросе к базе данных каждая транзакция видит снимок данных (версию базы данных), какой она была некоторое время назад, независимо от текущего состояния базовых данных. Это защищает транзакцию от просмотра противоречивых данных, которые могут быть вызваны (другими) одновременными обновлениями транзакций в одних и тех же строках данных, обеспечивая изоляцию транзакций для каждого сеанса базы данных.

Основное различие между моделями мультиверсионности и блокировки заключается в том, что в MVCC блокировки, полученные для запроса (чтения) данных, не конфликтуют с блокировками, полученными для записи данных, и поэтому чтение никогда не блокирует запись, а запись никогда не блокирует чтение.

Каждая строка может одновременно присутствовать в базе данных в нескольких версиях. Одну версию от другой надо как-то отличать С этой целью каждая версия имеет две отметки, определяющие «время» действия данной версии (xmin и xmax). В кавычках — потому, что используется не время как таковое, а специальный увеличивающийся счетчик. И этот счетчик — номер транзакции.
+ Когда строка создается, значение xmin устанавливается в номер транзакции, выполнившей команду INSERT, а xmax не заполняется.
+ Когда строка удаляется, значение xmax текущей версии помечается номером транзакции, выполнившей DELETE.
+ Когда строка изменяется командой UPDATE, фактически выполняются две операции: DELETE и INSERT. В текущей версии строки устанавливается xmax, равный номеру транзакции, выполнившей UPDATE. Затем создается новая версия той же строки; значение xmin у нее совпадает с значением xmax предыдущей версии.



## Партицирование