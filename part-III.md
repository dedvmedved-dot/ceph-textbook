# Часть III. Развёртывание *(75 стр.)*

> **Цель:** научиться планировать, разворачивать и масштабировать Ceph-кластер — от ручного метода для понимания внутренностей до промышленного cephadm, включая замкнутый контур.
> **После этой части вы сможете:** спроектировать кластер под задачу, развернуть его через cephadm, обновить и масштабировать на лету, развернуть Ceph без доступа в Интернет.

---

## Глава 6. Планирование кластера *(18 стр.)*

### 6.1. Требования: CPU, RAM, сеть — почему именно столько *(4 стр.)*

Правильное планирование — это 80% успеха. Ошибка на этапе проектирования (например, слишком слабые процессоры) приведёт к проблемам с производительностью, которые потом нельзя исправить без замены оборудования.

#### CPU (процессор)

**Правило: ~1 ядро на OSD.**

Почему именно 1 ядро? OSD — это однопоточный процесс (по крайней мере, основной цикл обработки запросов). Один OSD активно использует одно ядро. Дополнительные ядра нужны для:

| Процесс | Ядер |
|---------|------|
| Каждый OSD | ~1 |
| MON | 2–4 (зависит от размера кластера) |
| MGR | 1–2 |
| MDS (на каждый активный ранг) | 2–4 |
| RGW (на экземпляр) | 2–4 |
| Система (ОС, сеть) | 2–4 |

**Пример расчёта для сервера с 12 HDD:**
- 12 OSD × 1 ядро = 12 ядер
- MON (если на этом же сервере) = 2 ядра
- Система = 2 ядра
- **Итого:** 16 ядер минимум → 1× Xeon Silver 16C или 2× Xeon Silver 8C

**Частота vs количество ядер:** для Ceph важнее **количество** ядер, а не частота. Высокая частота помогает только при одном OSD, но при 12 OSD работа распределяется по ядрам.

#### RAM (оперативная память)

**Правило: ~4 ГБ на OSD (базово) + кеш BlueStore.**

| Компонент | Память |
|-----------|--------|
| OSD (базово) | 3–5 ГБ (процесс + служебные структуры) |
| BlueStore cache | 1–4 ГБ на HDD-OSD, до 16 ГБ на NVMe-OSD |
| MON | 2–4 ГБ |
| MDS | 1 ГБ на 1 ТБ метаданных (грубо) |
| Система | 4 ГБ |

**Пример:**
- Сервер с 12 HDD-OSD: 12 × (4 + 3) = 84 ГБ + MON 2 ГБ + система 4 ГБ = **90 ГБ → 96–128 ГБ RAM**

**Важно:** OOM (Out Of Memory) killer в Linux убивает процессы при нехватке памяти. Если OOM убьёт OSD, PG начнут деградировать. **Лучше больше памяти, чем меньше.**

#### Диски: HDD vs SSD vs NVMe

| Тип | IOPS (4k random) | MB/s (seq) | Latency | Цена/ТБ | Использование |
|-----|-----------------|-----------|---------|---------|--------------|
| HDD 7.2k | 100–150 | 200–250 | 4–10 ms | ~2 000 ₽ | Холодные данные, бэкапы |
| SSD SATA | 5k–20k | 500–550 | 0.1–1 ms | ~8 000 ₽ | Виртуализация, базы |
| NVMe | 100k–1M | 3 000–14 000 | 0.01–0.1 ms | ~15 000 ₽ | High-perf, WAL/DB |

---

### 6.2. Топологии *(4 стр.)*

#### Топология 1: Минимальная (3 узла)

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Узел 1   │  │ Узел 2   │  │ Узел 3   │
│ MON+MGR  │  │ MON+MGR  │  │ MON+MGR  │
│ OSD×4    │  │ OSD×4    │  │ OSD×4    │
└──────────┘  └──────────┘  └──────────┘
    ↑              ↑              ↑
    └──────────────┴──────────────┘
         10/25GbE public+cluster (единая сеть)
