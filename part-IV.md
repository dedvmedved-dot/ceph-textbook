# Часть IV. Эксплуатация, мониторинг и диагностика *(95 стр., 10 кейсов)*

> **Цель:** освоить мониторинг, научиться системно диагностировать неисправности и моделировать отказы.
> **После этой части вы сможете:** настроить полный мониторинг Ceph, расшифровать любой HEALTH-код, найти первопричину неисправности по симптомам, отработать 10 типовых кейсов отказа.

---

## Глава 10. Мониторинг: полный обзор *(25 стр.)*

### 10.1. `ceph status` — построчный разбор *(4 стр.)*
- Каждое поле: `cluster.id`, `health`, `services`, `data`, `io`, `progress`
- HEALTH_OK / HEALTH_WARN / HEALTH_ERR: пороги и смысл
- `services`: MON (quorum), MGR (active/standby), OSD (up/in)
- `data`: pools, PG, usage
- `io`: client reads/writes, recovery reads/writes

### 10.2. `ceph health detail` — классификация *(4 стр.)*
- Полная таблица HEALTH-кодов (>50): код → severity → причина → решение
- WARN: nearfull, pgs degraded, clock skew, mons down (1/3)
- ERR: full OSD, pgs inconsistent, mons down (2/3), no active mgr
- MUTE: как временно заглушить известную проблему

### 10.3. Prometheus *(5 стр.)*
- Экспортёры: `prometheus` модуль MGR, node_exporter на каждом узле
- Метрики: `ceph_osd_*`, `ceph_pg_*`, `ceph_mon_*`, `ceph_rbd_*`
- Alerts: Prometheus rules для Ceph (полный набор)
- `promtool check rules` — валидация

### 10.4. Grafana *(4 стр.)*
- Официальные дашборды: OSD overview, Pool overview, Host overview
- Кастомные панели: latency heatmap, PG state timeline, IOPS per OSD
- Скриншоты + JSON моделей дашбордов

### 10.5. Ceph Dashboard *(3 стр.)*
- Активация: `ceph dashboard enable`, SSL, пользователи
- Разделы: Cluster, Hosts, OSD, Pools, Block (RBD), Filesystem (CephFS), Object (RGW)
- `ceph dashboard` CLI: создание пользователя, `set-grafana-api-url`

### 10.6. Логи *(3 стр.)*
- `/var/log/ceph/ceph-{mon,osd,mgr,mds}.*.log`
- journald: `journalctl -u ceph-<fsid>@osd.0`
- `ceph log last [N]` — последние N записей кластерного журнала
- Приоритеты: `debug_ms`, `debug_osd`, `debug_mon` — когда повышать

### 10.7. Практикум: настрой мониторинг *(2 стр.)*
- Prometheus + Grafana + Ceph Dashboard для 5-узлового кластера
- Настроить алерт на OSD > 80%

---

## Глава 11. Диагностика неисправностей *(28 стр.)*

### 11.1. Методика диагностики *(3 стр.)*
- Системный подход: симптом → гипотеза → проверка → первопричина → устранение
- DOT-схема: дерево решений для типовых симптомов
- Что проверять в первую очередь: `ceph status`, `ceph health detail`, `dmesg`

### 11.2. HEALTH_WARN: 20 предупреждений *(5 стр.)*
- Полный разбор: `PG_DEGRADED`, `OSD_DOWN`, `MON_CLOCK_SKEW`, `NEARFULL`, ...
- Для каждого: точная причина, какой порог, как исправить

### 11.3. HEALTH_ERR: 10 критических ошибок *(4 стр.)*
- `OSD_FULL`, `PG_DAMAGED`, `MON_DOWN (2/3)`, `CACHE_POOL_NEAR_FULL`, ...
- Первопричины: почему они возникают, цепочка событий

### 11.4. OSD down/out *(4 стр.)*
- `ceph osd tree` — статус OSD (up/down, in/out)
- `ceph osd find <id>` — на каком хосте OSD
- `ceph daemon osd.X status` — admin socket
- Причины: процесс упал, диск отмонтировался, сеть, OOM killer

### 11.5. PG stuck *(5 стр.)*
- Inactive: PG не может выбрать acting set
- Unclean: не все реплики синхронизированы
- Inconsistent: расхождение данных между репликами
- Degraded: меньше реплик, чем нужно
- Peered, stale, undersized: что каждый означает

### 11.6. Сетевая диагностика *(3 стр.)*
- Clock skew: `chronyc tracking`, `timedatectl`
- Packet loss: `ping -M do -s 8972` (проверка MTU)
- MTU mismatch: почему ломает Ceph, `tracepath`

### 11.7. Инструменты диагностики *(4 стр.)*
- `ceph tell osd.X injectargs --debug_osd 20` — динамическая отладка
- `ceph daemon osd.X dump_historic_ops` — история медленных операций
- `ceph-objectstore-tool`: экспорт/импорт OSD в аварийном режиме
- `ceph-kvstore-tool`: низкоуровневый доступ к RocksDB

