# Перенос кластера Patroni/PostgreSQL 16 в новый датацентр

## Назначение документа

Инструкция описывает порядок переноса production-кластера Patroni (PostgreSQL 16) в
новый датацентр по схеме «standby cluster → promote → обратный standby». Метод
инициализации нового кластера — `pg_basebackup`. Слоты логической репликации
пересоздаются вручную.

## Исходные данные

| Параметр | Значение |
|---|---|
| Текущий кластер | `pgdf` (DF): Leader + 2 async replica |
| Целевой кластер | `pgbj` (BJ): Leader + 1 sync replica + 2 async replica |
| DCS | etcd v3, отдельный кластер в каждом ДЦ |
| Логическая репликация | Слоты созданы вручную на одной реплике DF через `psql ... replication=database`. На Leader слотов нет |
| Состояние pgbj | Развёрнут Ansible с собственным `initdb`. `system_identifier` отличается от prod, требуется повторная инициализация PGDATA |

Узел DF, на котором размещены логические слоты, далее обозначается `df-slots`.

## Условные обозначения

```bash
export PATRONICTL_CONFIG_FILE=/etc/patroni/patroni.yml
export ETCDCTL_API=3
export OLD=pgdf NEW=pgbj
```

Переменная `PATRONICTL_CONFIG_FILE` позволяет вызывать `patronictl` без параметра
`-c`. При её отсутствии путь к конфигурации указывается явно:
`patronictl -c /etc/patroni/patroni.yml <команда>`.

---

## План работ

| Этап | Шаг | Операция | Простой |
|---|---|---|---|
| Подготовка | 1 | Определить узел со слотами, включить `hot_standby_feedback` | нет |
| Подготовка | 2 | Создать физический слот `pgbj_standby` на prod | нет |
| Подготовка | 3 | Повторно инициализировать PGDATA pgbj как standby cluster | нет |
| Подготовка | 4 | Определить размещение слотов на pgbj | нет |
| Подготовка | — | Наблюдение за работой standby, несколько суток | нет |
| Окно | 5 | Остановить нагрузку на prod | начало |
| Окно | 6 | Дождаться вычитывания логическими подписчиками | да |
| Окно | 7 | Зафиксировать позиции подписчиков, отключить подписки | да |
| Окно | 7.1 | Остановить DF штатным способом | да |
| Окно | 7.2 | Дождаться применения WAL на BJ до shutdown checkpoint | да |
| Окно | 8.1 | Выполнить promote pgbj | да |
| Окно | 8.2–8.3 | Создать логические слоты | да |
| Окно | 8.4 | Сверить LSN слотов с позициями подписчиков | да |
| Окно | 8.5 | Переключить подписки на pgbj | да |
| Окно | 8.6 | Включить синхронный режим | да |
| Окно | 9 | Переключить трафик, запустить приложения | окончание |
| Завершение | — | Период наблюдения | нет |
| Завершение | 10 | Перевести DF в standby нового кластера | нет |

Контрольные точки, требующие обязательной проверки перед переходом к следующему шагу:

- **7.2** — применение WAL на BJ до shutdown checkpoint DF. При невыполнении
  возникает расхождение таймлиний, и шаг 10 потребует `pg_rewind`.
- **8.4** — соответствие LSN созданных слотов позициям подписчиков. При невыполнении
  часть изменений не будет передана подписчикам.

---

## Необратимые операции

В процедуре шесть операций, после выполнения которых возврат к предыдущему состоянию
либо требует дополнительных затрат, либо невозможен. Операции Н4 и Н5 связаны с
потерей данных.

| ID | Шаг | Что становится необратимым | Потеря данных | Контрольная мера |
|---|---|---|---|---|
| Н1 | 3 | PGDATA узлов pgbj удалён, кластер удалён из DCS | Для production отсутствует. При ошибочном указании узлов DF — потеря production-данных | Явный перечень узлов, сверка `system_identifier` до удаления |
| Н2 | 7.1 | Завершается период, в котором DF является источником данных | Отсутствует при штатной остановке | `pg_controldata` в состоянии `shut down`, достаточное значение `TimeoutStopSec` |
| Н3 | 8.1 | Точка расхождения таймлиний фиксируется окончательно | Отсутствует до появления клиентской записи на BJ | Приложения не запускаются до шага 8.5, контроль `xact_commit` на BJ |
| Н4 | 8.2–8.3 | LSN созданного слота не может быть уменьшен | Изменения в интервале расхождения не передаются подписчику, диагностические сообщения отсутствуют | Соблюдение порядка 6 → 7 → 8.1 → 8.2, проверка `pg_waldump` |
| Н5 | 9 | На BJ появляются данные, отсутствующие на DF | При возврате — весь объём записи на BJ с момента `T_switch` | Установленный интервал принятия решения о возврате |
| Н6 | 10 | Возврат на DF становится невозможен | Отсутствует. Удаляется последняя копия состояния на момент остановки DF | Период наблюдения, холодная копия PGDATA до выполнения шага |

### Н1. Удаление PGDATA узлов pgbj (шаг 3)

Необратимая часть шага 3:

```bash
patronictl remove $NEW
for h in bj-pg1 bj-pg2 bj-pg3 bj-pg4; do
  ssh $h "sudo -u postgres find /var/lib/postgresql/16/main -mindepth 1 -delete"
done
```

Контрольная мера, выполняется до удаления:

```bash
for h in bj-pg1 bj-pg2 bj-pg3 bj-pg4; do
  ssh $h "hostname; sudo -u postgres /usr/lib/postgresql/16/bin/pg_controldata \
    -D /var/lib/postgresql/16/main | grep 'system identifier'"
done
# ни один из идентификаторов не должен совпадать с идентификатором prod
```

Для pgbj операция не связана с потерей значимых данных: кластер развёрнут Ansible,
production-данные отсутствуют, PGDATA будет создан заново клонированием с prod.

Контрольная мера относится к риску ошибочного указания узлов. Имена узлов DF и BJ
различаются незначительно, при этом операция выполняется с правами суперпользователя
и необратима. Совпадение `system_identifier` с prod означает, что команда выполняется
на узлах DF. Узлы перечисляются явным списком, без использования групп инвентаря и
шаблонов.

### Н2. Остановка DF (шаг 7.1)

Необратимая часть:

```bash
for h in df-pg3 df-pg2 df-pg1; do ssh $h systemctl stop patroni; done
```

