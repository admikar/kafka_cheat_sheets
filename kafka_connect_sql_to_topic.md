# Kafka connect sql totopic
Когда говорят, что Kafka Connect поставляется вместе с Kafka, это не какая-то скрытая функциональность Kafka-брокеров. Это именно отдельное приложение, которое имеет настройки подключения к Kafka и источнику/приёмнику.

Но сначала нужно ввести три важных термина:
 - worker — инстанс/сервер Kafka Connect;
 - connector — Java class + пользовательские настройки;
 - task — один процесс connector'a.

**Worker** — экземпляр Kafka Connect. Kafka Connect можно запускать в двух режимах: standalone и distributed, на нескольких нодах или виртуальных машинах. То есть можно просто запустить один worker или собрать кластер worker’ов. Рекомендуется использовать standalone-режим при локальной разработке, настройке и отладке коннекторов, а distributed — в боевых условиях.

### Преимущество distributed mode

Предположим, мы запустили четыре worker'а Kafka Connect и создали три connector'а с разным количеством task'ов.

 - Во-первых, Kafka Connect автоматически распределит таски коннекторов по разным воркерам.
 - Во-вторых, Kafka Connect отслеживает своё состояние в кластере. Если обнаружит, что один из воркеров недоступен, выполнит перебалансировку и перераспределит недоступные таски по работающим воркерам.

Какие ещё задачи решает Kafka Connect:

 - отказоустойчивость (fault tolerance);
 - принцип «только один раз» (exactly once);
 - распределение (distribution);
 - упорядочивание (ordering).

Фреймворк используется для передачи данных из источника в Kafka либо из Kafka в приёмник. В соответствии с этим коннекторы делятся на два вида:

 - Source Connectors;
 - Sink Connectors.

Вы можете написать свой коннектор на Java и Scala. Для этого нужно создать подключаемый jar-файл, реализовав простой интерфейс коннектора.

### Как запустить Kafka Connect
**Локально**
Например, выберем версию Scala 2.12 (kafka_2.12-2.6.0.tgz). Распакуем архив и посмотрим в директорию kafka_2.12-2.6.0/bin. Там будут скрипты для запуска Apache Kafka (kafka-server-start.sh, kafka-server-stop.sh) и утилиты для работы с ней. Например, kafka-console-consumer.sh, kafka-console-producer.sh. А также там будут скрипты для запуска Kafka Connect (connect-distributed.sh, connect-standalone.sh), и многое другое.

Рекомендую зайти в директорию kafka_2.12-2.6.0/config — там вы увидите настройки по умолчанию запуска и Kafka-брокера, и Kafka Connect.

 - connect-distributed.properties
 - connect-standalone.properties

Вот так выглядит конфигурация по умолчанию config/connect-distributed.properties:
```sh
bootstrap.servers=localhost:9092
rest.port=8083
group.id=connect-cluster
key.converter=org.apache.kafka.connect.json.JsonConverter
value.converter=org.apache.kafka.connect.json.JsonConverter
key.converter.schemas.enable=true
value.converter.schemas.enable=true
internal.key.converter=org.apache.kafka.connect.json.JsonConverter
internal.value.converter=org.apache.kafka.connect.json.JsonConverter
internal.key.converter.schemas.enable=false
internal.value.converter.schemas.enable=false
offset.storage.topic=connect-offsets
config.storage.topic=connect-configs
status.storage.topic=connect-status
offset.flush.interval.ms=10000
plugin.path=/opt/kafka/plugins
```

Чтобы запустить Kafka Connect, выполните команду:
```sh
cd kafka_2.12-2.6.0
bin/connect-standalone.sh config/connect-standalone.properties
```

**Docker**

Во многих Docker-образах используется этот же подход, поэтому вам достаточно переопределить CMD в Dockerfile, чтобы получить образ с Kafka Connect.

Например:
```sh
CMD ["bin/connect-distributed.sh", "cfg/connect-distributed.properties"]
```
базовый образ: https://hub.docker.com/r/confluentinc/cp-kafka-connect-base
образ с установленными коннекторами: https://hub.docker.com/r/confluentinc/cp-kafka-connect

### Запуск коннекторов
После того, как вы запустите Kafka Connect, вы можете запускать на нём свои коннекторы.

Для управления Kafka Connect используется REST API. Полную документацию по нему можно посмотреть на сайте. Я опишу лишь те методы, которые нам понадобятся для демонстрации работы Kafka Connect.

Запросим список классов коннекторов, которые добавлены в ваш Kafka Connect:
```sh
curl -X GET "${KAFKA_CONNECT_HOST}:8083/connector-plugins" -H "Content-Type: application/json"
```
То есть вы можете создавать коннекторы только этих классов. Если хотите добавить новый класс, нужно скачать jar этого коннектора и добавить в директорию plugin.path из настройки Kafka Connect. См. файл connect-distributed.properties.

