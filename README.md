# Настройка шардированного кластера MongoDB

## Описание
Этот проект представляет собой шардированный кластер MongoDB, состоящий из:
- 3 конфигурационных серверов
- 1 шарда, состоящего из 3 реплик
- 1 маршрутизатора `mongos`

## Запуск кластера
1. Запустить контейнеры:
   ```sh
   docker-compose up -d
   ```
2. Инициализировать репликационные наборы.

### Инициализация конфигурационных серверов
Подключиться к одному из них:
```sh
docker exec -it configdb1 mongosh
```

Выполнить:
```js
rs.initiate({
  _id: "cfgreplset",
  configsvr: true,
  members: [
    { _id: 0, host: "configdb1:27017" },
    { _id: 1, host: "configdb2:27017" },
    { _id: 2, host: "configdb3:27017" },
  ],
});
```

### Инициализация шарда
Подключиться к первому узлу шарда и выполнить:
```sh
docker exec -it shard1-repl1 mongosh
```
```js
rs.initiate({
  _id: "shard1",
  members: [
    { _id: 0, host: "shard1-repl1:27017" },
    { _id: 1, host: "shard1-repl2:27017" },
    { _id: 2, host: "shard1-repl3:27017" },
  ],
});
```

### Добавление шарда в `mongos`
Подключиться к `mongos`:
```sh
docker exec -it mongos mongosh
```
```js
sh.addShard("shard1/shard1-repl1:27017");
sh.status()
```

## Выбор ключа шардирования и балансировка
Импорт базы данных и создание коллекции с шардированием:
```sh
docker cp ~/MongoDB-RS/airbnb_data.json config1:/airbnb_data.json
```
```sh
docker exec -it config1 mongoimport --db airbnb --collection booking --file /airbnb_data.json
```
```js
use airbnb
sh.enableSharding("airbnb")
sh.shardCollection("airbnb.booking", { host_id: "hashed" })
```

## Нагрузочное тестирование и отказоустойчивость
- Добавить данные в коллекцию и проверить распределение данных с помощью `sh.status()`.
- Остановить один из узлов (`docker stop shard1_1`) и проверить доступность данных.
- Поднять его обратно (`docker start shard1_1`) и посмотреть на восстановление репликации.

## Настройка аутентификации и ролей
1. Включить авторизацию, добавив в `docker-compose.yml` опцию `--auth`.
2. Создать администратора:
```js
use admin
 db.createUser({
   user: "admin",
   pwd: "password",
   roles: [ { role: "root", db: "admin" } ]
 })
```
3. Перезапустить контейнеры и подключиться с аутентификацией.

### Настройка многоролевого доступа
Создать пользователей с разными ролями:
```js
use admin

// Создание пользователя для чтения
 db.createUser({
   user: "readUser",
   pwd: "readPass",
   roles: [ { role: "read", db: "airbnb" } ]
 })

// Создание пользователя с доступом на запись
 db.createUser({
   user: "writeUser",
   pwd: "writePass",
   roles: [ { role: "readWrite", db: "airbnb" } ]
 })

// Создание пользователя-администратора базы данных
 db.createUser({
   user: "dbAdmin",
   pwd: "adminPass",
   roles: [ { role: "dbAdmin", db: "airbnb" } ]
 })

// Создание пользователя с доступом к мониторингу
 db.createUser({
   user: "monitorUser",
   pwd: "monitorPass",
   roles: [ { role: "clusterMonitor", db: "admin" } ]
 })
```
Теперь пользователи могут подключаться с соответствующими правами.

## Результаты

* Шардированный кластер MongoDB с репликацией успешно настроен.
* Балансировка данных между шардами работает.
* Кластер устойчив к отказам.
* Аутентификация и многоролевой доступ настроены.

## Проблемы

* Возможно, потребуется настройка брандмауэра для доступа к портам MongoDB.
* При большом количестве данных балансировка может занять продолжительное время.
* Сложность в настройке аутентификации

