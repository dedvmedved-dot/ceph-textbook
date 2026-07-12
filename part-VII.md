# Часть VII. Разработка и интеграция *(55 стр.)*

> **Цель:** освоить программный доступ к Ceph через все интерфейсы — librados (Python), RBD, CephFS, RGW (S3 API).
> **После этой части вы сможете:** написать приложение на Python, работающее с объектами Ceph; управлять RBD-образами, снапшотами, клонами; монтировать CephFS; работать с RGW через boto3.

---

## Глава 22. Программный доступ: librados и Python *(18 стр.)*

### 22.1. Уровни доступа *(2 стр.)*
- librados (C/C++): низкоуровневый доступ к объектам
- librbd: блочные устройства (поверх librados)
- libcephfs: файловая система (поверх librados)
- RGW REST API: S3/Swift (HTTP)

### 22.2. Python-клиент *(3 стр.)*
- Пакет: `python3-rados` (Ubuntu), `python-rados` (RHEL)
- Подключение: `rados.Rados(conffile='/etc/ceph/ceph.conf')`
- Аутентификация: keyring, clientname
- Обработка ошибок: `rados.Error`, таймауты

### 22.3. CRUD объектов *(5 стр.)*
- Полный код: создание контекста → открытие пула → запись → чтение → удаление
- Синхронный цикл: `write()`, `read()`
- Асинхронный (AIO): `aio_write()`, `aio_read()`, `aio_flush()`, callback
- OMAPA (object map): `set_omap()`, `get_omap_vals()` — атомарные метаданные

### 22.4. Практикум: миграция объектов *(8 стр.)*
- Приложение: перенос объектов из пула A в пул B
- Сравнение: синхронный vs асинхронный — замер производительности
- Обработка ошибок: retry, exponential backoff
- Полный код + разбор каждой строки

---

## Глава 23. Блочные устройства RBD *(14 стр.)*

### 23.1. RBD внутри *(3 стр.)*
- DOT-схема: слои RBD — block device → striping → objects in RADOS
- Object map: отслеживание выделенных блоков
- Layering: copy-on-write для клонов
- Stripe unit, stripe count, object size

### 23.2. Снапшоты и клоны *(3 стр.)*
- Снапшот: `rbd snap create <image>@<snap>`
- Защита: `rbd snap protect` — нельзя удалить снапшот с клонами
- Клон: `rbd clone <pool>/<image>@<snap> <pool>/<clone>`
- Flatten: `rbd flatten <clone>` — отрыв от родителя, полная копия

### 23.3. RBD Mirror *(3 стр.)*
- Конфигурация: `rbd mirror pool enable`, `rbd mirror image enable`
- Мониторинг: `rbd mirror pool status`, `rbd mirror image status`
- Failover: `demote` → `promote`
- DOT-схема: первичный → вторичный кластер

### 23.4. Практикум *(5 стр.)*
- Снапшот → запись новых данных → клон → rollback
- Настройка RBD Mirror между двумя кластерами
- Failover и обратное переключение

---

## Глава 24. Файловая система CephFS *(12 стр.)*

### 24.1. CephFS внутри *(3 стр.)*
- DOT-схема: файл → inode → объекты данных в RADOS
- MDS: журнал, subtree, балансировка
- Иерархия снапшотов: `.snap/`

### 24.2. Мульти-MDS *(3 стр.)*
- Pinning: привязка каталогов к конкретному MDS
- Балансировка: `ceph fs set <fs_name> max_mds 4`
- Отказоустойчивость: standby-replay (горячий резерв с синхронизацией журнала)

### 24.3. NFS-Ganesha *(3 стр.)*
- Экспорт CephFS через NFSv4: когда клиент не может монтировать CephFS нативно
- DOT-схема: клиенты NFS → Ganesha → CephFS
- `ceph nfs cluster create`, `ceph nfs export create`

### 24.4. Практикум *(3 стр.)*
- Монтирование CephFS: kernel driver vs ceph-fuse
- Снапшот: `mkdir .snap/test`, восстановление файла
- Мульти-MDS: pinning каталога, проверка `ceph fs status`

---

## Глава 25. Объектное хранилище RGW *(11 стр.)*

### 25.1. S3-совместимый API *(3 стр.)*
- Полный список операций: bucket (create, list, delete), object (put, get, delete, multipart)
- Аутентификация: S3 signature v4, IAM users, access key + secret key
- Ограничения: максимальный размер объекта, bucket-квоты

### 25.2. Multi-site *(3 стр.)*
- Зоны: zonegroup, zone, realm
- Sync policy: `allowed`, `forbidden`, `filter`
- Конфликты: last-writer-wins, разрешение
- DOT-схема: мульти-DC топология RGW

### 25.3. Практикум: boto3 *(5 стр.)*
- Установка boto3, настройка endpoint_url
- Создание bucket, загрузка объекта, скачивание, удаление
- IAM policy: пользователь с доступом только к одному bucket
- Lifecycle policy: автоматическое удаление старых объектов

---

| Навигация | |
|-----------|---|
| ← Часть VI | [part-VI.md](part-VI.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Часть VIII | [part-VIII.md](part-VIII.md) |