**Запросим список запущенных коннекторов:**
```sh
curl -X GET "${KAFKA_CONNECT_HOST}/connectors" -H "Content-Type: application/json"
```
**Получение информации о запущенном коннекторе**
Общая информация:
```sh
curl -X GET "${KAFKA_CONNECT_HOST}/connectors/my-sink-jdbc" -H "Content-Type: application/json"
```
Конфигурация запущенного коннектора (config):
```sh
curl -X GET "${KAFKA_CONNECT_HOST}/connectors/my-sink-jdbc/config" -H "Content-Type: application/json"
```
Состояние запущенного коннектора (status):
```sh
curl -X GET "${KAFKA_CONNECT_HOST}/connectors/my-sink-jdbc/status" -H "Content-Type: application/json"
```

**Создание коннектора**
```sh
curl -X POST "${KAFKA_CONNECT_HOST}/connectors" -H "Content-Type: application/json" -d '{ \
    "name": "my-new-connector", \
    "config": { \
      "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector", \
      "tasks.max": 1,
      "topics": "mysql-table01,mysql-table02", \
      "connection.url": "jdbc:postgresql://postgres:5432/catalog", \
      "connection.user": "postgres", \
      "connection.password": "postgres", \
      "auto.create": "true" \
    } \
  }'
```

То есть необходимо методом POST отправить конфигурацию коннектора.

Обратите внимание, что имя коннектора должно быть уникальным в вашем кластере Kafka Connect. Но вы можете создавать несколько коннекторов одного класса с разными настройками.

Также у любого коннектора есть три обязательных параметра:

 - name — уникальное имя;
 - connector.class — класс коннектора;
 - tasks.max — максимальное количество потоков, в которых может работать коннектор.

###Настройка коннекторов
**Jdbc и Debezium**

Когда ищешь коннекторы для баз данных, первое, что находишь — JdbcSourceConnector и JdbcSinkConnector.

Нам отлично подходит JdbcSinkConnector в качестве sink-коннектора. Он подписывается на топик Kafka и выполняет запросы на добавление, изменение и удаление данных в базе.

Но в качестве Source-коннектора он нам не подходит, так как делает SQL-запросы в базу по таймеру, а это создает ещё бо̒льшую нагрузку на базу-источник. Мы как раз хотим от этого уйти.

Но нам подходит DebeziumMysqlConnector. Он делает одну классную вещь: подключается к MySQL-кластеру как обычная MySQL-реплика и умеет читать бинлог. Таким образом, мы не создаём дополнительную нагрузку на базу (за исключением встроенных механизмов MySQL-репликации).

Помимо этого, у Debezium-коннектора есть ещё одно преимущество перед Jdbc. Так как Debezium отслеживает бинлог, он может определять моменты удаления записей в базе данных. У Jdbc нет такой возможности, так как он берёт текущее состояние базы и ничего не знает о предыдущем состоянии.

**Debezium Connector**
Все настройки коннектора можно посмотреть на https://debezium.io/documentation/reference/1.2/connectors/mysql.html

Давайте рассмотрим настройки коннектора и обсудим выбор некоторых параметров.

Файл debezium-config.json:
```sh
{
  "name": "my-debezium-mysql-connector",
  "config": {
    "tasks.msx": 1,
    "connector.class": "io.debezium.connector.mysql.MySqlConnector",
    "database.hostname": "${MYSQL_HOST}",
    "database.serverTimezone": "Europe/Moscow",
    "database.port": "${MYSQL_PORT}",
    "database.user": "${MYSQL_USER}",
    "database.password": "${MYSQL_PASS}",
    "database.server.id": "223355",
    "database.server.name": "monolyth_db",
    "table.whitelist": "${MYSQL_DB}.table_name1",
    "database.history.kafka.bootstrap.servers": "${KAFKA_BROKER}",
    "database.history.kafka.topic": "monolyth_db.debezium.history",
    "database.history.skip.unparseable.ddl": true,
    "snapshot.mode": "initial",
    "time.precision.mode": "connect"
  }
}
```
Следует иметь в виду, что этот пользователь должен иметь права:
```sh
GRANT SELECT, RELOAD, SHOW DATABASES, REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO 'user' IDENTIFIED BY 'password';
```
ID реплики, под которым будет зарегистрирован коннектор, и его имя сервера:
```sh
    "database.server.id": "223355",
    "database.server.name": "monolyth_db",
```
Список таблиц для синхронизации:
```sh
"table.whitelist": "${MYSQL_DB}.table_name1,${MYSQL_DB}.table_name2",
```
Имена базы и таблицы необходимо указывать через запятую.
Настройки создания snapshot'а:
```sh
    "database.history.kafka.bootstrap.servers": "${KAFKA_BROKER}",
    "database.history.kafka.topic": "debezium.db.history",
    "snapshot.mode": "initial",
```
**Для чего нужен snapshot**
Когда ваш коннектор Debezium MySQL запускается в первый раз, он выполняет начальный согласованный снимок вашей базы данных и сохраняет его в топик Kafka. Даже если вы будете отслеживать только несколько таблиц из базы, в database.history будет записана вся схема. Но можно не переживать из-за размера этого топика, он будет очень маленьким (менее 1 Мб).