Контроль результата и запаса по таймауту:

```bash
ssh df-pg1 "sudo -u postgres /usr/lib/postgresql/16/bin/pg_controldata \
  -D /var/lib/postgresql/16/main | grep 'cluster state'"
# ожидаемое значение: Database cluster state: shut down

systemctl show patroni -p TimeoutStopUSec
# значение должно превышать типовое время выполнения shutdown checkpoint
```

Формально операция обратима: кластер может быть запущен повторно, он остаётся на
своей таймлинии и сохраняет роль primary. Вместе с тем после остановки DF перестаёт
быть источником данных, а логическая репликация для подписчиков приостанавливается.

Обоснование требования к штатной остановке. При штатном завершении процесс
checkpointer записывает shutdown checkpoint, после чего процессы walsender передают
остаток WAL и завершаются. Эта запись становится последней в таймлинии. Новый кластер
применяет её, и при promote следующая таймлиния начинается с LSN, следующего за ней.
Истории кластеров не перекрываются, расхождение не возникает, и на шаге 10 узлы
переключаются на новую таймлинию без `pg_rewind` — аналогично штатному
`patronictl switchover`.

При аварийном завершении процесса (`kill -9`, OOM-killer, значение
`TimeoutStopSec` меньше времени остановки) последней записью окажется произвольная
точка WAL. При следующем запуске PostgreSQL применит WAL и добавит собственные
записи, что приведёт к расхождению таймлиний и необходимости `pg_rewind` на шаге 10.
Потеря данных при этом не происходит ни в одном из вариантов.

### Н3. Promote pgbj (шаг 8.1)

Необратимая часть:

```bash
patronictl edit-config $NEW --force -s standby_cluster=null
```

Фиксация состояния непосредственно после promote:

```sql
-- на bj-pg1
SELECT pg_current_wal_lsn();     -- зафиксировать как promote_lsn, используется в Н5
SELECT xact_commit, tup_inserted, tup_updated, tup_deleted
FROM pg_stat_database WHERE datname='appdb';
-- счётчики не должны изменяться: клиентская запись отсутствует
```

Promote создаёт новую таймлинию и фиксирует точку расхождения в файле истории. Данный
LSN не может быть изменён впоследствии: все записи, выполненные любым из кластеров
после этой точки, существуют только в его собственной истории. Средства объединения
двух таймлиний в PostgreSQL отсутствуют.

До появления клиентской записи на BJ возврат выполняется без потери данных: DF
запускается, подписки возвращаются на него, BJ инициализируется повторно либо
приводится к состоянию DF через `pg_rewind`. Служебные записи после promote
(end-of-recovery, checkpoint, running_xacts) формируются всегда и на обратимость не
влияют.

### Н4. Создание логических слотов (шаги 8.2 и 8.3)

Необратимая часть:

```bash
psql "host=bj-pg1 port=5432 dbname=appdb user=replicator replication=database" \
  -c "CREATE_REPLICATION_SLOT sub_analytics_slot LOGICAL pgoutput (SNAPSHOT 'nothing');"
```

Контрольная мера, выполняется до `ALTER SUBSCRIPTION ... ENABLE`:

```bash
# на bj-pg1: remote_lsn — из шага 7, confirmed_flush_lsn — из шага 8.4
sudo -u postgres pg_waldump -p /var/lib/postgresql/16/main/pg_wal \
  -s <remote_lsn> -e <confirmed_flush_lsn> 2>/dev/null | grep -c "COMMIT"
# ожидаемое значение: 0
```

Значение больше нуля означает, что соответствующие транзакции не будут переданы
подписчику.

Логический слот создаётся на текущем LSN. Возможность создания слота на более ранней
позиции отсутствует, функция `pg_replication_slot_advance` перемещает слот только в
направлении увеличения LSN. Таким образом, позиция создания слота является нижней
границей передаваемых им изменений.

При выполнении `START_REPLICATION` подписчик указывает начальную позицию, равную
своему `remote_lsn`. Если эта позиция меньше `confirmed_flush_lsn` слота, PostgreSQL
не возвращает ошибку, а перемещает начальную позицию до `confirmed_flush_lsn`.
Соответствующее сообщение формируется на уровне DEBUG и в типовой конфигурации
логирования не отображается. Подписка остаётся в состоянии `active`, значение
`remote_lsn` увеличивается, метрики мониторинга соответствуют норме. Изменения в
интервале `[remote_lsn подписчика; confirmed_flush_lsn слота)` подписчику не
передаются.

Устранение расхождения после его возникновения. WAL с соответствующими изменениями
присутствует на BJ, поскольку был получен репликацией со старого кластера. Создать
слот, передающий изменения с более раннего LSN, невозможно. Уменьшение значения
`pg_replication_origin` подписчика средствами `pg_replication_origin_advance`
технически допустимо, однако позиция слота на publisher при этом не изменяется, и
начальная позиция будет повторно перемещена до `confirmed_flush_lsn`. Единственный
способ устранения — повторная начальная загрузка затронутых таблиц с
`copy_data = true` либо сверка и ручное внесение расхождений.

Этим определяется требование к порядку шагов: вычитывание подписчиками (6),
фиксация их позиций (7), promote (8.1), создание слотов (8.2–8.3), при отсутствии
клиентских транзакций на BJ до завершения шага 8.2. Каждая транзакция, выполненная
между promote и созданием слота, увеличивает интервал расхождения на свой объём.

### Н5. Переключение трафика (шаг 9)

Необратимая часть — переключение точки входа на `pg-prod.bj.example.com`
(DNS, HAProxy либо конфигурация приложений) и последующий запуск приложений
владельцами сервисов.

Оценка объёма данных, теряемых при возврате. Выполняется в любой момент после
`T_switch`:

```sql
-- на bj-pg1
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '<promote_lsn>'::pg_lsn))
         AS wal_written_after_switch;
SELECT xact_commit, tup_inserted, tup_updated, tup_deleted
FROM pg_stat_database WHERE datname='appdb';
```

После переключения трафика приложения выполняют запись в BJ, и эти изменения на DF
отсутствуют. Возврат на DF означает их полную потерю. Обратная логическая
репликация в данной схеме не применяется, так как подписка на DF требует
работающего primary, а DF остановлен.

