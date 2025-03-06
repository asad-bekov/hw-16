# Домашнее задание к занятию «Репликация и масштабирование. Часть 1»
*Асадбеков Асадбек*

## Задание 1

**На лекции рассматривались режимы репликации master-slave, master-master, опишите их различия.**

*Ответить в свободной форме*.

### Master-Slave Репликация

**Преимущества:**
- Простота настройки
- Масштабируемость операций чтения
- Изоляция операций записи и чтения

**Недостатки:**
- Единая точка отказа
- Задержка репликации
- Ограниченная масштабируемость записи

### Master-Master Репликация

**Преимущества:**
- Высокая доступность
- Балансировка нагрузки
- Гибкость при сбоях

**Недостатки:**
- Сложность настройки
- Риск конфликтов
- Сложность администрирования

## Задание 2

**Выполните конфигурацию master-slave репликации, примером можно пользоваться из лекции.**

*Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.*

**Docker-Compose файл**

```yaml
version: '3.1'

services:
  mysql-master:
    image: mysql:8.0
    container_name: mysql-master
    environment:
      - MYSQL_ROOT_PASSWORD=root_password
      - MYSQL_DATABASE=replication_db
    ports:
      - "3306:3306"
    volumes:
      - ./master-data:/var/lib/mysql
      - ./master.cnf:/etc/mysql/conf.d/master.cnf

  mysql-slave:
    image: mysql:8.0
    container_name: mysql-slave
    environment:
      - MYSQL_ROOT_PASSWORD=root_password
    ports:
      - "3307:3306"
    volumes:
      - ./slave-data:/var/lib/mysql
      - ./slave.cnf:/etc/mysql/conf.d/slave.cnf
```

**Конфигурационные файлы MySQL**

**Файл `master.cnf` (настроен для мастера):**

```ini
[mysqld]
server-id=1
log-bin=mysql-bin
binlog_do_db=replication_db
```

**Файл `slave.cnf` (настроен для слейва):**

```ini
[mysqld]
server-id=2
relay-log=mysqld-relay-bin
replicate_do_db=replication_db
```
![alt text](https://github.com/asad-bekov/hw-16/blob/main/img/1.png)

**Создание пользователя репликации на мастере**

Подключение к контейнеру мастера:

```bash
docker exec -it mysql-master mysql -uroot -p
```

SQL-команды:

```sql
CREATE USER 'repl'@'%' IDENTIFIED WITH mysql_native_password BY 'repl_password';
GRANT REPLICATION SLAVE ON *.* TO 'repl'@'%';
FLUSH PRIVILEGES;
SHOW MASTER STATUS;
```

**Настройка слейва**

Подключение к контейнеру слейва:

```bash
docker exec -it mysql-slave mysql -uroot -p
```

Команда для подключения к мастеру:

```sql
CHANGE MASTER TO
  MASTER_HOST='mysql-master',
  MASTER_USER='repl',
  MASTER_PASSWORD='repl_password',
  MASTER_LOG_FILE='mysql-bin.000003',
  MASTER_LOG_POS=994;
START SLAVE;
```

Проверка статуса репликации:

```sql
SHOW SLAVE STATUS\G
```
![alt text](https://github.com/asad-bekov/hw-16/blob/main/img/2.png)

**Тестирование работоспособности на мастере**

```sql
USE replication_db;
CREATE TABLE test (id INT AUTO_INCREMENT PRIMARY KEY, message VARCHAR(255));
INSERT INTO test (message) VALUES ('message by Asad');
```

**Проверка наличие данных на слейве**

```sql
USE replication_db;
SELECT * FROM test;
```
![alt text](https://github.com/asad-bekov/hw-16/blob/main/img/3.png)

## Задание 3

**Выполните конфигурацию master-master репликации. Произведите проверку.**

*Приложите скриншоты конфигурации, выполнения работы: состояния и режимы работы серверов.*

**Docker-Compose файл**

```yaml
version: '3.1'

services:
  mysql-node1:
    image: mysql:8.0
    container_name: mysql-node1
    environment:
      - MYSQL_ROOT_PASSWORD=root_password
      - MYSQL_DATABASE=replication_db
    ports:
      - "3306:3306"
    volumes:
      - ./node1-data:/var/lib/mysql
      - ./node1.cnf:/etc/mysql/conf.d/node1.cnf

  mysql-node2:
    image: mysql:8.0
    container_name: mysql-node2
    environment:
      - MYSQL_ROOT_PASSWORD=root_password
      - MYSQL_DATABASE=replication_db
    ports:
      - "3307:3306"
    volumes:
      - ./node2-data:/var/lib/mysql
      - ./node2.cnf:/etc/mysql/conf.d/node2.cnf
```

**Конфигурационные файлы для узлов**

**Файл `node1.cnf`**

```ini
[mysqld]
server-id=1
log-bin=mysql-bin
binlog_do_db=replication_db
auto_increment_increment=2
auto_increment_offset=1
```

**Файл `node2.cnf`**

```ini
[mysqld]
server-id=2
log-bin=mysql-bin
binlog_do_db=replication_db
auto_increment_increment=2
auto_increment_offset=2
```
![alt text](https://github.com/asad-bekov/hw-16/blob/main/img/4.png)

![alt text](https://github.com/asad-bekov/hw-16/blob/main/img/5.png)

**Тестирование**

На `mysql-node1`:

```sql
USE replication_db;
INSERT INTO test (message) VALUES ('sent from node1');
```

На `mysql-node2`:

```sql
USE replication_db;
SELECT * FROM test;
```
![alt text](https://github.com/asad-bekov/hw-16/blob/main/img/6.png)