---

## Глава 12. Моделирование неисправностей: 10 кейсов *(22 стр.)*

> Каждый кейс: сценарий → как смоделировать → симптомы → диагностика → устранение → проверка.

### 12.1. Кейс 1: отказ одного OSD-диска *(2 стр.)*
- Модель: `systemctl stop ceph-<fsid>@osd.X`
- Симптомы: `PG_DEGRADED`, `OSD_DOWN`
- Восстановление: `systemctl start ...`, backfill

### 12.2. Кейс 2: потеря сети на OSD-узле *(2 стр.)*
- Модель: `iptables -A INPUT -j DROP` на OSD-хосте
- Симптомы: OSD помечаются down, degraded PGs
- Таймауты: `osd_heartbeat_grace`

### 12.3. Кейс 3: split-brain MON *(2 стр.)*
- Модель: iptables-блокировка одного MON
- Симптомы: кворум 2/3, `MON_CLOCK_SKEW`
- Восстановление: разблокировка → перевыборы

### 12.4. Кейс 4: отказ двух OSD одновременно *(2 стр.)*
- Модель: остановка 2 OSD с общими PG
- Симптомы: `PG_DEGRADED`, `undersized`, риск потери данных при min_size=1
- Восстановление: запуск OSD → backfill

### 12.5. Кейс 5: зависший OSD *(2 стр.)*
- Модель: `kill -STOP <osd-pid>`
- Симптомы: OSD не отвечает на heartbeats, slow requests
- Диагностика: `ceph daemon osd.X perf dump`, `dump_historic_ops`

### 12.6. Кейс 6: OSD nearfull *(2 стр.)*
- Модель: забить OSD данными до 85%+
- Симптомы: `OSD_NEARFULL`, `HEALTH_WARN`
- Эвакуация: `ceph osd reweight`, добавление OSD

### 12.7. Кейс 7: медленный диск *(2 стр.)*
- Модель: `dm-delay` device-mapper
- Симптомы: slow requests, `osd perf` показывает высокую latency
- Диагностика: `iostat -x 1`, `ceph daemon osd.X dump_historic_ops`

### 12.8. Кейс 8: повреждение ранга MDS *(2 стр.)*
- Модель: убить процесс MDS во время активной записи
- Симптомы: CephFS недоступна, MDS в replay
- Восстановление: журнал MDS, standby takeover

### 12.9. Кейс 9: PG inconsistent *(2 стр.)*
- Модель: ручное повреждение объекта на одном OSD
- Симптомы: `PG_INCONSISTENT`
- `ceph pg repair` — когда помогает, когда нет

### 12.10. Кейс 10: потеря MON majority *(3 стр.)*
- Модель: остановка 2 из 3 MON
- Симптомы: кластер не отвечает, нет кворума
- Ручное восстановление: извлечение monmap, формирование нового кворума

### 12.11. Практикум *(1 стр.)*
- Выбрать 5 кейсов, отработать на стенде, записать логи, оформить выводы

---

## Глава 13. Инструментарий диагностики *(20 стр.)*

### 13.1. Admin socket *(4 стр.)*
- `ceph daemon osd.X help` — полный список команд
- DOT-схема: как admin socket взаимодействует с демоном
- Ключевые команды: `perf dump`, `dump_historic_ops`, `config show`, `dump_mempools`

### 13.2. `ceph tell` *(3 стр.)*
- Отправка команд демонам: `ceph tell osd.X injectargs`
- Динамическое изменение конфигурации без перезапуска
- Опасные команды: `heap start_profiler`, `dump_reservations`

### 13.3. `ceph pg` *(4 стр.)*
- `ceph pg dump` — полный дамп PG
- `ceph pg <pgid> query` — детальная информация о PG
- `ceph pg map <pgid>` — на каких OSD лежит PG
- `ceph pg ls` — список PG с состоянием

### 13.4. `rados` CLI *(3 стр.)*
- `rados lspools`, `rados ls -p <pool>`
- `rados get/put`, `rados listomapkeys`, `rados listxattr`
- Ручная работа на уровне объектов — для отладки

### 13.5. `ceph-objectstore-tool` *(3 стр.)*
- Аварийный экспорт OSD: `--op export --data-path ...`
- Импорт: восстановление OSD из экспорта
- Когда это нужно: потеря монмапов, ручное восстановление

### 13.6. `ceph-kvstore-tool` *(3 стр.)*
- Прямой доступ к RocksDB на OSD
- `--op list`, `--op get`, `--op store-crc`
- Диагностика повреждений на уровне ключ-значение

---

| Навигация | |
|-----------|---|
| ← Часть III | [part-III.md](part-III.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Часть V | [part-V.md](part-V.md) |