Объём теряемых данных увеличивается пропорционально времени работы BJ. До выполнения
шага 10 DF сохраняется в виде согласованной копии на момент shutdown checkpoint,
и возврат выполняется запуском DF с переключением на него подписок и трафика.

Интервал принятия решения о возврате и критерии возврата (пороговые значения
error rate, latency, результаты функциональных проверок) определяются до начала
работ.

### Н6. Перевод DF в standby (шаг 10)

Контрольная мера, выполняется до запуска. DF остановлен штатно, копия каталога
является согласованной:

```bash
ssh df-pg1 "tar -C /var/lib/postgresql/16 -cf - main | zstd -3 \
  > /backup/df-pgdata-$(date +%F).tar.zst"
# альтернатива: снимки виртуальных машин всех узлов DF
```

Необратимая часть:

```bash
patronictl resume $OLD --wait
ssh df-pg1 systemctl start patroni
```

После выполнения операции DF начинает следовать новой таймлинии, его собственная
история становится недоступной, логические слоты на `df-slots` удаляются. При
выполнении `pg_rewind` дополнительно очищается каталог `pg_replslot`.

Потеря данных в результате самой операции не происходит. Операция удаляет копию
состояния на момент остановки DF, которая является контрольной мерой для Н5. По
этой причине шаг 10 выполняется отдельно от технологического окна, после периода
наблюдения.

### Соответствие момента обнаружения проблемы и объёма потерь

| Момент обнаружения | Потеря данных | Порядок действий |
|---|---|---|
| До шага 8.1 (шаги 5–7) | Отсутствует | Запустить DF, снять ограничения `pg_hba`, включить подписки |
| После 8.1, до 8.5, при отсутствии записи на BJ | Отсутствует | Запустить DF, вернуть подписки; BJ привести к состоянию DF через `pg_rewind` либо инициализировать повторно |
| Расхождение в слоте обнаружено после 8.5 | Изменения в интервале расхождения | Выполнить повторную начальную загрузку затронутых таблиц с `copy_data = true` либо сверить вручную |
| После `T_switch` | Весь объём записи на BJ с момента `T_switch` | Остановить приложения, создать резервную копию BJ и выгрузку расхождения, запустить DF, перенести расхождение вручную |
| После шага 10 | Возврат на DF невозможен | Восстановление из холодной копии, созданной в Н6 |

### Обратимые операции

Следующие операции отменяются обратным действием и данных не затрагивают:
изменение `pg_hba` с последующим `patronictl reload`, `patronictl pause` и `resume`,
`ALTER SUBSCRIPTION ... DISABLE` и `SET (slot_name = NONE)`, включение синхронного
режима, переключение DNS, удаление неиспользуемых слотов на `df-slots` после
переключения подписок.

---

## Подготовительный этап

Выполняется за 6 дней до технологического окна, простоя не требует.

### 1. Определить размещение логических слотов

```bash
for h in df-pg1 df-pg2 df-pg3; do
  echo "=== $h ==="
  psql -h $h -U postgres -d postgres -Atc "
    SELECT slot_name, plugin, database, active, restart_lsn, confirmed_flush_lsn,
           wal_status, conflicting
    FROM pg_replication_slots WHERE slot_type='logical';"
done
```

Слоты размещены не на Leader, поэтому опрашиваются все узлы кластера. Результат
зафиксировать в таблице: `slot_name`, `plugin`, `database`, узел, потребитель.

Узел со слотами не подлежит повторной инициализации до завершения миграции:
`patronictl reinit` удалит слоты.

Проверить работоспособность слотов:

```bash
# на df-slots
psql -U postgres -Atc "SHOW hot_standby_feedback;"           # ожидается on
# на Leader DF
psql -U postgres -Atc "SELECT slot_name, slot_type, catalog_xmin FROM pg_replication_slots;"
```

При значении `hot_standby_feedback = off` слоты на реплике могут быть
инвалидированы процессом VACUUM на Leader. Параметр включается через reload,
перезапуск не требуется.

### 2. Создать физический слот на prod

```bash
patronictl edit-config $OLD
```

```yaml
slots:
  pgbj_standby:
    type: physical
```

Слот обеспечивает сохранность WAL, необходимого новому кластеру, и удержание
`catalog_xmin` при работе `hot_standby_feedback`.

### 3. Повторно инициализировать pgbj как standby cluster

Параметр `standby_cluster` указывается в переменных Ansible
(`patroni_standby_cluster` в vitabaks/postgresql_cluster), после чего конфигурация
рендерится повторно:

```yaml
# bootstrap.dcs в patroni.yml
standby_cluster:
  host: pg-prod.df.example.com
  port: 5432
  primary_slot_name: pgbj_standby

postgresql:
  use_pg_rewind: true
  use_slots: true
  parameters:
    wal_level: logical
    wal_log_hints: 'on'
    hot_standby_feedback: 'on'
  pg_hba:                                    # bootstrap.pg_hba при клонировании не применяется
    - local  all   postgres                        peer
    - host   all   all          10.2.0.0/24        scram-sha-256
    - host   replication  replicator  10.2.0.0/24  scram-sha-256
    - host   replication  replicator  10.1.0.0/24  scram-sha-256
    - host   all          replicator  10.1.0.0/24  scram-sha-256
  create_replica_methods:
    - basebackup
  basebackup:
    max-rate: '100M'
    checkpoint: 'fast'
    compress: 'server-zstd'
```

Пароли `postgres` и `replicator` в секции `authentication` и в `.pgpass` на узлах BJ
привести к значениям prod. При клонировании роли будут получены от prod, и пароли,
сгенерированные Ansible для BJ, станут недействительными.

```bash
# остановить Patroni, удалить кластер из DCS, очистить PGDATA
for h in bj-pg1 bj-pg2 bj-pg3 bj-pg4; do ssh $h systemctl stop patroni; done
patronictl remove $NEW
for h in bj-pg1 bj-pg2 bj-pg3 bj-pg4; do
  ssh $h "sudo -u postgres find /var/lib/postgresql/16/main -mindepth 1 -delete"
done

# запускать узлы последовательно, дожидаясь состояния streaming
ssh bj-pg1 systemctl start patroni    # клонируется с prod, принимает роль Standby Leader
patronictl list $NEW
ssh bj-pg2 systemctl start patroni    # далее bj-pg3, bj-pg4
```

Перед выполнением удаления — см. контрольную меру Н1.