```

- 3 узла, всё на каждом (гиперконвергентная топология)
- MON: 3 (кворум 2, выдерживает отказ 1)
- OSD: 12 (по 4 на узел)
- Отказоустойчивость: 1 узел целиком

#### Топология 2: Средняя (5+ узлов)

```
┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐
│ MON-1    │ │ MON-2    │ │ MON-3    │ │ OSD-1    │ │ OSD-2    │
│ MGR      │ │ MGR      │ │ MGR      │ │ OSD×12   │ │ OSD×12   │
└──────────┘ └──────────┘ └──────────┘ └──────────┘ └──────────┘
    ↑              ↑              ↑           ↑            ↑
    └──────────────┴──────────────┴───────────┴────────────┘
         10/25GbE public ──── отдельно ──── 25/100GbE cluster
```

- MON-узлы выделенные (3 шт.)
- OSD-узлы (2+ шт., расширяется)
- MGR на MON-узлах
- Раздельные сети: public (клиентская) + cluster (репликация)

#### Топология 3: Enterprise (10+ узлов, мульти-DC)

```
DC1 (Москва)                        DC2 (Санкт-Петербург)
┌─────────────────────┐            ┌─────────────────────┐
│ MON-1, MON-2, MON-3 │            │ MON-4, MON-5        │
│ MGR×2, MDS×2        │            │ MGR (standby)       │
│ OSD×12 (300 ТБ)     │◄──────────►│ OSD×12 (300 ТБ)     │
│ RGW×2               │  синхр.    │ RGW×2               │
└─────────────────────┘  сеть (lat └─────────────────────┘
                         < 2 ms)
```

- Два дата-центра с синхронной репликацией
- MON: 3+2 (кворум 3, DC1 имеет большинство — защита от split-brain)
- CRUSH с site-aware правилами
- RGW multi-site

---

### 6.3. Сеть: public vs cluster, Jumbo Frames, LACP *(4 стр.)*

#### Public vs cluster network

Ceph рекомендует разделять два типа трафика:

| Сеть | Трафик | Требования |
|------|--------|-----------|
| **Public** | Клиент ↔ OSD, клиент ↔ MON | Пропускная способность, доступность |
| **Cluster** | OSD ↔ OSD (репликация, recovery, backfill) | Низкая задержка, высокая пропускная способность |

**Почему разделение:**
- Recovery после отказа OSD может насытить сеть, и клиенты перестанут получать данные вовремя
- Разделяя сети физически (разные интерфейсы) или логически (VLAN), вы изолируете клиентский трафик от внутреннего

```bash
# Пример: public на 10.0.1.0/24 (eth0, VLAN 101)
#          cluster на 10.0.2.0/24 (eth1, VLAN 102)
ceph config set global public_network 10.0.1.0/24
ceph config set global cluster_network 10.0.2.0/24
```

#### Jumbo Frames (MTU 9000)

Стандартный Ethernet-фрейм вмещает 1500 байт полезных данных (MTU = Maximum Transmission Unit). Для Ceph, где типичная операция записи — 4 МБ объект, это означает 2800+ фреймов на один объект. Каждый фрейм — это прерывание процессора, обработка заголовков, накладные расходы.

**Jumbo Frames (MTU 9000)** увеличивают полезную нагрузку в 6 раз:
- Меньше фреймов → меньше прерываний → ниже загрузка CPU → ниже задержка
- Типичный выигрыш: 10–20% на пропускной способности, 5–10% на latency

**Как включить:**
```bash
# На всех узлах + коммутаторе
ip link set eth1 mtu 9000
# В /etc/netplan/ или /etc/network/interfaces — постоянно
```

**Когда НЕ включать:** если в сети есть устройства с MTU 1500 — фрагментация фреймов «съест» весь выигрыш.

#### LACP (Link Aggregation Control Protocol)

Объединение нескольких физических интерфейсов в один логический для увеличения пропускной способности и отказоустойчивости.

```
eth0 ─┐
       ├── bond0 (LACP mode 4) ── 20/50/100 GbE
