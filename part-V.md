# Часть V. Производительность и тюнинг *(85 стр., 8 кейсов)*

> **Цель:** научиться измерять, анализировать и улучшать производительность Ceph на всех уровнях: OSD/BlueStore, сеть, балансировка нагрузки.
> **После этой части вы сможете:** снять baseline производительности, протюнинговать BlueStore под нагрузку, настроить QoS, найти и устранить узкое место.

---

## Глава 14. Методология тестирования *(20 стр.)*

### 14.1. Что и зачем измеряем *(4 стр.)*
- IOPS (Input/Output Operations Per Second): случайный доступ
- Throughput (MB/s): последовательный доступ
- Latency (ms): время ответа — среднее, p50, p95, p99
- Хвосты (tail latency): почему p99 важнее среднего

### 14.2. Инструменты *(5 стр.)*
- `rados bench`: встроенный бенчмарк — write/read/rand
- `rbd bench`: бенчмарк на уровне RBD — `--io-type write/read/rand`
- `fio` + librbd engine: самый гибкий инструмент
- DOT-схема: как инструменты тестирования взаимодействуют с кластером

### 14.3. Профили нагрузки *(3 стр.)*
- 100% случайное чтение (4k): OLTP-базы данных
- 100% последовательная запись (1M): бэкапы, потоковое видео
- 70% чтение / 30% запись: виртуализация
- Выбор профиля под бизнес-задачу

### 14.4. Снятие baseline *(3 стр.)*
- Эталонные замеры до любого тюнинга
- Что фиксировать: модель дисков, версия Ceph, топология, pg_num
- Хранение результатов: Grafana snapshot, JSON, скрипт

### 14.5. Практикум: baseline *(5 стр.)*
- Скрипт автоматического бенчмаркинга: `rados bench` + `rbd bench` + `fio`
- Построение графиков: gnuplot / matplotlib
- Интерпретация: IOPS, MB/s, latency — что хорошо, что плохо

---

## Глава 15. Тюнинг OSD и BlueStore *(22 стр.)*

### 15.1. BlueStore: путь байта *(4 стр.)*
- DOT-схема слоёв: BlockDevice → BlueFS → RocksDB → объекты
- BlockDevice: raw-доступ к диску (минуя ФС)
- BlueFS: мини-файловая система для RocksDB (WAL, DB, slow)
- RocksDB: key-value хранилище для метаданных объектов

### 15.2. WAL и DB на быстрых дисках *(4 стр.)*
- Когда это нужно: HDD-OSD с NVMe-разделом под WAL/DB
- Как настроить: `--block.db` и `--block.wal` при создании OSD
- Как измерить эффект: A/B-тест с `fio`, сравнение latency

### 15.3. Параметры OSD *(3 стр.)*
- `osd_op_threads`: потоки обработки операций
- `osd_recovery_op_priority`: приоритет recovery над клиентом
- `osd_max_backfills`: сколько PG одновременно восстанавливать
- `osd_recovery_max_active`: параллельные recovery-операции

### 15.4. bluestore_cache_size *(3 стр.)*
- Что кешируется: метаданные объектов, onode, bnode
- Автотюнинг: `bluestore_cache_autotune=true` (Squid+)
- Ручная настройка: `bluestore_cache_size_hdd`, `bluestore_cache_size_ssd`

### 15.5. bluestore_compression *(3 стр.)*
- Алгоритмы: snappy, zlib, zstd, lz4
- Когда помогает: сжимаемые данные (текст, ВМ-образы с нулями)
- Когда вредит: несжимаемые данные (видео, уже сжатое) — overhead CPU
- `ceph osd pool set <pool> compression_algorithm zstd`

### 15.6. Практикум: A/B-тест BlueStore *(5 стр.)*
- До: замеры с дефолтными настройками (fio)
- После: WAL на NVMe, compression=zstd, cache_size увеличен
- Таблица сравнения: IOPS, latency p99, CPU utilisation

---

## Глава 16. Тюнинг сети *(18 стр.)*

### 16.1. Сетевая модель Ceph *(3 стр.)*
- Async messenger v2 (MSGR2): TCP/TLS, сжатие, многопоточность
- Public vs cluster: зачем разделять физически или VLAN-ами
- Порты: `ms_bind_port_min` / `ms_bind_port_max`