Пропуск определений в снимке, которые по каким-то причинам не удалось распарсить:
```sh
    "database.history.skip.unparseable.ddl": true,
```
Эту опцию мы включили, потому что сталкивались с такими ошибками, когда определения в бинлоге использовали неверный синтаксис. Сервер MySQL более-менее интерпретирует эти инструкции и потому не падает. Но анализатор SQL-запросов в DebeziumConnector'е с ними не справляется и падает с ошибкой. Чтобы не падать, а игнорировать нечитаемые запросы, необходимо включить эту опцию.

Точность типа данных time:
```sh
	"time.precision.mode": "connect",
```
Эта настройка уменьшает точность типа данных time с микросекунд до миллисекунд.

Описанную конфигурацию уже можно использовать для production-окружения. А в документации есть полный перечень настроек с подробным описанием.

Также нашу конфигурацию можно дополнить различными трансформерами по преобразованию данных и маршрута топиков. И один из важнейших трансформеров в проекте Debezium — io.debezium.transforms.ExtractNewRecordState. Почитать подробнее о нём можно в документации. Если кратко: вам потребуется его использовать для преобразования формата Debezium в формат Jdbc.

В целом, все трансформации рекомендуется использовать на стороне Sink-коннектора, а Source-коннекторы должны отправлять данные в топик Kafka без изменений.

**Создание Debezium MySqlConnector:**
```sh
curl  -X POST ${KAFKA_CONNECT_HOST}/connectors -H "Content-Type: application/json" -d @debezium-config.json
```

При создании коннектора вы можете получить ошибку:

Connector configuration is invalid and contains the following 1 error(s):
Configuration is not defined: database.history.connector.id
Configuration is not defined: database.history.connector.class
Unable to connect: Communications link failure

