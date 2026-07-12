# Часть III. Развёртывание *(75 стр.)*

> **Цель:** научиться планировать, разворачивать и масштабировать Ceph-кластер — от ручного метода для понимания внутренностей до промышленного cephadm, включая замкнутый контур.
> **После этой части вы сможете:** спроектировать кластер под задачу, развернуть его через cephadm, обновить и масштабировать на лету, развернуть Ceph без доступа в Интернет.

---

## Глава 6. Планирование кластера *(18 стр.)*

### 6.1. Требования к оборудованию *(4 стр.)*
- CPU: почему ~1 ядро на OSD, что дают высокие частоты
- RAM: OSD (~4 GB baseline + bluestore_cache), MON+ MDS (~1 GB/TB метаданных)
- Расчёт: OSD × 4 GB + MON × 2 GB + MDS × 2 GB + запас 20%
- Диски: HDD (cold storage), SSD (capacity), NVMe (performance)
- Сеть: 10GbE минимально, 25GbE рекомендовано, 100GbE для high-perf

### 6.2. Топологии *(4 стр.)*
- DOT-схемы: 3 узла, 5 узлов, 10+ узлов
- Гиперконвергентная: OSD + MON на одних и тех же узлах
- Разделённая: выделенные MON/ MDS-узлы
- Мульти-DC: растянутый кластер с site-aware CRUSH

### 6.3. Сеть *(4 стр.)*
- Public network: трафик клиент↔Ceph
- Cluster network: трафик OSD↔OSD (репликация, recovery, backfill)
- Разделение: физическое (разные интерфейсы) vs логическое (VLAN)
- Jumbo Frames, LACP, QoS — что, когда и зачем
- DOT-схема: сегментация сети для Ceph

### 6.4. Диски *(3 стр.)*
- HDD: дешёво, ёмко, медленно (100–200 IOPS)
- SSD SATA: средне (10k–100k IOPS)
- NVMe: быстро (100k–1M IOPS)
- WAL на NVMe, DB на SSD: как ускорить HDD-OSD
- Расчёт соотношения: WAL = 512 MB, DB = 4% от ёмкости HDD

### 6.5. Практикум: проект кластера *(3 стр.)*
- Задача 1: кластер на 50 TB полезных, репликация ×3, бюджетно
- Задача 2: кластер на 200 TB, SSD-only, low latency
- Задача 3: отказоустойчивость на уровне DC (два дата-центра)
- Для каждой: топология, диски, сеть, расчёт стоимости

---

## Глава 7. Ручное развёртывание *(20 стр.)*

### 7.1. Подготовка узлов *(3 стр.)*
- ОС: Ubuntu 22.04/24.04, RHEL 9, Astra Linux
- chrony/ntpd: синхронизация времени (clock skew > 0.05s = WARN)
- hostname: FQDN, /etc/hosts
- Подсети: public 10.0.1.0/24, cluster 10.0.2.0/24
- firewalld/iptables: порты MON (3300, 6789), OSD (6800–7300), MDS, RGW

### 7.2. cephadm изнутри *(3 стр.)*
- Что такое cephadm: Python-скрипт, который управляет Docker/Podman-контейнерами
- Откуда берётся: `curl --silent --remote-name ...`
- Что ставит: cephadm бинарник + bootstrap контейнеров
- Отличие от ceph-deploy и ceph-ansible

### 7.3. `cephadm bootstrap` — полный разбор *(5 стр.)*
- Каждая строка вывода — что происходит в этот момент:
  1. Проверка preflight (Python, Podman/Docker, время, диски)
  2. Генерация FSID, ключей (mon., admin, bootstrap-*)
  3. Создание MON (первый, он же лидер)
  4. Создание MGR
  5. Создание ключей для оркестратора
  6. Активация модулей MGR
  7. Создание дефолтного пула `.mgr`
  8. Вывод dashboard URL
- DOT-схема: что создаётся при bootstrap

### 7.4. Добавление узлов *(3 стр.)*
- `ceph orch host add <hostname> <ip> [--labels mon,mgr,osd]`
- Как cephadm подключается: SSH-ключ `/etc/ceph/ceph.pub` → `authorized_keys`
- Метки (labels): `_admin`, `mon`, `mgr`, `osd`, `mds`, `rgw`
- `ceph orch host ls` — проверяем