Секция `bootstrap` обрабатывается только при первичной инициализации кластера, что
определяет необходимость удаления кластера из DCS. Логические слоты на pgbj до
технологического окна отсутствуют, они создаются вручную после promote (шаг 8).

После запуска всех узлов кластер работает в режиме standby несколько суток, в течение
которых контролируется величина отставания и выполняется проверка плановых операций
(switchover внутри BJ).

### 4. Определить размещение слотов на новом кластере

| Вариант | Преимущества | Ограничения |
|---|---|---|
| Leader `bj-pg1` | Создание слота выполняется без задержки; слоты сохраняются при switchover внутри BJ при декларации в `slots:`; conninfo указывает на VIP | Логическое декодирование выполняется на primary |
| Async replica `bj-pg4` | Логическое декодирование не нагружает primary | Создание слота на кластере без активности не завершается (см. 8.3); слот существует только на данном узле; conninfo указывает на конкретный узел; `reinit` узла удаляет слот |

При выборе Leader слоты декларируются в DCS, что обеспечивает их копирование на
реплики и сохранение при switchover:

```bash
patronictl edit-config $NEW
```

```yaml
slots:
  pgdf_standby:              # используется на шаге 10
    type: physical
  sub_analytics_slot:
    type: logical
    database: appdb
    plugin: pgoutput
```

Patroni удаляет слоты, отсутствующие в секции `slots:`. Перечень должен включать все
слоты, в том числе физические.

---

## Технологическое окно

### 5. Остановить нагрузку на prod (T+0)

```bash
patronictl pause $OLD --wait && patronictl pause $NEW --wait
```

Остановка приложений и регламентных заданий, выполняющих запись в кластер,
производится владельцами сервисов и в данной инструкции не детализируется.
Перечень источников записи и порядок их остановки определяются до начала работ.

В `pg_hba.conf` на всех узлах DF оставить доступ только для `postgres` из
административной подсети и для `replication` с подсетей DF и BJ, после чего
выполнить `patronictl reload $OLD --force`.

```sql
-- на Leader DF: завершить оставшиеся клиентские сессии
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE backend_type='client backend' AND pid <> pg_backend_pid()
  AND usename NOT IN ('postgres','replicator');

-- проверить отсутствие подготовленных транзакций
SELECT * FROM pg_prepared_xacts;
```

```bash
# контроль отсутствия клиентской записи: LSN изменяется только служебными записями
psql -U postgres -Atc "SELECT pg_current_wal_lsn();"
```

Ограничение `pg_hba` действует на период вычитывания подписчиками. До штатной
остановки кластер сохраняет возможность приёма записи, и любая клиентская транзакция
изменит позицию, до которой должны дойти подписчики.

Подготовленные транзакции (`pg_prepared_xacts`) требуется завершить или откатить:
при наличии они переносятся на новый кластер и удерживают блокировки и `xmin`.

### 6. Дождаться вычитывания логическими подписчиками

```sql
-- на Leader DF
SELECT pg_current_wal_lsn();
```

```sql
-- на df-slots: ожидается replayed = LSN Leader и behind = 0 по всем слотам
SELECT pg_last_wal_replay_lsn() AS replayed;

SELECT slot_name, active,
       pg_wal_lsn_diff(pg_last_wal_replay_lsn(), confirmed_flush_lsn) AS behind
FROM pg_replication_slots WHERE slot_type='logical';
```

Контроль выполняется на `df-slots`, а не на Leader. Условие полного вычитывания
состоит из двух частей: реплика применила WAL до текущего LSN Leader, и слоты достигли
позиции реплики.

Слот в состоянии `active = false` не продвигается. Порядок действий определяется до
начала работ: запуск потребителя либо удаление слота с последующей повторной
инициализацией подписки.

### 7. Зафиксировать позиции подписчиков и отключить подписки

```sql
-- на каждом subscriber
SELECT external_id, remote_lsn FROM pg_replication_origin_status;
SELECT subname, subslotname, subtwophasestate, subpublications FROM pg_subscription;

ALTER SUBSCRIPTION sub_analytics DISABLE;
ALTER SUBSCRIPTION sub_analytics SET (slot_name = NONE);
```

Значения `remote_lsn` зафиксировать в файле: они используются на шаге 8.4. Значение
`subtwophasestate` определяет необходимость опции `TWO_PHASE` при создании слота
(`e` — enabled, `d` — disabled).

Конструкция `SET (slot_name = NONE)` не сбрасывает `pg_replication_origin`, позиция
применения сохраняется, и подписка продолжит работу с неё на новом publisher.
`DROP SUBSCRIPTION` удаляет origin и требует полной повторной загрузки таблиц.

#### 7.1. Остановить DF штатным способом

```bash
# Leader останавливается последним: BJ получает WAL с него
for h in df-pg3 df-pg2 df-pg1; do ssh $h systemctl stop patroni; done
# Patroni выполняет остановку PostgreSQL через pg_ctl -m fast
```

```bash
# зафиксировать LSN shutdown checkpoint: до этой позиции должен дойти BJ
ssh df-pg1 "sudo -u postgres /usr/lib/postgresql/16/bin/pg_controldata \
  -D /var/lib/postgresql/16/main | grep -E 'cluster state|Latest checkpoint location|REDO location'"
# ожидаемое значение: Database cluster state: shut down
# значение 'shut down in recovery' на репликах df-pg2 и df-pg3 соответствует норме
```

Если состояние Leader отличается от `shut down`, расхождение таймлиний уже
возникло, и на шаге 10 потребуется `pg_rewind`. В этом случае проверить наличие
`wal_log_hints` или контрольных сумм.

Требование к штатной остановке и его обоснование приведены в Н2. Параметр
`default_transaction_read_only` не обеспечивает решение данной задачи: работающий
primary формирует checkpoint по таймауту, записи `XLOG_RUNNING_XACTS` с интервалом
около 15 секунд и записи autovacuum, в результате чего расхождение возникает.

#### 7.2. Дождаться применения WAL на BJ

```sql
-- на bj-pg1: ожидается LSN shutdown checkpoint из шага 7.1
SELECT pg_last_wal_replay_lsn();
```

При штатной остановке процессы walsender передают остаток WAL до завершения, поэтому
значения LSN обычно совпадают сразу. Если применение не достигло требуемой позиции,
запустить Leader повторно и остановить его снова:

