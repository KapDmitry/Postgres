### 1 Проверим производительность single instance на запись 
Запрос на вставку взял из лекции

```
INSERT INTO book.tickets (fkRide, fio, contact, fkSeat)
VALUES (
	ceil(random()*100)
	, (array(SELECT fam FROM book.fam))[ceil(random()*110)]::text || ' ' ||
    (array(SELECT nam FROM book.nam))[ceil(random()*110)]::text
    ,('{"phone":"+9' || (1000000000::bigint + floor(random()*9000000000)::bigint)::text || '"}')::jsonb
    , ceil(random()*100));
```

```
/usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload2.sql -p 5432 thai
```

Результат 
![alt text](image.png)

### 2 Проверим производительность single instance на чтение

Запрос (также из лекции)
```
\set r random(1, 5000000) 
SELECT id, fkRide, fio, contact, fkSeat FROM book.tickets WHERE id = :r;
```

```
/usr/lib/postgresql/17/bin/pgbench -c 8 -j 4 -T 10 -f ~/workload.sql -p 5432 thai
```
![alt text](image-1.png)

### 3 Настроим primary кластер для репликации 

## 1 Запускаем второй кластер на той же ВМ на порту 7432
![alt text](image-2.png)
![alt text](image-3.png)

## 2 Проверим необходимые настройки в postgresql.conf
Все настройки оставил дефолтными, так как они подходят для выполнения учебного задания. 
![alt text](image-4.png)
![alt text](image-5.png)
![alt text](image-6.png)

## 3 Создадим пользователя для репликации 
```
CREATE USER replica WITH REPLICATION ENCRYPTED PASSWORD 'replica123';
```
![alt text](image-7.png)
## 4 Явно создадим слот репликации 
```
SELECT pg_create_physical_replication_slot('replica_test');
```
![alt text](image-8.png)

## 5 Добавим в pg_hba.conf информацию о пользователе, подготовленном для репликации

Оказалось, что pg_hba.conf уже автоматом добавлены все пользователи для репликации с localhost 

![alt text](image-9.png)

## 6 Остановим реплику 

![alt text](image-10.png)

## 7 Очистим директорию реплики 

```
sudo rm -rf /var/lib/postgresql/17/rep

```
![alt text](image-11.png)

## 8 Запускаем pg_basebackup

```
pg_basebackup -h localhost -p 5432 -U replica -R -S replica_test -D /var/lib/postgresql/17/rep
```

![alt text](image-12.png)

## 9 Запускаем реплику 
Проверяем, что репликация успешно работает
![alt text](image-13.png)
Видим, что ```sync_state``` = ```async```

![alt text](image-14.png)

![alt text](image-15.png)

![alt text](image-16.png)

## 10 Проводим тесты на запись 

![alt text](image-17.png)

Видно падение на 7000 tps

## 11 Тест на чтение с мастера 

![alt text](image-18.png)

Не уступает чтению с сингл инстанса (что логично, при условии отстутсвия какой либо другой значимой нагрузки на мастер в этот момент)

## 12 Тест на чтение с реплики 

![alt text](image-19.png)

Получили даже немного больше, но в целом +- тоже самое 

### Вывод 
При создании реплики и использовании асинхронной физической репликации достаточно ощутимо упала скорость записи, скорость чтения же осталась такой же, а с реплики даже получилась чуть больше.