eth1 ─┘
```

- **Mode 4 (802.3ad):** требует поддержки на коммутаторе, распределяет трафик по хешу (IP/порт)
- **Mode balance-alb:** без поддержки коммутатора, балансировка на стороне сервера

---

### 6.4. Выбор дисков: HDD vs SSD vs NVMe, WAL/DB placement *(3 стр.)*

#### Стратегия WAL/DB

Для HDD-OSD размещение WAL и DB на быстром NVMe даёт огромный прирост:

```
┌─────────────────────────────────────────────┐
│ NVMe 1.6 ТБ                                  │
│ ┌──────────┐ ┌──────────┐ ┌──────────┐      │
│ │ OSD 1    │ │ OSD 2    │ │ OSD 3    │ ...  │
│ │ WAL: 2G  │ │ WAL: 2G  │ │ WAL: 2G  │      │
│ │ DB:  30G │ │ DB:  30G │ │ DB:  30G │      │
│ └──────────┘ └──────────┘ └──────────┘      │
└─────────────────────────────────────────────┘
         ↓              ↓              ↓
┌──────────┐ ┌──────────┐ ┌──────────┐
│ HDD 20T  │ │ HDD 20T  │ │ HDD 20T  │
│ OSD 1    │ │ OSD 2    │ │ OSD 3    │
└──────────┘ └──────────┘ └──────────┘
```

**Расчёт:** один NVMe 1.6 ТБ может обслужить 40+ HDD-OSD (WAL 2 ГБ + DB 30 ГБ ≈ 32 ГБ на OSD). Но учтите: если NVMe выйдет из строя — все 40 OSD потеряют WAL/DB и потребуют восстановления.

#### SSD vs NVMe для WAL/DB

- **NVMe:** низкая задержка, высокая пропускная способность. Идеально для WAL/DB.
- **SSD SATA:** приемлемо для DB, но для WAL лучше NVMe (WAL синхронный — критична задержка).

---

### 6.5. Практикум: проектируем кластер *(3 стр.)*

**Задача 1:** 50 ТБ полезных, репликация ×3, бюджетно.
- Сырая ёмкость: 150 ТБ
- Диски: 10 ТБ HDD → 15 шт. + 3 запасных = 18 дисков
- Серверы: 12-дисковые → 2 сервера
- Сеть: 10GbE, общая public+cluster
- MON: 3 шт. (на тех же серверах? или отдельных мини-ПК?)

**Задача 2:** 200 ТБ, SSD-only, low latency (< 1ms).
- Диски: 7.68 ТБ SSD SATA → 27 шт. (только данные) + репликация ×3 = ~80 шт.
- Серверы: 24-дисковые NVMe-серверы → 4 сервера
- Сеть: 25GbE public + 100GbE cluster
- MON: выделенные, SSD-only

**Задача 3:** Отказоустойчивость на уровне DC (два дата-центра).
- CRUSH правило с site-aware: 2 копии в DC1, 1 копия в DC2
- MON: 5 (3 в DC1, 2 в DC2 — большинство в DC1)
- RGW multi-site
- Требование: latency между DC < 2 мс для синхронной репликации

**Для каждой задачи опишите:** топологию, конфигурацию серверов, CRUSH rule, схему сети.

---

## Глава 7. Ручное развёртывание: пошаговый разбор *(20 стр.)*

### 7.1. Подготовка узлов *(3 стр.)*

**Узел** — это любой сервер (физический или виртуальный), на котором работают компоненты Ceph. Подготовка одинакова для всех узлов.

#### 1. Операционная система

Ceph Squid 19.2.x поддерживает:
- Ubuntu 22.04/24.04 LTS
- RHEL/CentOS 9, Rocky Linux 9
- Debian 12
- Astra Linux SE 1.7+ (российский дистрибутив)

**Для учебника используем Ubuntu 24.04 LTS.**

#### 2. Синхронизация времени (критично!)

MON требуют расхождения часов не более 0.05 секунд. Большее расхождение → `HEALTH_WARN: clock skew`.

```bash
apt install chrony -y
systemctl enable --now chrony
chronyc tracking   # проверить синхронизацию
# Важно: stratum, offset < 0.05s
```

**Если chrony недоступен (замкнутый контур):** настройте один узел как локальный NTP-сервер для остальных.

#### 3. Hostname и DNS

Каждый узел должен иметь уникальное имя и разрешаться по DNS или `/etc/hosts`:

```bash
hostnamectl set-hostname ceph-mon1.example.com