```bash
ssh df-pg1 systemctl start patroni    # дождаться состояния streaming на bj-pg1
ssh df-pg1 systemctl stop patroni
```

Promote не выполняется до совпадения значений LSN. При невыполнении этого условия
возникает расхождение таймлиний, и шаг 10 потребует `pg_rewind`.

### 8. Promote и создание логических слотов

#### 8.1. Выполнить promote

```bash
patronictl resume $NEW --wait
patronictl edit-config $NEW --force -s standby_cluster=null    # удалить блок standby_cluster
patronictl list $NEW                                           # ожидается роль Leader, TL +1
```

```sql
-- на bj-pg1
SELECT pg_current_wal_lsn();     -- зафиксировать как promote_lsn
SELECT xact_commit, tup_inserted, tup_updated, tup_deleted
FROM pg_stat_database WHERE datname='appdb';
```

Приложения на данном этапе не запускаются. Любая запись смещает позицию создания
слота относительно позиций подписчиков, см. 8.4 и Н4.

#### 8.2. Создать слоты на Leader нового кластера

```bash
psql "host=bj-pg1 port=5432 dbname=appdb user=replicator replication=database" \
  -c "CREATE_REPLICATION_SLOT sub_analytics_slot LOGICAL pgoutput (SNAPSHOT 'nothing');"
```

Эквивалент средствами SQL, при подключении к базе `appdb`:

```sql
SELECT * FROM pg_create_logical_replication_slot('sub_analytics_slot','pgoutput', false, false);
--                                     temporary ↑      two_phase ↑
```

Первая команда возвращает `slot_name | consistent_point | snapshot_name | output_plugin`.
Значение `consistent_point` зафиксировать: это LSN, с которого слот передаёт
изменения, он используется на шаге 8.4.

- `dbname=appdb` — логический слот привязан к базе подключения. Подключение
  выполняется к базе, указанной в поле `database` исходного слота.
- `SNAPSHOT 'nothing'` — снимок не требуется, начальная загрузка данных не
  выполняется (`copy_data = false`). В прежнем синтаксисе — `NOEXPORT_SNAPSHOT`.
- При `subtwophasestate = 'e'` добавляется опция `TWO_PHASE`:
  `... LOGICAL pgoutput (SNAPSHOT 'nothing', TWO_PHASE);`

#### 8.3. Создать слоты на async replica нового кластера

Альтернатива шагу 8.2 при размещении слотов на реплике.

```sql
-- на bj-pg4, подключение к appdb
SELECT * FROM pg_create_logical_replication_slot('sub_analytics_slot','pgoutput');
```

Запрос не завершится. В отдельной сессии, при необходимости несколько раз:

```sql
-- на bj-pg1 (Leader)
SELECT pg_log_standby_snapshot();
```

Создание логического слота на standby требует записи `XLOG_RUNNING_XACTS` от primary.
На кластере без активности такая запись не формируется. Функция
`pg_log_standby_snapshot()` добавлена в PostgreSQL 16 для данного случая.

На узле требуется `hot_standby_feedback = on` и `use_slots: true` в конфигурации
Patroni (физический слот на Leader), иначе слот будет инвалидирован при работе
VACUUM. Контроль состояния:

```sql
-- на bj-pg4
SELECT slot_name, wal_status, conflicting FROM pg_replication_slots;
-- значение conflicting = true означает непригодность слота, требуется пересоздание
```

#### 8.4. Сверить LSN слотов с позициями подписчиков

```sql
-- на узле нового кластера, где созданы слоты
SELECT slot_name, restart_lsn, confirmed_flush_lsn FROM pg_replication_slots
WHERE slot_type='logical';
```

```bash
# на bj-pg1: remote_lsn — из шага 7, confirmed_flush_lsn — из запроса выше
sudo -u postgres pg_waldump -p /var/lib/postgresql/16/main/pg_wal \
  -s <remote_lsn> -e <confirmed_flush_lsn> 2>/dev/null | grep -c "COMMIT"
# ожидаемое значение: 0
```

Проверяется не равенство значений LSN, а отсутствие зафиксированных транзакций в
интервале расхождения. Превышение `confirmed_flush_lsn` над `remote_lsn` на величину
служебных записей (end-of-recovery, checkpoint, running_xacts) соответствует норме.

Механизм, определяющий данное требование, описан в Н4. При `START_REPLICATION`
начальная позиция подписчика, меньшая `confirmed_flush_lsn` слота, перемещается до
`confirmed_flush_lsn` без возврата ошибки.

При наличии зафиксированных транзакций в интервале требуется повторная начальная
загрузка затронутых таблиц с `copy_data = true` либо сверка и ручное внесение
расхождений. Уменьшение позиции слота невозможно:
`pg_replication_slot_advance` перемещает слот только в направлении увеличения LSN.

```sql
-- перемещение слота вперёд, если требуется пропустить применённый подписчиком WAL
-- выполняется на узле, где размещён слот
SELECT pg_replication_slot_advance('sub_analytics_slot', '<remote_lsn>'::pg_lsn);
```

#### 8.5. Переключить подписки

```sql
-- на каждом subscriber
-- host: VIP Leader при размещении слотов по 8.2, либо узел bj-pg4 при размещении по 8.3
ALTER SUBSCRIPTION sub_analytics
  CONNECTION 'host=pg-prod.bj.example.com port=5432 dbname=appdb user=replicator password=...';
ALTER SUBSCRIPTION sub_analytics SET (slot_name = 'sub_analytics_slot');
ALTER SUBSCRIPTION sub_analytics ENABLE;
```

```sql
-- на publisher: слот переходит в состояние active, confirmed_flush_lsn увеличивается
SELECT slot_name, active, active_pid, confirmed_flush_lsn FROM pg_replication_slots;
-- на subscriber: remote_lsn увеличивается
SELECT external_id, remote_lsn FROM pg_replication_origin_status;
```

`REFRESH PUBLICATION` выполняется только с параметром `WITH (copy_data = false)`.
В противном случае запускается повторная начальная загрузка, приводящая к
дублированию данных и конфликтам.

Слоты на `df-slots` после переключения подписок не используются: подписка читает
данные с одного publisher, который размещён в BJ. DF остановлен, поэтому удаление
выполняется на шаге 10. До удаления слоты удерживают `catalog_xmin` и WAL.

#### 8.6. Включить синхронный режим