The last packet sent successfully to the server was 0 milliseconds ago. The driver has not received any packets from the server.
You can also find the above list of errors at the endpoint `/connector-plugins/{connectorType}/config/validate````

Эта ошибка говорит о том, что у вас указаны некорректные параметры подключения к MySQL. Проверьте ваши логин/пароль, а также убедитесь, что у пользователя есть права на репликацию (см. выше).

После создания коннектора можно проверить его состояние. Этот метод также используется в качестве метрики.
```sh
curl  -X GET ${KAFKA_CONNECT_HOST}/connectors/my-debezium-mysql-connector/status
```
После того, как мы запустили source connector, можно убедиться, что топики были созданы и можно прочитать из них данные. Для работы с Kafka будем использовать удобную утилиту kafkacat.
Какие топики были созданы нашим коннектором:
```sh
kafkacat -b ${KAFKA_BROKER} -L | grep 'monolyth_db'
```
Чтение данных из топика monolyth_db.debezium.history:
```sh
kafkacat -b ${KAFKA_BROKER} -t monolyth_db.debezium.history -C -f 'Offset: %o\nKey: %k\nPayload: %s\n--\n'
```
Чтение данных из топика monolyth_db.table_name1 (${MYSQL_DB} — имя вашей базы данных):
```sh
kafkacat -b ${KAFKA_BROKER} -t monolyth_db.${MYSQL_DB}.table_name1 -C -f 'Offset: %o\nKey: %k\nPayload: %s\n--\n'
```
В топиках вы увидите сообщения в формате avro (если вы использовали JsonSerializer для key, value серилизаторов). Вид и описание формата лучше прочитать в https://debezium.io/documentation/reference/1.2/connectors/mysql.html#mysql-connector-events_debezium

###JdbcSinkConnector
В качестве Sink коннектора будем использовать JdbcSinkConnector.
рассмотрим его конфигурацию
Создадим файл my-jdbc-sink-connector.json:
```sh
{
  "name": "my-jdbc-sink-connector",
  "config": {
    "connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
    "tasks.max": "2",
    "connection.url": "jdbc:postgresql://${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}",
    "connection.user": "${POSTGRES_USER}",
    "connection.password": "${POSTGRES_PASS}",
    "topics": "monolyth_db.${MYSQL_DB}.table_name1,monolyth_db.${MYSQL_DB}.table_name2",
    "pk.fields": "id",
    "pk.mode": "record_key",
    "auto.create": "false",
    "auto.evolve": "false",
    "insert.mode": "upsert",
    "delete.enabled": "true",
    "transforms": "route,unwrap,rename_field,ts_updated_at,only_fields",
    "transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
    "transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
    "transforms.route.replacement": "${PG_DB}.public.$3",
    "transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
    "transforms.unwrap.drop.tombstones": "false",
    "transforms.rename_field.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.rename_field.renames": "isDeleted:is_deleted,isActive:is_active",
    "transforms.ts_updated_at.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
    "transforms.ts_updated_at.target.type": "Timestamp",
    "transforms.ts_updated_at.field": "updated_at",
    "transforms.ts_updated_at.format": "yyyy-MM-dd'T'HH:mm:ssXXX",
    "transforms.only_fields.type": "org.apache.kafka.connect.transforms.ReplaceField$Value",
    "transforms.only_fields.whitelist": "id,title,url_tag,sort,hide,created_at,updated_at"
  }
}
```
Тут, конечно, три обязательных для любого коннектора параметра:
```sh
"name": "my-jdbc-sink-connector",
"connector.class": "io.confluent.connect.jdbc.JdbcSinkConnector",
"tasks.max": "2",
```
Настройки подключения:
```sh
"connection.url": "jdbc:postgresql://${POSTGRES_HOST}:${POSTGRES_PORT}/${POSTGRES_DB}",
"connection.user": "${POSTGRES_USER}",
"connection.password": "${POSTGRES_PASS}",
```
Потом перечисление топиков, на которые будем подписываться:
```sh
"topics": "monolyth_db.${MYSQL_DB}.table_name1,monolyth_db.${MYSQL_DB}.table_name2",
```
JdbcConnector использует один топик для одной таблицы. Сопоставление топика и таблицы происходит по имени. Для коррекции используется route-трансформер. О трансформерах поговорим чуть ниже.

Если вы указываете несколько топиков, то у них у всех должны быть одинаковые pk.fields.

Сообщения в Kafka имеют ключ, создаваемый на основании первичного ключа (Primary Key) таблицы. Какой именно PR в таблице, необходимо указать в параметрах pk.fields, чаще всего это просто id:
```sh
"pk.fields": "id",
"pk.mode": "record_key",
```
Ключ может быть составной. Например, для кросс-таблиц:
```sh
"pk.fields": "user_id,service_id",
```
Следующие параметры очень красноречивые. Мы отключаем автоматическое создание и удаление таблиц, и разрешаем удалять данные:
```sh
"auto.create": "false",
"auto.evolve": "false",
"insert.mode": "upsert",
"delete.enabled": "true",
```
**Трансформеры**

Последний блок настроек касается трансформеров.
```sh
"transforms": "route, unwrap, rename_field, ts_updated_at, only_fields",
```
Этот параметр указывает, какие трансформеры и в каком порядке выполнять. Они расположены в этой же конфигурации коннектора. Каждый трансформер имеет type (класс) и параметры.

Например, трансформер route отвечает за сопоставление имени топика и имени таблицы:
```sh
"transforms.route.type": "org.apache.kafka.connect.transforms.RegexRouter",
"transforms.route.regex": "([^.]+)\\.([^.]+)\\.([^.]+)",
"transforms.route.replacement": "${PG_DB}.public.$3",
```
Он используется в Debezium MySqlConnector: отправляет данные в Kafka топики с именами {server_name}.{database_name}.{table_name}, а JdbcSinkConnector принимает {database_name}.{schema_name}.{table_name}. Так как целевая база и таблица могут отличаться по именам (и у вас вряд ли имя базы будет public), то этот коннектор изменяет целевое имя топика.

Второй важный трансформер unwrap:
```sh
"transforms.unwrap.type": "io.debezium.transforms.ExtractNewRecordState",
"transforms.unwrap.drop.tombstones": "false",
```
Он преобразует формат Debezium в формат, с которым прекрасно работает JdbcSinkConnector.

Трансформеры rename_field, ts_updated_at и only_fields используются для переименования полей, преобразования дат и указания списка тех полей, которые необходимо синхронизировать. Так указывается конфигурация трансформера ts_updated_at:
```sh
"transforms.ts_updated_at.type": "org.apache.kafka.connect.transforms.TimestampConverter$Value",
"transforms.ts_updated_at.target.type": "Timestamp", 
"transforms.ts_updated_at.field": "updated_at", 
"transforms.ts_updated_at.format": "yyyy-MM-dd'T'HH:mm:ssXXX",
```

**Отмечу только, что 2 Гб памяти поду под Kafka Connect не хватает, и у нас поды по 4 Гб.**