# /etc/hosts на каждом узле
10.0.1.10 ceph-mon1.example.com ceph-mon1
10.0.1.11 ceph-mon2.example.com ceph-mon2
# ... все узлы
```

#### 4. Сеть

```bash
# Public: eth0, 10.0.1.0/24
# Cluster: eth1, 10.0.2.0/24 (опционально)

# Убедиться, что IP статические (netplan или /etc/network/interfaces)
ip a
```

#### 5. Брандмауэр

```bash
# ufw (Ubuntu) или firewalld (RHEL/Astra)
# Открыть порты для Ceph
ufw allow 22/tcp        # SSH
ufw allow 3300/tcp      # MON
ufw allow 6789/tcp      # MON (старый)
ufw allow 6800:7300/tcp # OSD
ufw allow 8443/tcp      # Dashboard
```

#### 6. Docker/Podman

Cephadm использует контейнеры для всех сервисов:

```bash
# Ubuntu 24.04 — Podman (рекомендуется)
apt install podman -y
# Или Docker
apt install docker.io -y
```

---

### 7.2. cephadm: что внутри, откуда берётся *(3 стр.)*

**cephadm** — это единый Python-скрипт (около 3000 строк), который:

- Управляет жизненным циклом контейнеров Ceph (запуск, остановка, обновление)
- Создаёт systemd unit-ы для каждого сервиса
- Подключается к узлам по SSH и выполняет команды
- Хранит конфигурацию кластера в `/etc/ceph/`
- Сам не требует установки пакетов — это один самодостаточный файл

Отличие от предшественников:
- **ceph-deploy** (устарел): устанавливал пакеты напрямую в ОС, не использовал контейнеры
- **ceph-ansible** (устарел): Ansible-плейбуки, сложная зависимость от версий Ansible
- **cephadm** (современный): контейнеры, декларативное управление, встроен в образ Ceph

---

### 7.3. `cephadm bootstrap` — полный разбор вывода *(5 стр.)*

Команда `cephadm bootstrap` запускает **первый** узел кластера — узел начальной загрузки (bootstrap node). Выполняется **один раз** за всю жизнь кластера:

```bash
cephadm bootstrap --mon-ip 10.0.1.10
```

#### Построчный разбор вывода

```bash
# 1. Проверка preflight
Verifying podman|docker is present...         # Проверяет, установлен ли Podman/Docker
Verifying lvm2 is present...                  # LVM для OSD
Verifying time synchronization is in place... # chrony/ntpd
Verifying network configuration...            # IP, hostname, DNS
```

```bash
# 2. Генерация FSID и ключей
Creating /etc/ceph/ceph.conf...               # Главный конфигурационный файл
Generating new FSID: 51fa3f5c-...             # FSID — уникальный идентификатор кластера
Creating /etc/ceph/ceph.client.admin.keyring  # Ключ admin-пользователя (полный доступ!)
Creating /var/lib/ceph/bootstrap-*            # Bootstrap-ключи для добавления новых узлов
```

```bash
# 3. Создание первого MON
Creating mon...                               # MON на bootstrap-узле
# Запускается контейнер: docker run ... quay.io/ceph/ceph:v19.2
# Внутри контейнера: ceph-mon --mkfs -i mon1 --monmap ...
```

```bash
# 4. Создание MGR
Creating mgr...                               # MGR на bootstrap-узле
Enabling mgr modules: prometheus, dashboard... # Модули мониторинга
```

```bash
# 5. Dashboard
Ceph Dashboard is now available at:
  URL: https://ceph-mon1:8443
  User: admin
  Password: xxxxxxxxxx