Установить `nosync: true` в секции tags на узлах `bj-pg3` и `bj-pg4`, после чего:

```bash
patronictl edit-config $NEW
```

```yaml
synchronous_mode: true
synchronous_node_count: 1
```

### 9. Переключить трафик

Переключить точку входа на `pg-prod.bj.example.com` — в DNS, на HAProxy либо в
конфигурации приложений, в зависимости от принятой схемы подключения. Запуск
приложений выполняется владельцами сервисов.

```sql
-- на bj-pg1: контроль появления клиентской нагрузки
SELECT usename, application_name, client_addr, count(*)
FROM pg_stat_activity WHERE backend_type='client backend'
GROUP BY 1,2,3 ORDER BY 4 DESC;

SELECT xact_commit, xact_rollback, tup_inserted, tup_updated, tup_deleted
FROM pg_stat_database WHERE datname='appdb';
```

```sql
-- на df-pg1 запись отсутствует: клиентских сессий нет
SELECT usename, application_name, client_addr FROM pg_stat_activity
WHERE backend_type='client backend' AND usename NOT IN ('postgres','replicator');
```

Зафиксировать время переключения как `T_switch`. Простой завершён.

Возврат на DF после `T_switch` сопровождается потерей всех изменений, выполненных
на BJ. Обратная логическая репликация в данной схеме не применяется: подписка
требует работающего primary, а DF остановлен. До выполнения шага 10 DF сохраняется
в виде согласованной копии на момент shutdown checkpoint. Интервал принятия решения
о возврате определяется до начала работ, см. Н5.

---

## Завершающий этап

### 10. Перевести DF в standby нового кластера

Выполняется после периода наблюдения, отдельно от технологического окна. До
выполнения шага DF сохраняется как копия состояния на момент остановки и является
единственным вариантом возврата.

Перед выполнением создать холодную копию каталога, см. контрольную меру Н6.

Повторная инициализация кластера не требуется. При штатной остановке (7.1)
расхождение таймлиний отсутствует, и узлы переключаются на новую таймлинию без
`pg_rewind`.

Проверка в режиме без изменений, при остановленном PostgreSQL:

```bash
# на df-pg1
sudo -u postgres /usr/lib/postgresql/16/bin/pg_rewind \
  --target-pgdata=/var/lib/postgresql/16/main \
  --source-server="host=pg-prod.bj.example.com user=postgres dbname=postgres" \
  --dry-run --progress
# сообщение "no rewind required" подтверждает отсутствие расхождения
```

```bash
patronictl edit-config $OLD
```

```yaml
standby_cluster:
  host: pg-prod.bj.example.com
  port: 5432
  primary_slot_name: pgdf_standby
```

```bash
patronictl resume $OLD --wait
ssh df-pg1 systemctl start patroni
patronictl list $OLD                         # ожидается роль Standby Leader на новой TL
ssh df-pg2 systemctl start patroni    # далее df-pg3
```

Узлы `df-pg2` и `df-pg3` являются репликами и собственной истории не формировали,
поэтому переключаются на новую таймлинию вслед за Standby Leader.

Удалить неиспользуемые слоты на `df-slots`:

```sql
-- на df-slots, после перехода узла в роль реплики нового кластера
SELECT slot_name, active, wal_status FROM pg_replication_slots WHERE slot_type='logical';
SELECT pg_drop_replication_slot('sub_analytics_slot');
```

Если `pg_rewind` потребовался и завершился с ошибкой — выполнить
`patronictl reinit $OLD <node>`.

---

## Известные проблемы и способы устранения

