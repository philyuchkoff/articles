# WAL-G — инструмент для управления бэкапом и восстановлением БД

* Исходники на  [Github](https://github.com/wal-g/wal-g).
* Статья на [Medium](https://medium.com/@philyuchkoff/wal-g-953490c74b98)

![](https://miro.medium.com/max/2560/1*4RDYehIXv93nUZX3_sWqBA.jpeg)

WAL-G — простой и эффективный инструмент для резервного копирования PostgreSQL (а еще MySQL/MariaDB, MongoDB и FoundationDB) в почти любые облака (Amazon S3/MinIO, Yandex Object Storage, Google Cloud Storage, Azure Storage, Swift Object Storage и просто на файловую систему).

В WAL-G есть одна одна очень полезная особенность — дельта-копии, которые хранят страницы файлов, изменившиеся с предыдущей версии резервной копии.

Расскажу на примере PostgreSQL и MongoDB, как пользоваться этой штукой.

## MinIO
Для хранения бэкапов я поднял у себя  [MinIO](https://min.io/)  — легкое хранилище объектов, совместимое с  [Amazon S3](https://aws.amazon.com/s3/):

````
wget https://dl.min.io/server/minio/release/linux-amd64/minio
chmod +x minio
MINIO_ROOT_USER=admin MINIO_ROOT_PASSWORD=password ./minio server /mnt/data --console-address ":9001"
sudo systemctl start minio
sudo systemctl enable minio
sudo systemctl status minio
````

## PostgreSQL

Для теста на одном из хостов (CentOS 7) поднимем PostgreSQL 14:
````
# Install the repository RPM:
sudo yum install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-7-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Install PostgreSQL:
sudo yum install -y postgresql14-server

# Optionally initialize the database and enable automatic start:
sudo /usr/pgsql-14/bin/postgresql-14-setup initdb
sudo systemctl enable postgresql-14
sudo systemctl start postgresql-14
````

создаю тестовую базу:

````
su — postgres   
psql   
create database test1;   
\c test1;   
CREATE TABLE indexing_table(created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW());
````

Генерирую и вливаю в тестовую базу данные размером около 1 Гб:

````
CREATE TABLE large_for_test (num1 bigint, num2 double precision, num3 double precision);   
INSERT INTO large_for_test (num1, num2, num3)   
SELECT round(random()*10), random(), random()*142   
FROM generate_series(1, 20000000) s(i);   
EXPLAIN (analyse, buffers)   
SELECT num1, avg(num3) as num3_avg, sum(num2) as num2_sum   
FROM large_for_test   
GROUP BY num1;
````

Чтобы архиватор внутри базы сам заливал WAL-журналы в облако и восстанавливался из них (в случае необходимости) настраиваю некоторые параметры в  `postgresql.conf`:

````
wal_level=replica   
archive_mode=on   
archive_command=’/usr/local/bin/wal-g wal-push \”%p\” >> /var/log/postgresql/archive_command.log 2>&1'   
archive_timeout=60   
restore_command=’/usr/local/bin/wal-g wal-fetch \”%f\” \”%p\” >> /var/log/postgresql/restore_command.log 2>&1'
````

Подробнее обо всех этих параметрах можно прочитать в  [переводе официальной документации](https://postgrespro.ru/docs/postgresql/12/runtime-config-wal).

И рестартую PostgreSQL:

`systemctl restart postgresql-14`

На тот же хост, где и PostgreSQL, ставлю WAL-G последней на данный момент версии 2.0.0 (по сути — это просто один бинарник, без внешних зависимостей):

````
curl -L "https://github.com/wal-g/wal-g/releases/download/v0.2.15/wal-g.linux-amd64.tar.gz" -o "wal-g.linux-amd64.tar.gz"
tar -xzf wal-g.linux-amd64.tar.gz
mv wal-g /usr/local/bin/
````

Для конфигурирования WAL-G нужно положить файл  `.walg.json`  (пример файла  [здесь](https://github.com/wal-g/wal-g/issues/545)) в домашнюю директорию пользователя  `postgres`:

````
cat > /var/lib/pgsql/.walg.json << EOF
{
"AWS_ENDPOINT": "http://10.0.0.160:9000",
"WALG_S3_PREFIX": "s3://pgbackup",
"AWS_ACCESS_KEY_ID": "minioadmin",
"AWS_SECRET_ACCESS_KEY": "minioadmin",
"AWS_S3_FORCE_PATH_STYLE": "true",
"WALG_COMPRESSION_METHOD": "brotli",
"WALG_DELTA_MAX_STEPS": "5",
"PGDATA": "/var/lib/pgsql/12/main",
"PGHOST": "/var/run/postgresql/.s.PGSQL.5432"
}
EOF
````

Не забываем поменять владельца этого файла:

`chown postgres: /var/lib/pgsql/.walg.json`

## Бэкап

Проверяю создание полного бэкапа: в WAL-G это аргумент запуска  `backup-push`. Для начала выполняю эту команду вручную от пользователя  `postgres`, чтобы убедиться что всё хорошо:

````
time su — postgres -c ‘/usr/local/bin/wal-g backup-push /var/lib/pgsql/12/data’   
INFO: 2020/08/17 06:05:00.211318 Couldn’t find previous backup. Doing full backup.   
INFO: 2020/08/17 06:05:00.225448 Calling pg_start_backup()   
INFO: 2020/08/17 06:05:01.819922 Walking …   
INFO: 2020/08/17 06:05:01.820466 Starting part 1 …   
INFO: 2020/08/17 06:05:21.049285 Finished writing part 1.   
INFO: 2020/08/17 06:05:21.049323 Starting part 2 …   
INFO: 2020/08/17 06:05:21.149069 Finished writing part 2.   
INFO: 2020/08/17 06:05:21.659995 Starting part 3 …   
INFO: 2020/08/17 06:05:21.660060 /global/pg_control   
INFO: 2020/08/17 06:05:21.660309 Finished writing part 3.   
INFO: 2020/08/17 06:05:21.662065 Calling pg_stop_backup()   
INFO: 2020/08/17 06:05:23.878346 Starting part 4 …   
INFO: 2020/08/17 06:05:23.878460 backup_label   
INFO: 2020/08/17 06:05:23.878490 tablespace_map   
INFO: 2020/08/17 06:05:23.878635 Finished writing part 4.   
INFO: 2020/08/17 06:05:24.924564 Wrote backup with name base_00000001000000000000007E   
   
real 0m24.829s   
user 0m20.669s   
sys 0m2.323s
````

Бэкап ~1 Гб БД выполняется очень быстро.

В MinIO в нужном бакете появляются бэкапы, 1 Гб БД ужимается до 351.92 Мб.

Если всё прошло без ошибок, то далее можно добавить периодический запуск этой команды в  `crontab`. Необходимо учесть, что хранение всех бэкапов вечно не имеет никакого смысла, поэтому нужно настроить удаление старых резервных копий (можно в том же кроне).

## Восстановление

Попробуем восстановиться по-взрослому, удалив все данные (БД и пользователей):

Останавливаю PostrgeSQL:

`systemctl stop postgresql-12`

Удаляю все данные из текущей базы:

`rm -rf /var/lib/pgsql/12/data`

Скачиваю резервную копию и разархивирую её:

`su — postgres -c ‘/usr/local/bin/wal-g backup-fetch /var/lib/pgsql/12/data LATEST’`

От имени пользователя  _postgres_  помещаю рядом с базой специальный файл-сигнал для восстановления (подробнее  [здесь](https://postgrespro.ru/docs/postgresql/12/runtime-config-wal#RUNTIME-CONFIG-WAL-ARCHIVE-RECOVERY)) :

`su — postgres -c ‘touch /var/lib/pgsql/12/data/recovery.signal’`

Запускаю базу данных, чтобы она инициировала процесс восстановления:

`systemctl start postgresql-12 && systemctl status postgresql-12`

Смотрим логи, там красота :)

````
2020–08–17 08:57:55.281 EDT [25536] LOG: starting archive recovery   
2020–08–17 08:57:55.876 EDT [25536] LOG: restored log file “000000010000000100000044” from archive   
2020–08–17 08:57:56.617 EDT [25536] LOG: redo starts at 1/44000028   
2020–08–17 08:57:56.834 EDT [25536] LOG: restored log file “000000010000000100000045” from archive   
2020–08–17 08:57:58.006 EDT [25536] LOG: consistent recovery state reached at 1/45000050   
2020–08–17 08:57:58.007 EDT [25533] LOG: database system is ready to accept read only connections   
2020–08–17 08:57:58.223 EDT [25536] LOG: restored log file “000000010000000100000046” from archive   
2020–08–17 08:57:59.811 EDT [25536] LOG: redo done at 1/46000148   
2020–08–17 08:58:00.096 EDT [25536] LOG: restored log file “000000010000000100000046” from archive   
2020–08–17 08:58:00.740 EDT [25536] LOG: selected new timeline ID: 2   
2020–08–17 08:58:02.291 EDT [25536] LOG: archive recovery complete   
2020–08–17 08:58:07.408 EDT [25533] LOG: database system is ready to accept connections
````

Проверяем, что все на месте, размеры совпадают. Все отлично работает.

## Восстановление на определенное время

Если нужно восстановить базу на определенную минуту, то в  `recovery.conf`  добавляем параметр  `recovery_target_time`  (задается в формате  `YYYY-MM-DD HH:mm:ss`)— указываем на какое время восстановить базу:

`````
restore_command = ‘/usr/local/bin/wal-fetch.sh “%f” “%p”’   
recovery_target_time = ‘2020–08–14 09:00:00’
`````

## Резюмируя по Postgres

-   На каждом хосте с Postgres устанавливаем WAL-G
-   на каждом хосте с Postgres правим  `postgresql.conf`
-   в  `home`  пользователя  `postrges`  кладем  `.walg.json`
-   ставим команду резервного копирования (и зачистки ненужных старых бэкапов) в  `cron`

Чтобы в дальнейшем все было красиво, я у себя для каждого хоста с БД создаю отдельный бакет (например, по имени хоста).

# MongoDB

Для тестов на одном из хостов поднимаю MongoDB 4.4 и использую тот же инстанс MinIO c созданным под эти цели бакетом  `mongo`.

Заливаю в Монгу тестовые данные:
````
curl -LO https://raw.githubusercontent.com/mongodb/docs-assets/primer-dataset/primer-dataset.json
mongoimport --db test --collection restaurants --file /tmp/primer-dataset.json

2020-08-18T06:05:46.661-0400 connected to: mongodb://localhost/
2020-08-18T06:05:48.746-0400 25359 document(s) imported successfully. 0 document(s) failed to import.
````

## Бэкап

В директории пользователя, от имени которого запускается бэкап, должен лежать файл  `.walg.json`  c примерно следующим содержимым (подробное описание параметров  [в документации](https://github.com/wal-g/wal-g/blob/master/MongoDB.md)):

````
{   
 “WALG_S3_PREFIX”: “s3://mongo”,   
 “AWS_ACCESS_KEY_ID”: “minioadmin”,   
 “AWS_SECRET_ACCESS_KEY”: “minioadmin”,   
 “AWS_ENDPOINT”: “http://minio:9000",   
 “AWS_S3_FORCE_PATH_STYLE”: “true”,   
 “WALG_STREAM_CREATE_COMMAND”: “mongodump — archive — uri=\”mongodb://127.0.0.1:27017\””,   
 “WALG_STREAM_RESTORE_COMMAND”: “mongorestore — archive — drop — uri=\”mongodb://127.0.0.1:27017\””,   
 “MONGODB_URI”: “mongodb://127.0.0.1:27017”,   
 “WALG_SENTINEL_USER_DATA”: “{\”key\”: \”value\”}”,   
 “OPLOG_ARCHIVE_TIMEOUT_INTERVAL”: “5s”,   
 “OPLOG_ARCHIVE_AFTER_SIZE”: “1048576”,   
 “WALG_COMPRESSION_METHOD”: “brotli”,   
 “WALG_DISK_RATE_LIMIT”: “41943040”,   
 “WALG_NETWORK_RATE_LIMIT”: “10485760”,   
 “WALG_LOG_LEVEL”: “DEVEL”,   
 “WALG_UPLOAD_CONCURRENCY” : 1,   
 “OPLOG_PITR_DISCOVERY_INTERVAL”: “168h”   
} 
````

Особо нужно обратить внимание на параметр  `MONGODB_URI`, который используется для подключения к инстансу MongoDB — например, при необходимости подключения к определенной БД, при необходимости указать пользователя/логин/пароль для подключения и т.п. ([документация](https://docs.mongodb.com/manual/reference/connection-string/)).

Для бэкапа нужно выполнить команду

`wal-g backup-push`
   
````
2020–09–05T03:51:21.247–0400 writing admin.system.version to archive on stdout   
2020–09–05T03:51:21.252–0400 done dumping admin.system.version (1 document)   
2020–09–05T03:51:21.252–0400 writing test.restaurants to archive on stdout   
2020–09–05T03:51:21.324–0400 done dumping test.restaurants (25359 documents)   
INFO: 2020/09/05 03:51:21.504031 FILE PATH: stream_20200905T075121Z/stream.br   
INFO: 2020/09/05 03:51:21.511747 Removing sigmask. Shutting down
````

После выполнения команды бэкап появляется в бакете.

Для периодичного бэкапа можно запускать эту команду в cron’e (и опять же, не забываем удалять старые бэкапы).

## Восстановление

Смотрим командой  `wal-g backup-list`  список имеющихся у нас бэкапов:

````
wal-g backup-list  
 
name last_modified wal_segment_backup_start   
stream_20200905T075121Z 2020–09–05T07:51:21Z ZZZZZZZZZZZZZZZZZZZZZZZZ   
stream_20200905T082140Z 2020–09–05T08:21:40Z ZZZZZZZZZZZZZZZZZZZZZZZZ
````

и командой  `wal-g backup-fetch backup-name`  восстанавливаем нужный нам бэкап:

````
wal-g backup-fetch stream_20200905T075121Z   
   
2020–09–05T04:11:39.297–0400 preparing collections to restore from   
2020–09–05T04:11:40.295–0400 reading metadata for test.restaurants from archive on stdin   
2020–09–05T04:11:40.319–0400 restoring test.restaurants from archive on stdin   
2020–09–05T04:11:40.717–0400 no indexes to restore   
2020–09–05T04:11:40.717–0400 finished restoring test.restaurants (25359 documents, 0 failures)   
2020–09–05T04:11:40.717–0400 25359 document(s) restored successfully. 0 document(s) failed to restore.   
INFO: 2020/09/05 04:11:40.720019 Removing sigmask. Shutting down
````

Чем WAL-G лучше того же  pg_dump (или  pg_basebackup или Barman) хорошо написано  [здесь](https://habr.com/ru/company/yandex/blog/415817/)  (очень хорошая статья для понимания, как это устроено и работает), если вкратце — то это инструмент разработчика и он не предназначен для работы с высоконагруженной БД.

Рекомендую попробовать, уверен, что понравится :)

И можно поставить лайк [статье](https://medium.com/@philyuchkoff/wal-g-953490c74b98), если понравилась :)