### 7.5. Развёртывание MON *(3 стр.)*
- `ceph orch apply mon --placement="3 host1,host2,host3"`
- Кворум: `ceph quorum_status` — читаем вывод
- Растягивание MON по разным стойкам/DC: зачем

### 7.6. Развёртывание OSD *(3 стр.)*
- `ceph orch daemon add osd host:/dev/sdb`
- Что внутри: `ceph-volume lvm create`, LVM PV/VG/LV, BlueStore
- Автоматическое обнаружение: `ceph orch apply osd --all-available-devices`

---

## Глава 8. cephadm и оркестрация *(20 стр.)*

### 8.1. Модель оркестрации *(4 стр.)*
- Service spec: YAML-описание сервиса (service_type, service_id, placement)
- Placement spec: `count:3`, `label:mon`, `hosts:[host1,host2]`
- `ceph orch apply -i spec.yaml` — построчный разбор YAML
- Примеры spec-ов: MON, OSD, MDS, RGW, NFS-Ganesha, Prometheus

### 8.2. Стек контейнеров *(3 стр.)*
- Podman (по умолчанию с Reef+) vs Docker
- systemd unit-ы: `systemctl status ceph-<fsid>@<service>`
- Образы: `quay.io/ceph/ceph:v19.2.x`
- Как посмотреть: `ceph orch ps`, `ceph orch ls`

### 8.3. Declarative state *(4 стр.)*
- Как оркестратор сравнивает «что есть» с «что должно быть»
- `ceph orch ls` — целевое состояние
- `ceph orch ps` — фактическое состояние
- Что происходит при расхождении: оркестратор запускает недостающие сервисы

### 8.4. Обновление кластера *(4 стр.)*
- `ceph orch upgrade start --ceph-version <version>`
- Фазы обновления: MON → MGR → OSD → MDS → RGW
- Контроль: `ceph orch upgrade status`, `ceph orch upgrade pause/resume`
- Откат: когда возможен, когда — нет

### 8.5. cephadm shell *(3 стр.)*
- `cephadm shell` — запуск контейнера с монтированными конфигами
- `--mount` для отладки
- Логи: `cephadm logs`, journald внутри контейнера

### 8.6. Практикум: масштабирование *(2 стр.)*
- Кластер: 3 узла → 5 узлов на лету
- Добавление, развёртывание MON и OSD
- Проверка: `ceph status`, перебалансировка данных

---

## Глава 9. Развёртывание в замкнутом контуре *(17 стр.)*

### 9.1. Air-gapped: ограничения *(2 стр.)*
- Что недоступно: пакетные репозитории (apt, yum), container registry (quay.io), PyPI
- Что нужно: локальное зеркало пакетов + локальный registry + офлайн-скрипты bootstrap
- План развёртывания air-gapped

### 9.2. Локальное зеркало пакетов *(4 стр.)*
- Aptly для DEB-пакетов (Ubuntu): создание зеркала, снапшоты, публикация
- createrepo для RPM-пакетов (RHEL/Astra): `createrepo_c`, `httpd`-раздача
- Полный список пакетов Ceph и зависимостей

### 9.3. Локальный container registry *(4 стр.)*
- Podman + `registry:2` — поднимаем локальный Docker Registry
- Skopeo: `skopeo copy` — вытягиваем образы на машине с интернетом, переносим
- Полный список образов: `quay.io/ceph/ceph:v19.2*`, `quay.io/ceph/keepalived`, `quay.io/prometheus/...`
- Авторизация: htpasswd, самоподписанный сертификат

### 9.4. cephadm в офлайне *(3 стр.)*
- `--image <local-registry>/ceph/ceph:v19.2.x`
- `--no-container-init` — не тянуть образы извне
- Кастомный `ceph.conf` с указанием локального registry
- Переменная окружения `CEPHADM_IMAGE`

### 9.5. Практикум: развёртывание без интернета *(4 стр.)*
- Зеркало Aptly + Registry + образы Ceph
- Полный Ansible playbook для air-gapped bootstrap
- Развёртывание кластера на 3 узлах без единого внешнего запроса
- Верификация: `curl` наружу должен падать, а кластер — работать

---

| Навигация | |
|-----------|---|
| ← Часть II | [part-II.md](part-II.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Часть IV | [part-IV.md](part-IV.md) |