```

```bash
# 6. Клиентские ключи
Generating client.admin keyring...
# Сохраняется в /etc/ceph/ceph.client.admin.keyring
```

**Итог:** после bootstrap у нас есть:
- 1 MON (он же лидер)
- 1 MGR
- Dashboard (веб-интерфейс)
- admin-ключ
- Bootstrap-ключи (для добавления новых узлов — их нельзя терять!)

---

### 7.4. Добавление узлов *(3 стр.)*

```bash
# Добавляем узел в кластер
ceph orch host add ceph-mon2 10.0.1.11 --labels _admin,mon,mgr,osd

# Что происходит:
# 1. cephadm по SSH подключается к 10.0.1.11
# 2. Копирует /etc/ceph/ceph.pub в ~/.ssh/authorized_keys
# 3. Устанавливает Podman/Docker (если нет)
# 4. Копирует конфиги в /etc/ceph/
# 5. Узел готов принимать сервисы

# Проверяем
ceph orch host ls
# Вывод:
# HOST       ADDR       LABELS          STATUS
# ceph-mon1  10.0.1.10  mon,mgr,osd
# ceph-mon2  10.0.1.11  mon,mgr,osd
```

**Метки (labels):**
- `_admin` — узел имеет admin-ключ (может управлять кластером)
- `mon`, `mgr`, `osd`, `mds`, `rgw` — на узле можно размещать соответствующие сервисы

---

### 7.5. Развёртывание MON *(3 стр.)*

```bash
# Разместить MON на трёх узлах
ceph orch apply mon --placement="ceph-mon1,ceph-mon2,ceph-mon3"

# Альтернативно — по меткам
ceph orch apply mon --placement="label:mon"

# Проверяем
ceph mon stat
# e3: 3 mons at {mon1=10.0.1.10:6789,mon2=10.0.1.11:6789,mon3=10.0.1.12:6789},
# election epoch 12, leader 0 mon1, quorum 0,1,2
```

**Кворум 3/3** — кластер полностью отказоустойчив по MON. Если упадёт любой один MON (например, mon2), кворум останется 2/3 — кластер продолжит работу.

---

### 7.6. Развёртывание OSD *(3 стр.)*

```bash
# Вариант 1: автоматически все доступные устройства
ceph orch apply osd --all-available-devices

# Вариант 2: конкретное устройство на конкретном узле
ceph orch daemon add osd ceph-osd1:/dev/sdb

# Что происходит внутри при создании OSD:
# 1. ceph-volume lvm zap /dev/sdb         # Очистка диска
# 2. ceph-volume lvm create --data /dev/sdb # Создание LVM PV/VG/LV
# 3. ceph-volume lvm activate --all       # Активация OSD (контейнер)
# 4. systemd unit: ceph-<fsid>@osd.<id>   # Сервис в systemd
```

**Проверка:**
```bash
ceph osd tree
# ID  CLASS  WEIGHT  TYPE NAME           STATUS  REWEIGHT  PRI-AFF
# -1         80.000  root default
# -3         20.000      host ceph-osd1
#  0   hdd   20.000          osd.0          up   1.00000  1.00000
#  1   hdd   20.000          osd.1          up   1.00000  1.00000
```

---

## Глава 8. cephadm и оркестрация *(20 стр.)*

### 8.1. Модель оркестрации: service spec *(4 стр.)*

Cephadm использует **декларативную модель** (declarative): вы описываете желаемое состояние в YAML, а оркестратор приводит реальность к этому состоянию.

**Service Spec (спецификация сервиса)** — YAML-файл, описывающий, что и где запускать:

```yaml
service_type: mon
service_id: mon
placement:
  hosts:
    - ceph-mon1
    - ceph-mon2
    - ceph-mon3
---
service_type: mgr
service_id: mgr
placement:
  hosts:
    - ceph-mon1
    - ceph-mon2
