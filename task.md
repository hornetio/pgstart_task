# Тестовое задание инженера по отказоустойчивости
Здравствуйте, меня зовут Емельянов Илья и это мое решение тестового задания от postgres.



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
### Запрос