| Проблема | Способ устранения |
|---|---|
| `system_identifier` pgbj отличается от prod, потоковая репликация невозможна | Выполнить повторную инициализацию PGDATA: `patronictl remove`, очистка каталога, клонирование с prod. ОС, Patroni, etcd и сетевая конфигурация не изменяются |
| `standby_cluster` в `bootstrap.dcs` не применяется на работающем кластере | Удалить кластер из DCS: секция `bootstrap` обрабатывается только при первичной инициализации |
| Ansible перезаписывает ручные изменения `patroni.yml` | Указывать `standby_cluster` в переменных плейбука |
| При клонировании не применяются `bootstrap.pg_hba` и `bootstrap.users` | Перенести правила в `postgresql.pg_hba`; пароли на BJ заранее привести к значениям prod |
| Слоты размещены на реплике, проверки на Leader их не отображают | Выполнять проверки вычитывания на `df-slots`: `confirmed_flush_lsn` относительно `pg_last_wal_replay_lsn()` этой реплики |
| Слоты на реплике не управляются Patroni и удаляются при `reinit` узла | Не выполнять повторную инициализацию `df-slots` до завершения миграции. На новом кластере размещать слоты на Leader с декларацией в `slots:` |
| Слоты на реплике инвалидируются VACUUM на Leader (`conflicting = true`) | Включить `hot_standby_feedback = on` на узле со слотами и физический слот на Leader (`use_slots: true`). Удержание `catalog_xmin` обеспечивается совместным применением |
| Логические слоты не переносятся потоковой репликацией | Создать слоты вручную после promote, до запуска приложений (8.2) |
| Создание слота на реплике нового кластера не завершается | Выполнить `SELECT pg_log_standby_snapshot();` на Leader в отдельной сессии |
| Слот создан на позиции, превышающей позицию подписчика | Соблюдать порядок: остановка записи, вычитывание подписчиками, promote, создание слотов до запуска приложений. Контроль: `pg_waldump ... \| grep -c COMMIT` = 0 |
| `confirmed_flush_lsn` слота превышает `remote_lsn` подписчика | Проверять отсутствие зафиксированных транзакций в интервале. Служебные записи в интервале соответствуют норме |
| Требуется уменьшить позицию слота | Не поддерживается, `pg_replication_slot_advance` перемещает слот только вперёд. Выполнить повторную загрузку таблиц с `copy_data = true` либо сверку вручную |
| Слот создан в другой базе | Подключаться к базе, указанной в поле `database` исходного слота: `dbname=` в conninfo определяет привязку слота |
| Подписка с `two_phase` не использует слот, созданный без `TWO_PHASE` | Проверить `pg_subscription.subtwophasestate`; при значении `e` создавать слот с опцией `TWO_PHASE` |
| `DROP` и `CREATE SUBSCRIPTION` приводят к потере позиции применения | Использовать `ALTER SUBSCRIPTION ... SET (slot_name = ...)`: `pg_replication_origin` сохраняется. `REFRESH PUBLICATION` выполнять с `copy_data = false` |
| Слот в состоянии `active = false` не позволяет дождаться `behind = 0` | Определить порядок действий до начала работ: запуск потребителя либо удаление слота с повторной инициализацией подписки |
| Слоты удерживают WAL, возможно заполнение `pg_wal` на prod | Настроить оповещение по `pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn)`. Выбрать режим: `max_slot_wal_keep_size = -1` либо ограничение с риском перехода слота в состояние `lost` |
| conninfo подписки указывает на конкретный узел при размещении слотов на реплике | Выделить VIP или backend HAProxy на данный узел либо принять зависимость логической репликации от доступности узла |
| Требуется исключить запись до штатной остановки | Применить ограничение `pg_hba` и `pg_terminate_backend`. Параметр `default_transaction_read_only` снимается клиентом через `SET` и не действует на суперпользователя |
| Запись в оба кластера одновременно | Провести инвентаризацию источников записи, ограничить `pg_hba`, заранее снизить DNS TTL до 30–60 с, контролировать `pg_current_wal_lsn()` на DF после переключения |
| Необходимость повторной инициализации DF при переводе в standby | Не требуется. При штатной остановке (7.1) расхождение отсутствует и узлы переключаются на новую таймлинию. `pg_rewind` применяется при нештатной остановке, `reinit` — при невозможности `pg_rewind` |
| Неиспользуемые слоты на `df-slots` удерживают WAL и `catalog_xmin` | Выполнить `pg_drop_replication_slot` на шаге 10, после перехода узла в роль реплики нового кластера |
| Применение изменений с нового кластера при разошедшихся таймлиниях | Привести DF к точке расхождения через `pg_rewind`. Штатная остановка DF (7.1) исключает возникновение расхождения |
| Расхождение таймлиний при работающем primary без клиентской нагрузки | Checkpoint по таймауту, `XLOG_RUNNING_XACTS` с интервалом около 15 с и autovacuum формируют записи WAL. Требуется штатная остановка до состояния `shut down` |
| Promote выполнен до применения WAL на BJ до shutdown checkpoint | Сверять LSN на шаге 7.2 до promote. При отставании запустить Leader DF и остановить повторно |
| `pg_rewind` завершается с ошибкой: требуются контрольные суммы или `wal_log_hints` | Включить `wal_log_hints` на prod заранее, плановым поочерёдным перезапуском узлов |
| Состояние DF отличается от `shut down` | Запустить и остановить штатно через `pg_ctl -m fast stop`. Расхождение уже возникло, на шаге 10 потребуется `pg_rewind` и наличие `wal_log_hints` или контрольных сумм |
| Возврат после `T_switch` сопровождается потерей записей нового кластера | Определить интервал принятия решения и не выполнять шаг 10 до его истечения: DF сохраняется как копия, готовая к запуску. Обратная репликация неприменима, так как требует работающего primary на DF |
| Перевод DF в standby исключает возможность возврата | Выполнять шаг 10 отдельно от технологического окна, после периода наблюдения |
| Unlogged-таблицы не содержат данных после promote | Определить перечень заранее, перенести содержимое через `COPY` после promote, до запуска приложений |
| Подготовленные транзакции переносятся на новый кластер | Проверить `pg_prepared_xacts` до promote, завершить или откатить транзакции |
| Растянутый между ДЦ кластер etcd теряет кворум при недоступности канала | Использовать отдельный кластер etcd в каждом ДЦ и различные значения `scope` |
| Patroni в режиме `pause` не выполняет failover при отказе | Минимизировать время работы в режиме paused, обеспечить дежурство с правами `patronictl failover --force` |
| Синхронная реплика увеличивает время фиксации транзакции | Размещать синхронную реплику в одном ДЦ с Leader; значение `synchronous_mode_strict` определить отдельно |
| Различия версий glibc и ICU на BJ влияют на корректность текстовых индексов | Сверить версии до клонирования. При различии выполнить `REINDEX` и `ALTER DATABASE ... REFRESH COLLATION VERSION` |
| Отсутствие резервного копирования и мониторинга на новом кластере | Настроить резервное копирование и оповещения на pgbj непосредственно после promote |

---

## Откат после шага 9

Раздел описывает порядок возврата на `pgdf` в случае, когда проблема обнаружена
после переключения трафика.

### Условия применимости

| Условие | Действие при невыполнении |
|---|---|
| Шаг 10 не выполнен, `pgdf` остановлен и сохраняет собственную таймлинию | Восстановление из холодной копии, созданной в Н6 |
| Решение принято в пределах интервала, определённого в Н5 | Объём потерь соответствует фактическому времени работы `pgbj` |

Возврат сопровождается потерей всех изменений, выполненных на `pgbj` с момента
`T_switch`. Обратная логическая репликация в схеме не применяется, объединение двух
таймлиний средствами PostgreSQL невозможно.

### О1. Остановить запись в pgbj

Остановка приложений и регламентных заданий выполняется владельцами сервисов.
После этого завершить оставшиеся клиентские сессии:

```sql
-- на bj-pg1
SELECT pg_terminate_backend(pid) FROM pg_stat_activity
WHERE backend_type='client backend' AND pid <> pg_backend_pid()
  AND usename NOT IN ('postgres','replicator');
```

Остановка записи фиксирует итоговый объём расхождения между кластерами.

### О2. Оценить объём потерь

```sql
-- на bj-pg1
SELECT pg_size_pretty(pg_wal_lsn_diff(pg_current_wal_lsn(), '<promote_lsn>'::pg_lsn))
         AS wal_written_after_switch;

SELECT xact_commit, tup_inserted, tup_updated, tup_deleted
FROM pg_stat_database WHERE datname='appdb';

SELECT schemaname, relname, n_tup_ins, n_tup_upd, n_tup_del
FROM pg_stat_user_tables
WHERE n_tup_ins + n_tup_upd + n_tup_del > 0
ORDER BY n_tup_ins + n_tup_upd + n_tup_del DESC LIMIT 50;
```

Перечень таблиц из последнего запроса определяет состав выгрузки на шаге О3.

### О3. Сохранить состояние pgbj

Выполняется до запуска `pgdf`. После возврата трафика данные `pgbj` не могут быть
получены повторно.