---
service_type: osd
service_id: default_drive_group
placement:
  host_pattern: '*'           # все узлы с меткой osd
data_devices:
  all: true                   # использовать все свободные диски
---
service_type: rgw
service_id: myrgw
placement:
  count: 2                    # 2 экземпляра на любых узлах с меткой rgw
  label: rgw
spec:
  rgw_realm: default
  rgw_zonegroup: default
  rgw_zone: default
```

Применение:
```bash
ceph orch apply -i cluster-spec.yaml
```

**Placement spec (спецификация размещения):** определяет, на каких узлах запускать сервис:

| Placement | Значение |
|-----------|----------|
| `hosts: [host1, host2]` | Конкретные узлы |
| `label: mon` | Все узлы с меткой `mon` |
| `count: 3` | Количество экземпляров (на любых узлах с подходящей меткой) |
| `count-per-host: 2` | По 2 экземпляра на каждом подходящем узле |
| `host_pattern: 'mon*'` | Все узлы, имя которых соответствует glob-шаблону |

---

### 8.2. Стек контейнеров: Podman, systemd, образы *(3 стр.)*

Каждый сервис Ceph (MON, OSD, MGR, MDS, RGW) работает в **отдельном контейнере**:

```bash
# Посмотреть все контейнеры Ceph
ceph orch ps

# Пример вывода:
# NAME            HOST       STATUS   PORTS
# mon.mon1        ceph-mon1  running  10.0.1.10:6789
# mon.mon2        ceph-mon2  running  10.0.1.11:6789
# mgr.mon1        ceph-mon1  running
# osd.0           ceph-osd1  running
# osd.1           ceph-osd1  running
```

**Как устроен контейнер Ceph (systemd):**

```
systemd unit: ceph-<fsid>@osd.0.service
    └── docker|podman run quay.io/ceph/ceph:v19.2.4
            └── ceph-osd -n osd.0 -f
```

Cephadm генерирует systemd unit для каждого сервиса. Это позволяет:
- Автоматически запускать сервис при загрузке ОС
- Мониторить через `systemctl status`
- Смотреть логи через `journalctl -u ceph-...`

**Образы (images):** все контейнеры используют один и тот же образ `quay.io/ceph/ceph:v19.2.4`, но запускают **разные команды** внутри (MON, OSD, MGR...):

```bash
# Это один и тот же образ, но с разными entrypoint!
# MON:  /usr/bin/ceph-mon -n mon.mon1 -f
# OSD:  /usr/bin/ceph-osd -n osd.0 -f
# MGR:  /usr/bin/ceph-mgr -n mgr.mon1 -f
```

---

### 8.3. Declarative state *(4 стр.)*

Оркестратор cephadm работает по принципу **reconciliation loop (цикла согласования):**

```
Бесконечный цикл:
  1. Прочитать желаемое состояние (spec)
  2. Прочитать фактическое состояние (ps)
  3. Сравнить
  4. Довести фактическое до желаемого
  5. Пауза 10 секунд → повторить
```

**Пример:** вы добавили spec на 3 MON. Оркестратор видит:
- Фактически: 1 MON (после bootstrap)
- Желаемое: 3 MON
- Действие: запустить MON на ceph-mon2 и ceph-mon3

**Пример 2:** OSD.5 упал (контейнер остановился):
- Оркестратор видит: фактически OSD.5 stopped
- Желаемое: OSD должны быть running
- Действие: перезапустить контейнер

Это обеспечивает **самовосстановление** на уровне сервисов: упавший контейнер будет перезапущен автоматически.

---

### 8.4. Обновление: `cephadm upgrade` *(4 стр.)*

Обновление кластера выполняется **на лету** (rolling upgrade), без остановки обслуживания клиентов:

```bash
# 1. Проверить HEALTH_OK
ceph health

# 2. Начать обновление до указанной версии
ceph orch upgrade start --ceph-version 19.2.4