### 16.2. Jumbo Frames *(3 стр.)*
- MTU 9000 vs 1500: меньше пакетов — меньше CPU — ниже latency
- Как измерить эффект: `ping -M do -s 8972`, `iperf3`
- Когда НЕ включать: смешанная сеть с MTU=1500, фрагментация

### 16.3. TCP vs RDMA *(3 стр.)*
- `ms_type = async+rdma` (экспериментально)
- RDMA (RoCE, InfiniBand): latency < 10 µs vs TCP 50–100 µs
- Когда оправдано: high-perf кластеры, NVMe-oF

### 16.4. QoS и приоритезация *(3 стр.)*
- DOT-схема: полосы трафика — client, recovery, scrub
- `osd_client_op_priority`, `osd_recovery_op_priority`, `osd_scrub_priority`
- `ceph config set osd osd_op_queue_cut_off high` — динамическое переключение

### 16.5. Балансировка *(3 стр.)*
- LACP (802.3ad, mode 4): агрегация каналов
- mode balance-alb: альтернатива без поддержки коммутатора
- Multi-home: несколько IP на OSD для разных сетей

### 16.6. Практикум: Jumbo Frames *(3 стр.)*
- Замеры latency до (MTU 1500) и после (MTU 9000)
- `rados bench` + `ping -s 8972`
- Таблица: средняя latency, p99, CPU utilisation

---

## Глава 17. Моделирование снижения производительности *(25 стр.)*

> Каждый кейс: имитация проблемы → как проявляется → диагностика → тюнинг → замер до/после.

### 17.1. Кейс 1: медленный диск *(3 стр.)*
- Имитация: `dm-delay` (device-mapper delay)
- Симптомы: slow requests, рост latency
- Диагностика: `iostat`, `ceph daemon osd.X dump_historic_ops`
- Устранение: замена диска, `ceph osd out`

### 17.2. Кейс 2: перегрузка сети *(3 стр.)*
- Имитация: `iperf3` в фоне, насыщение канала
- Симптомы: клиентская latency растёт, OSD heartbeat loss
- Диагностика: `iftop`, `nicstat`, `ceph osd perf`
- Тюнинг: QoS-приоритеты, отдельная cluster network

### 17.3. Кейс 3: deep-scrub во время нагрузки *(3 стр.)*
- Имитация: `ceph osd deep-scrub <pgid>` во время `rados bench`
- Симптомы: клиентская latency ×2–5
- Тюнинг: `osd_scrub_sleep`, `osd_scrub_begin_hour` (ночное окно)

### 17.4. Кейс 4: recovery после отказа *(3 стр.)*
- Имитация: `ceph osd out X` → recovery
- Симптомы: клиентский трафик degraded
- Тюнинг: `osd_recovery_op_priority`, `osd_max_backfills`

### 17.5. Кейс 5: hot spot OSD *(3 стр.)*
- Имитация: неравномерный CRUSH weight
- Симптомы: один OSD загружен на 90%, остальные на 30%
- `ceph osd reweight-by-utilization`, `balancer` модуль MGR

### 17.6. Кейс 6: OSD nearfull *(3 стр.)*
- Имитация: заполнить OSD до 85%+
- Симптомы: throttling, `HEALTH_WARN`
- Мониторинг: алерт Prometheus, прогноз заполнения

### 17.7. Кейс 7: memory pressure *(3 стр.)*
- Имитация: слишком много PG на OSD (>200)
- Симптомы: OOM killer, OSD flapping
- `pg_autoscaler`, `osd_memory_target`

### 17.8. Кейс 8: клиентский шторм *(3 стр.)*
- Имитация: 100 одновременных `fio` клиентов
- Симптомы: насыщение OSD-потоков, рост очереди
- Тюнинг: `osd_op_queue`, backoff на клиенте

### 17.9. Практикум: чёрный ящик *(1 стр.)*
- Преподаватель вносит проблему → студент находит и устраняет без подсказок

---

| Навигация | |
|-----------|---|
| ← Часть IV | [part-IV.md](part-IV.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Часть VI | [part-VI.md](part-VI.md) |