```bash
pg_dumpall -h bj-pg1 -U postgres -f /backup/bj-full-$(date +%F-%H%M).sql
```

```bash
# выгрузка расхождения по таблицам из шага 2
pg_dump -h bj-pg1 -U postgres -d appdb \
  -t public.orders -t public.payments -t public.audit_log \
  --data-only --file=/backup/delta-after-switch.sql
```

Дополнительно рекомендуется снять снимки виртуальных машин узлов BJ.

### О4. Запустить pgdf

```bash
ssh df-pg1 systemctl start patroni    # узел был Leader, запускается в роли primary
patronictl list $OLD
ssh df-pg2 systemctl start patroni    # далее df-pg3
patronictl resume $OLD --wait
patronictl list $OLD
```

Конфигурация `pgdf` на данном этапе не содержит секции `standby_cluster`, так как
шаг 10 не выполнен. Кластер запускается на своей таймлинии в роли primary с
состоянием на момент shutdown checkpoint (шаг 7.1 основной процедуры).

### О5. Снять ограничения доступа на pgdf

```bash
# восстановить рабочий pg_hba.conf на всех узлах DF
patronictl reload $OLD --force
```

```sql
-- на df-pg1
SELECT * FROM pg_hba_file_rules WHERE error IS NOT NULL;
SELECT pg_current_wal_lsn();
```

### О6. Переключить трафик на pgdf

Переключить точку входа на `pg-prod.df.example.com`. Запуск приложений и
регламентных заданий выполняется владельцами сервисов.

```sql
-- на bj-pg1: контроль отсутствия записи
SELECT usename, application_name, client_addr FROM pg_stat_activity
WHERE backend_type='client backend' AND usename NOT IN ('postgres','replicator');
```

### О7. Привести подписчиков в согласованное состояние

```sql
-- на каждом subscriber
ALTER SUBSCRIPTION sub_analytics DISABLE;
SELECT external_id, remote_lsn FROM pg_replication_origin_status;
```

Позиция применения подписчика (`remote_lsn`) указывает на LSN таймлинии `pgbj` и на
`pgdf` не существует: `pgdf` остановлен на LSN shutdown checkpoint, который меньше
текущей позиции подписчика. Кроме того, подписчик содержит изменения, полученные с
`pgbj` после `T_switch`, которые на `pgdf` отсутствуют.

Простое переключение conninfo обратно на `df-slots` в этих условиях приведёт к
пропуску изменений `pgdf` в интервале между shutdown checkpoint и позицией
подписчика. Применимы два варианта.

**Вариант А — повторная начальная загрузка (рекомендуется).** Обеспечивает
согласованное состояние, время выполнения определяется объёмом таблиц.

```sql
-- на subscriber
DROP SUBSCRIPTION sub_analytics;

CREATE SUBSCRIPTION sub_analytics
  CONNECTION 'host=df-slots port=5432 dbname=appdb user=replicator password=...'
  PUBLICATION pub_analytics
  WITH (copy_data = true, create_slot = false, slot_name = 'sub_analytics_slot');
```

Слоты на `df-slots` сохранены, поскольку шаг 10 не выполнялся. Перед созданием
подписки проверить их наличие и состояние:

```sql
-- на df-slots
SELECT slot_name, active, wal_status, confirmed_flush_lsn FROM pg_replication_slots
WHERE slot_type='logical';
```

**Вариант Б — возврат позиции применения.** Применяется при недопустимости времени
полной загрузки. Требует последующей сверки данных.

```sql
-- на subscriber: external_id из запроса выше, LSN — shutdown checkpoint из шага 7.1 основной процедуры
SELECT pg_replication_origin_advance('pg_16405', '<shutdown_checkpoint_lsn>'::pg_lsn);

ALTER SUBSCRIPTION sub_analytics
  CONNECTION 'host=df-slots port=5432 dbname=appdb user=replicator password=...';
ALTER SUBSCRIPTION sub_analytics SET (slot_name = 'sub_analytics_slot');
ALTER SUBSCRIPTION sub_analytics ENABLE;
```

Изменения, полученные подписчиком с `pgbj`, остаются в его таблицах и при повторном
применении изменений `pgdf` приводят к конфликтам уникальных ключей и расхождению
данных. Состав расхождения определяется сверкой по таблицам из шага О2.

### О8. Перенести расхождение на pgdf

Выполняется при необходимости сохранить изменения, сделанные на `pgbj`.
Автоматический перенос невозможен: после возврата записи на `pgdf` значения
последовательностей могут совпасть со значениями, выданными на `pgbj`.

```bash
# анализ выгрузки из шага О3
less /backup/delta-after-switch.sql
```

Порядок переноса определяется прикладной логикой: сверка по естественным ключам,
переназначение идентификаторов, разрешение конфликтов. Скрипт сверки готовится до
начала работ, в составе плана отката.

### О9. Определить дальнейшее использование pgbj

`pgbj` находится на собственной таймлинии и для повторного использования в качестве
standby `pgdf` требует приведения к состоянию `pgdf`.

```bash
# на bj-pg1, при остановленном PostgreSQL
sudo -u postgres /usr/lib/postgresql/16/bin/pg_rewind \
  --target-pgdata=/var/lib/postgresql/16/main \
  --source-server="host=pg-prod.df.example.com user=postgres dbname=postgres" \
  --dry-run --progress
```

Операцию выполнять после завершения шагов О3 и О8: `pg_rewind` удаляет изменения
`pgbj` и очищает каталог `pg_replslot`. При невозможности `pg_rewind` выполняется
повторная инициализация по процедуре шага 3 основной процедуры.

### Контрольный перечень

| Пункт | Шаг |
|---|---|
| Запись в `pgbj` остановлена, клиентские сессии завершены | О1 |
| Объём потерь зафиксирован, перечень затронутых таблиц получен | О2 |
| Резервная копия `pgbj` и выгрузка расхождения созданы | О3 |
| `pgdf` работает в роли primary, Patroni выведен из режима paused | О4 |
| Ограничения `pg_hba` на `pgdf` сняты, ошибок в правилах нет | О5 |
| Трафик переключён, запись в `pgbj` отсутствует | О6 |
| Подписчики приведены в согласованное состояние | О7 |
| Решение по переносу расхождения принято и выполнено | О8 |
| Дальнейшее использование `pgbj` определено | О9 |