# 3. Мониторинг
ceph orch upgrade status
# {
#   "target_image": "quay.io/ceph/ceph:v19.2.4",
#   "in_progress": true,
#   "services_complete": ["mon", "mgr"],
#   "message": "Upgrading OSDs..."
# }
```

**Фазы обновления (автоматически):**
1. **MON** — по одному (остальные держат кворум)
2. **MGR** — по одному
3. **OSD** — по одному, с ожиданием active+clean перед следующим
4. **MDS** — standby → active (переключение без простоя)
5. **RGW** — по одному

**Откат:** если обновление не завершено (in_progress):
```bash
ceph orch upgrade stop
# Вернуть предыдущую версию вручную для обновлённых сервисов
```

---

### 8.5. cephadm shell: отладка контейнеров *(3 стр.)*

```bash
# Запустить shell внутри контейнера с конфигами кластера
cephadm shell

# Внутри контейнера доступны все команды Ceph:
ceph status
ceph osd tree
rados lspools

# Подключить каталог с хоста
cephadm shell --mount /tmp/logs:/mnt/logs

# Посмотреть логи конкретного сервиса
cephadm logs --name osd.0

# Зайти в контейнер конкретного сервиса
cephadm enter --name osd.0
```

---

### 8.6. Практикум: масштабирование *(2 стр.)*

**Сценарий:** у вас кластер из 3 узлов (ceph-mon1..3). Нужно добавить OSD-узел.

```bash
# 1. Подготовить новый узел (см. §7.1)
# 2. Добавить в кластер
ceph orch host add ceph-osd4 10.0.1.20 --labels osd

# 3. Развернуть OSD на его дисках
ceph orch daemon add osd ceph-osd4:/dev/sdb
ceph orch daemon add osd ceph-osd4:/dev/sdc

# 4. Наблюдать за перебалансировкой
watch -n 5 'ceph status; ceph pg stat'
# PG должны переходить: active+clean → active+clean+remapped → active+clean

# 5. Дождаться HEALTH_OK
ceph health
```

**Задание:** засеките время от добавления узла до HEALTH_OK. Сравните с теоретической скоростью backfill (~1 PG/сек на OSD).

---

## Глава 9. Развёртывание в замкнутом контуре *(17 стр.)*

### 9.1. Air-gapped: ограничения, требования *(2 стр.)*

**Замкнутый контур (air-gapped environment)** — среда, где серверы **не имеют доступа в Интернет**. Это типично для:
- Государственных организаций (КИИ — критическая информационная инфраструктура)
- Закрытых военных сетей
- Изолированных промышленных сетей (АСУ ТП)
- Исследовательских лабораторий

**Что недоступно:**
- Пакетные репозитории (`apt update` не работает)
- Container registry (`docker pull quay.io/ceph/ceph` не работает)
- PyPI (`pip install` не работает)
- NTP-серверы в интернете
- DNS-резолвинг внешних имён

**Что нужно создать локально:**
1. Зеркало DEB/RPM-пакетов (Aptly, createrepo)
2. Локальный Docker/Podman Registry
3. Локальный NTP-сервер (chrony)
4. Локальный DNS (или /etc/hosts)

---

### 9.2. Локальное зеркало пакетов *(4 стр.)*

#### Aptly для Ubuntu (DEB)

```bash
# Установка Aptly на машине с интернетом («мастер-узел»)
apt install aptly -y

# Создание зеркала Ubuntu 24.04
aptly mirror create ubuntu-2404 http://archive.ubuntu.com/ubuntu noble main universe
aptly mirror update ubuntu-2404

# Создание зеркала Ceph
aptly mirror create ceph-squid https://download.ceph.com/debian-squid noble main
aptly mirror update ceph-squid

# Снапшот (заморозить состояние)
aptly snapshot create ubuntu-2404-snap from mirror ubuntu-2404
aptly snapshot create ceph-squid-snap from mirror ceph-squid

