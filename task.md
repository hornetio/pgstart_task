# Тестовое задание инженера по отказоустойчивости
Здравствуйте, меня зовут Емельянов Илья и это мое решение тестового задания от PGStart.



### Рабочее окружение
Был склонирован репозиторий для выполнения задачи
```git
git clone https://github.com/pkonotopov/ndxg65cdMD2zudNrK4bNXtFuRkJ92V test-env
```
Далее был запущен docker container
```
docker compose up -d
```
А также было проверено состояние контейнеров
<pre>NAME                  IMAGE               COMMAND                  SERVICE    CREATED          STATUS                      PORTS
interview_metabase    metabase/metabase   &quot;/app/run_metabase.sh&quot;   metabase   21 minutes ago   Up 21 minutes               0.0.0.0:13030-&gt;3000/tcp
tasks_postgres        postgres:14         &quot;docker-entrypoint.s…&quot;   postgres   21 minutes ago   Up 21 minutes               0.0.0.0:15466-&gt;5432/tcp
tasks_postgres_load   ubuntu              &quot;bash -c &apos;apt update…&quot;   load       21 minutes ago   Exited (0) 17 minutes ago   
</pre>
После этого подключился к БД
```
docker compose exec postgres psql -U postgres interview
```
После этого переходим в DBeaver и подключаемся к localhost:15466


В итоге запущен контейнер, а также имеется подключение к базе данных, поэтому можно переходить непосредственно к задаче.
### Оптимизация запроса
Далее было необходимо выполнить следующий запрос:
```
explain analyze verbose
SELECT body 
  FROM posts 
 WHERE body ILIKE '%postgres%awesome%'
    OR body ILIKE '%postgres%amazing%';
```
Вот план данного запроса:
|QUERY PLAN|
|----------|
|Gather  (cost=1000.00..29347.07 rows=44 width=839) (actual time=3600.432..35493.502 rows=40 loops=1)|
|  Output: body|
|  Workers Planned: 2|
|  Workers Launched: 2|
|  ->  Parallel Seq Scan on public.posts  (cost=0.00..28342.67 rows=18 width=839) (actual time=3578.881..35389.263 rows=13 loops=3)|
|        Output: body|
|        Filter: ((posts.body ~~* '%postgres%awesome%'::text) OR (posts.body ~~* '%postgres%amazing%'::text))|
|        Rows Removed by Filter: 73451|
|        Worker 0:  actual time=2218.249..35362.229 rows=9 loops=1|
|        Worker 1:  actual time=4927.508..35350.127 rows=17 loops=1|
|Planning Time: 25.177 ms|
|Execution Time: 35493.888 ms|


Как видим, запланированное и фактическое время выполнения отличаются.


Для ускорения выполнения запроса было решено пойти самым тривиальным путем - создание GIN(generalized inverted index)


Необходимо было довавить расширение pq_trgm для like выражений
```
create extension if not exists pg_trgm;
```
Далее был создан индекс (создание заняло около минуты)
```
create index ix_body_posts on posts using GIN (body gin_trgm_ops);
```
Теперь посмотрим, что напишет нам планировщик в итоге выполненных действий
```
explain analyze verbose
SELECT body 
  FROM posts 
 WHERE body ILIKE '%postgres%awesome%'
    OR body ILIKE '%postgres%amazing%';
```
|QUERY PLAN|
|----------|
|Bitmap Heap Scan on public.posts  (cost=296.35..467.68 rows=44 width=839) (actual time=32.650..42.851 rows=40 loops=1)|
|  Output: body|
|  Recheck Cond: ((posts.body ~~* '%postgres%awesome%'::text) OR (posts.body ~~* '%postgres%amazing%'::text))|
|  Rows Removed by Index Recheck: 20|
|  Heap Blocks: exact=59|
|  ->  BitmapOr  (cost=296.35..296.35 rows=44 width=0) (actual time=32.316..32.321 rows=0 loops=1)|
|        ->  Bitmap Index Scan on ix_body_posts  (cost=0.00..148.17 rows=22 width=0) (actual time=17.038..17.039 rows=36 loops=1)|
|              Index Cond: (posts.body ~~* '%postgres%awesome%'::text)|
|        ->  Bitmap Index Scan on ix_body_posts  (cost=0.00..148.17 rows=22 width=0) (actual time=15.266..15.267 rows=24 loops=1)|
|              Index Cond: (posts.body ~~* '%postgres%amazing%'::text)|
|Planning Time: 8.583 ms|
|Execution Time: 43.151 ms|


Как видим скорость выполнения запроса значительно улучшилась, перейдем к ответам на вопросы.
## Создание индекса в нагруженной системе
Планирование и создание индекса

Когда база данных находится под высокой транзакционной нагрузкой, создание индексов может значительно повлиять на производительность системы. В связи с этим необходимо тщательно планировать и проводить этот процесс.

Анализ нагрузки: Перед созданием индекса важно проанализировать текущую нагрузку на систему. Это позволит определить, какие запросы испытывают наибольшее давление и требуют оптимизации с помощью индексов.

Выбор подходящего момента: Желательно создавать индексы во время низкой активности системы, чтобы минимизировать влияние на работу пользователей. Это может быть период ночных часов или другие временные интервалы, когда активность пользователей минимальна.

Партиционирование: Если база данных подвержена значительной нагрузке, можно рассмотрить возможность партиционирования. Это позволит создавать индексы на отдельных частях данных, снижая влияние операций создания индексов на работу всей базы данных.

Использование CONCURRENTLY: В PostgreSQL существует возможность создания индексов с помощью ключевого слова CONCURRENTLY, которое позволяет создавать индексы без блокировки таблицы. Это полезно в нагруженных системах, так как позволяет избежать длительных блокировок, которые могут привести к задержкам в обработке запросов.

## Параметры PostgreSQL для оптимизации создания индекса

Для ускорения создания индексов в условиях повышенной нагрузки можно настроить некоторые параметры PostgreSQL:

maintenance_work_mem: Увеличение этого параметра позволяет выделить больше памяти для операций обслуживания, включая создание индексов. Однако следует быть осторожным, чтобы не выделить слишком много памяти, что может привести к увеличению нагрузки на систему.

max_parallel_workers: Настройка этого параметра позволяет PostgreSQL использовать больше параллельных рабочих процессов для выполнения операций, включая создание индексов. Это может ускорить процесс создания индексов в многоядерных системах.

wal_level и max_wal_size: Настройка параметров журнализации транзакций может повлиять на скорость создания индексов. Увеличение уровня журнализации и размера WAL-файлов может улучшить производительность во время создания индексов, за счет снижения накладных расходов на журнализацию.