# Публикация (сервер репозитория)
aptly publish snapshot ubuntu-2404-snap ceph-squid-snap
# Репозиторий доступен по HTTP на порту 8080
```

На узлах без интернета:
```bash
# /etc/apt/sources.list.d/local.list
deb http://<мастер-узел>:8080/ noble main
apt update
```

#### createrepo для RHEL/Astra Linux (RPM)

```bash
# На машине с интернетом
reposync --repo=ceph-squid --download-path=/srv/repos/
createrepo_c /srv/repos/ceph-squid/

# Раздача через httpd (Apache)
# /etc/httpd/conf.d/repos.conf
Alias /repos /srv/repos
<Directory /srv/repos>
    Options Indexes FollowSymLinks
    Require all granted
</Directory>
```

---

### 9.3. Локальный container registry *(4 стр.)*

```bash
# На мастер-узле с интернетом:
# 1. Поднимаем локальный registry
mkdir /opt/registry
podman run -d -p 5000:5000 --name registry \
  -v /opt/registry:/var/lib/registry \
  registry:2

# 2. Скачиваем образы Ceph
CEPH_VERSION=19.2.4
IMAGES=(
  quay.io/ceph/ceph:v${CEPH_VERSION}
  quay.io/ceph/keepalived:2.1.5
  quay.io/prometheus/prometheus:v2.51.0
  quay.io/prometheus/node-exporter:v1.7.0
  quay.io/prometheus/alertmanager:v0.27.0
  quay.io/ceph/grafana:10.4.0
)

for img in "${IMAGES[@]}"; do
  podman pull $img
  podman tag $img localhost:5000/${img#quay.io/}
  podman push localhost:5000/${img#quay.io/}
done

# 3. Экспорт registry для переноса на изолированную сеть
tar czf registry.tar.gz -C /opt registry
# Перенести физически (USB-накопитель) и развернуть на изолированной сети
```

На узлах без интернета:
```bash
# /etc/containers/registries.conf
[[registry]]
location = "<мастер-узел>:5000"
insecure = true
```

---

### 9.4. cephadm offline *(3 стр.)*

```bash
# Bootstrap с локальным образом
cephadm --image <мастер-узел>:5000/ceph/ceph:v19.2.4 \
        bootstrap --mon-ip 10.0.1.10

# Если образ не тянется автоматически (--no-container-init):
cephadm --image <registry>/ceph/ceph:v19.2.4 \
        --no-container-init \
        bootstrap --mon-ip 10.0.1.10

# Указать глобально после bootstrap:
ceph config set mgr mgr/cephadm/container_image_ceph \
    <registry>/ceph/ceph:v19.2.4
```

---

### 9.5. Практикум: полное развёртывание без интернета *(4 стр.)*

**Цель:** развернуть Ceph-кластер из 3 узлов, у которых отключены все внешние сетевые интерфейсы.

**План (playbook):**

```yaml
# ansible/playbook-air-gapped.yml
- name: Развёртывание Ceph в замкнутом контуре
  hosts: all
  tasks:

  - name: Настройка локального репозитория
    copy:
      src: files/local.list
      dest: /etc/apt/sources.list.d/local.list
    # local.list содержит:
    # deb http://192.168.0.1:8080/ noble main

  - name: Установка Podman
    apt: name=podman state=present

  - name: Настройка локального container registry
    copy:
      src: files/registries.conf
      dest: /etc/containers/registries.conf
    # Содержит [registry] с location="192.168.0.1:5000"

  - name: Проверка, что интернета нет
    command: curl --connect-timeout 5 http://archive.ubuntu.com
    register: result
    failed_when: result.rc == 0  # ДОЛЖНО упасть!

  - name: Bootstrap Ceph
    command: >
      cephadm --image 192.168.0.1:5000/ceph/ceph:v19.2.4
      bootstrap --mon-ip {{ ansible_default_ipv4.address }}
```

**Верификация:** после развёртывания выполнить `ceph status`, `ceph osd tree`, `rados put/get`. Кластер должен полностью работать, при этом `curl http://archive.ubuntu.com` должен завершаться ошибкой.

---

| Навигация | |
|-----------|---|
| ← Часть II | [part-II.md](part-II.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Часть IV | [part-IV.md](part-IV.md) |
