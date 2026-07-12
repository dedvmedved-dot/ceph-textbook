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

```dot
digraph G {
    rankdir=TB;
    bgcolor="white";
    fontname="Arial";
    fontcolor="black";
    splines=ortho;
    node [shape=box, style="filled,rounded", fontname="Arial", fontcolor="black"];
    edge [fontname="Arial", fontcolor="black"];

    node1 [label="Узел 1\nMON+MGR\nOSD×4", fillcolor="#BBDEFB"];
    node2 [label="Узел 2\nMON+MGR\nOSD×4", fillcolor="#C8E6C9"];
    node3 [label="Узел 3\nMON+MGR\nOSD×4", fillcolor="#FFE0B2"];

    net [label="10/25GbE public+cluster\n(единая сеть)", shape=box, style="filled", fillcolor="#E0E0E0"];

    {rank=same; node1; node2; node3;}

    node1 -> net [dir=none];
    node2 -> net [dir=none];
    node3 -> net [dir=none];
}
```

- 3 узла, всё на каждом (гиперконвергентная топология)
- MON: 3 (кворум 2, выдерживает отказ 1)
- OSD: 12 (по 4 на узел)
- Отказоустойчивость: 1 узел целиком

#### Топология 2: Средняя (5+ узлов)

```dot
digraph G {
    rankdir=TB;
    bgcolor="white";
    fontname="Arial";
    fontcolor="black";
    splines=ortho;
    node [shape=box, style="filled,rounded", fontname="Arial", fontcolor="black"];
    edge [fontname="Arial", fontcolor="black"];

    mon1 [label="MON-1\nMGR", fillcolor="#BBDEFB"];
    mon2 [label="MON-2\nMGR", fillcolor="#BBDEFB"];
    mon3 [label="MON-3\nMGR", fillcolor="#BBDEFB"];
    osd1 [label="OSD-1\nOSD×12", fillcolor="#C8E6C9"];
    osd2 [label="OSD-2\nOSD×12", fillcolor="#C8E6C9"];

    pub [label="10/25GbE public", shape=box, style="filled", fillcolor="#E3F2FD"];
    clu [label="25/100GbE cluster", shape=box, style="filled", fillcolor="#FFECB3"];

    {rank=same; mon1; mon2; mon3; osd1; osd2;}

    mon1 -> pub [dir=none];
    mon2 -> pub [dir=none];
    mon3 -> pub [dir=none];
    osd1 -> pub [dir=none];
    osd2 -> pub [dir=none];

    mon1 -> clu [dir=none];
    mon2 -> clu [dir=none];
    mon3 -> clu [dir=none];
    osd1 -> clu [dir=none];
    osd2 -> clu [dir=none];
}
```

- MON-узлы выделенные (3 шт.)
- OSD-узлы (2+ шт., расширяется)
- MGR на MON-узлах
- Раздельные сети: public (клиентская) + cluster (репликация)

#### Топология 3: Enterprise (10+ узлов, мульти-DC)

```dot
digraph {
    bgcolor=white
    rankdir=LR
    node [fontname="Arial", fontsize=11, fontcolor=black, shape=box, style="filled,rounded"]
    edge [fontname="Arial", fontsize=9, fontcolor=black, color="#555555", arrowsize=0.8]

    subgraph cluster_dc1 {
        label="DC1 (Москва)"
        fontname="Arial" fontsize=12 fontcolor=black
        color="#E3F2FD" style="filled"
        mon1 [label="MON-1, MON-2, MON-3" fillcolor="#90CAF9"]
        mgr1 [label="MGR×2, MDS×2" fillcolor="#90CAF9"]
        osd1 [label="OSD×12 (300 ТБ)" fillcolor="#BBDEFB"]
        rgw1 [label="RGW×2" fillcolor="#BBDEFB"]
    }

    subgraph cluster_dc2 {
        label="DC2 (Санкт-Петербург)"
        fontname="Arial" fontsize=12 fontcolor=black
        color="#FFF3E0" style="filled"
        mon2 [label="MON-4, MON-5" fillcolor="#FFCC80"]
        mgr2 [label="MGR (standby)" fillcolor="#FFCC80"]
        osd2 [label="OSD×12 (300 ТБ)" fillcolor="#FFE0B2"]
        rgw2 [label="RGW×2" fillcolor="#FFE0B2"]
    }

    dc1_edge [label="синхр. сеть\n(lat < 2 ms)" shape=plaintext fontname="Arial" fontsize=10]
    osd1 -> dc1_edge -> osd2 [dir=both, style=bold, color="#1976D2"]
}
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

```dot
digraph {
    bgcolor=white
    rankdir=TB
    node [fontname="Arial", fontsize=11, fontcolor=black, shape=box, style="filled,rounded"]
    edge [fontname="Arial", fontsize=9, fontcolor=black, color="#555555", arrowsize=0.8]

    subgraph cluster_nvme {
        label="NVMe 1.6 ТБ"
        fontname="Arial" fontsize=12 fontcolor=black
        color="#E8EAF6" style="filled"
        penwidth=2

        subgraph cluster_osd1 {
            label="OSD 1" fontname="Arial" fontsize=10
            color="#C5CAE9" style="filled"
            wal1 [label="WAL: 2G" fillcolor="#9FA8DA" fontsize=10]
            db1  [label="DB: 30G" fillcolor="#9FA8DA" fontsize=10]
        }
        subgraph cluster_osd2 {
            label="OSD 2" fontname="Arial" fontsize=10
            color="#C5CAE9" style="filled"
            wal2 [label="WAL: 2G" fillcolor="#9FA8DA" fontsize=10]
            db2  [label="DB: 30G" fillcolor="#9FA8DA" fontsize=10]
        }
        subgraph cluster_osd3 {
            label="OSD 3" fontname="Arial" fontsize=10
            color="#C5CAE9" style="filled"
            wal3 [label="WAL: 2G" fillcolor="#9FA8DA" fontsize=10]
            db3  [label="DB: 30G" fillcolor="#9FA8DA" fontsize=10]
        }
    }

    hdd1 [label="HDD 20 ТБ\nOSD 1" fillcolor="#FFCDD2"]
    hdd2 [label="HDD 20 ТБ\nOSD 2" fillcolor="#FFCDD2"]
    hdd3 [label="HDD 20 ТБ\nOSD 3" fillcolor="#FFCDD2"]

    wal1 -> hdd1 [style=dashed, color="#C62828", label="WAL/DB\nссылка" fontsize=9]
    wal2 -> hdd2 [style=dashed, color="#C62828"]
    wal3 -> hdd3 [style=dashed, color="#C62828"]
}
```

**Расчёт:** один NVMe 1.6 ТБ может обслужить 40+ HDD-OSD (WAL 2 ГБ + DB 30 ГБ ≈ 32 ГБ на OSD). Но учтите: если NVMe выйдет из строя — все 40 OSD потеряют WAL/DB и потребуют восстановления.

#### SSD vs NVMe для WAL/DB

- **NVMe:** низкая задержка, высокая пропускная способность. Идеально для WAL/DB.
- **SSD SATA:** приемлемо для DB, но для WAL лучше NVMe (WAL синхронный — критична задержка).

---

### 6.5. Реальные серверные платформы: Dell PowerEdge R750xs и Supermicro *(6 стр.)*

Выбор правильного сервера — это не только про CPU и диски. Нужно учитывать: форм-фактор, слоты расширения, контроллеры, охлаждение, блоки питания, поддержку производителя. Рассмотрим две популярные платформы для Ceph.

#### Dell PowerEdge R750xs

**Назначение:** универсальный 2U-сервер для SDS (Software-Defined Storage). Идеален для Ceph OSD-узлов средней плотности.

**Ключевые характеристики:**
| Параметр | Значение |
|----------|----------|
| Форм-фактор | 2U rack |
| Процессоры | 2× Intel Xeon Scalable 3-го поколения (Ice Lake), до 32 ядер |
| Память | 16× DDR4-3200 RDIMM, до 1 ТБ |
| Дисковые отсеки | До 12× 3.5" SAS/SATA (front) + 2× 3.5" или 4× 2.5" (rear) |
| NVMe | До 8× NVMe U.2 (direct drive) |
| Сеть | 2× 1GbE (onboard) + OCP 3.0 + до 3× PCIe 4.0 |
| RAID | PERC H755 (рекомендуется в HBA-режиме для Ceph) |
| Управление | iDRAC9 Enterprise |
| Блоки питания | 2× 1400 Вт (hot-plug, 1+1 redundant) |

**Типовая конфигурация для Ceph OSD-узла (12 HDD + 2 NVMe):**
```
Dell R750xs:
├── 2× Intel Xeon Silver 4314 (16C, 2.4 GHz) — 32 ядра / 64 потока
├── 256 ГБ RAM (8× 32GB DDR4-3200 RDIMM)
├── Storage:
│   ├── 2× 480GB SSD SATA RAID1 (ОС, BOSS-карта)
│   ├── 2× 3.2TB NVMe U.2 (WAL/DB для HDD)
│   └── 12× 20TB SAS HDD 7.2k (данные OSD)
├── Сеть:
│   ├── 2× 25GbE SFP28 (OCP 3.0) — public network, bond LACP
│   └── 2× 100GbE QSFP28 (PCIe) — cluster network, bond LACP
├── PERC H755 в HBA-режиме (passthrough)
└── iDRAC9 Enterprise
```

**Расчёт стоимости (ориентировочно, 2024):**
| Компонент | Цена |
|-----------|------|
| Шасси + материнская плата + БП | ~$4,000 |
| 2× Xeon Silver 4314 | ~$1,400 |
| 256 ГБ RAM | ~$1,600 |
| 12× 20TB HDD | ~$5,400 |
| 2× 3.2TB NVMe | ~$1,800 |
| 2× 25GbE + 2× 100GbE адаптеры | ~$2,500 |
| **Итого (1 узел):** | **~$16,700** |

**Почему R750xs, а не R750:**
- R750xs дешевле на ~20%
- Достаточно слотов PCIe для Ceph (R750 нужен для GPU/FPGA)
- Та же надёжность (Dell enterprise-grade)
- Поддержка iDRAC (удалённое управление, мониторинг)

#### Supermicro SuperServer 6029P-E1CR12H

**Назначение:** бюджетный 2U-сервер, популярный в российских ЦОД (параллельный импорт, доступность запчастей).

**Ключевые характеристики:**
| Параметр | Значение |
|----------|----------|
| Форм-фактор | 2U rack |
| Процессоры | 2× Intel Xeon Scalable 2-го поколения (Cascade Lake), до 28 ядер |
| Память | 12× DDR4-2933 RDIMM, до 3 ТБ (с Optane PMem) |
| Дисковые отсеки | 12× 3.5" hot-swap SAS3/SATA3 |
| NVMe | 2× NVMe U.2 (опциональный кабель OCuLink) |
| Сеть | 2× 10GbE (onboard, X722) + 2× 1GbE |
| RAID | Broadcom 3108 (SAS3) — **переключить в HBA-режим (JBOD)** |
| Управление | IPMI 2.0 + KVM-over-LAN |
| Блоки питания | 2× 1200 Вт (1+1 redundant) |

**Типовая конфигурация для Ceph (бюджетный вариант):**
```
Supermicro 6029P:
├── 2× Intel Xeon Gold 6226R (16C, 2.9 GHz) — 32 ядра
├── 192 ГБ RAM (12× 16GB DDR4-2933 RDIMM)
├── Storage:
│   ├── 2× 240GB SSD SATA (RAID1, ОС)
│   ├── 1× 1.6TB NVMe U.2 (WAL/DB, общий — рискованно!)
│   └── 12× 16TB SAS HDD (данные OSD)
├── Сеть:
│   ├── 2× 10GbE (onboard, public)
│   └── 2× 25GbE SFP28 (PCIe, cluster) — опционально, для экономии — общая 10GbE
├── RAID в HBA-режиме
└── IPMI
```

**Расчёт стоимости (ориентировочно, 2024):**
| Компонент | Цена |
|-----------|------|
| Шасси + материнская плата + БП | ~$3,000 |
| 2× Xeon Gold 6226R | ~$1,200 |
| 192 ГБ RAM | ~$1,000 |
| 12× 16TB HDD | ~$3,600 |
| 1× 1.6TB NVMe | ~$600 |
| Сетевые адаптеры | ~$800 |
| **Итого (1 узел):** | **~$10,200** |

**Плюсы и минусы:**
- Плюс: значительно дешевле Dell; доступность на вторичном рынке РФ
- Минус: меньше слотов NVMe; IPMI слабее iDRAC; меньшая плотность RAM

#### Сравнение платформ для разных сценариев

**Сценарий А: Enterprise, критичная инфраструктура → Dell R750xs**
- Контракт поддержки ProSupport (4-часовая замена)
- iDRAC Enterprise с расширенным мониторингом
- Сертифицированные совместимые компоненты
- BOSS-карта для ОС (отдельно от данных)

**Сценарий Б: Бюджетный, стартап/лаборатория → Supermicro 6029P**
- Нет контракта поддержки (свои инженеры, запас запчастей)
- IPMI базовый (достаточно для KVM)
- Можно собирать из компонентов (кастомизация)
- Риск: один NVMe для WAL/DB всех OSD

**Сценарий В: High-Density (максимум ТБ на 1U) → Supermicro 6049P-E1CR60H**
- 60× 3.5" отсеков в 4U (top-loading)
- 2× Xeon Scalable
- Идеально для холодного хранения / архивов на Ceph
- ~$20,000 с 60× HDD

#### Расчёт CPU для реальных конфигураций

**Конфигурация 1: Dell R750xs с 12 HDD + 2 NVMe**
```
OSD:      12 HDD × 1 ядро  = 12 ядер
WAL/DB:   2 NVMe × 0.5 ядра =  1 ядро (метаданные)
MON:      (выделенный)      =  0 ядер (на отдельных узлах)
MGR/MDS:  (выделенный)      =  0 ядер
Система:                    =  2 ядра
Резерв:                     =  3 ядра
                          ─────────
Итого:                      18 ядер

→ 2× Xeon Silver 4314 (16C) = 32 ядра — с запасом 1.8×
→ Можно добавить ещё 6 OSD без замены CPU
```

**Конфигурация 2: Supermicro 6029P с 12 HDD (экономия)**
```
OSD:      12 HDD × 1 ядро  = 12 ядер
Система:                    =  2 ядра
                          ─────────
Итого:                      14 ядер минимум

→ 2× Xeon Silver 4214 (12C) = 24 ядра — достаточно
→ Экономия ~$400 на CPU
```

#### Расчёт памяти для реальных конфигураций

**Dell R750xs, 12 HDD (WAL/DB на NVMe):**
```
OSD:         12 × 5 ГБ                    = 60 ГБ
BlueStore:   12 × 3 ГБ (кеш, HDD)         = 36 ГБ
MON/MGR:     на отдельных узлах            =  0 ГБ
Система:                                   =  8 ГБ
OS cache (рекомендация):                   = 16 ГБ
                                         ─────────
Итого:                                     120 ГБ

→ 256 ГБ RAM: 2.1× запас — комфортно
→ Минимально: 128 ГБ RAM (без запаса)
```

**Supermicro, 12 HDD (WAL/DB на единственном NVMe):**
```
OSD:         12 × 5 ГБ                    = 60 ГБ
BlueStore:   12 × 2 ГБ (SATA SSD-кеш)     = 24 ГБ
Система + резерв:                          = 12 ГБ
                                         ─────────
Итого:                                     96 ГБ

→ 192 ГБ RAM: 2× запас
→ Минимально: 128 ГБ RAM
```

#### Выбор сетевых адаптеров

| Модель | Скорость | Порты | Цена | Применение |
|--------|----------|-------|------|------------|
| Intel X710-DA2 | 10GbE | 2× SFP+ | ~$250 | Бюджетная public |
| Intel XXV710-DA2 | 25GbE | 2× SFP28 | ~$400 | Стандартная public/cluster |
| Mellanox ConnectX-6 Dx | 100GbE | 2× QSFP28 | ~$1,200 | High-perf cluster |
| Broadcom P225P | 25GbE | 2× SFP28 | ~$350 | Альтернатива Intel |

**Рекомендация:** для production используйте два двухпортовых адаптера:
- 2× 25GbE (bond) для public network
- 2× 25GbE (bond) для cluster network
- Разные PCIe-слоты (разные NUMA-узлы) для отказоустойчивости

Или для high-performance:
- 2× 25GbE (bond) для public
- 2× 100GbE (bond) для cluster

---

### 6.6. Детальное проектирование сети: схемы, VLAN-ы, QoS *(5 стр.)*

#### Полная сетевая схема production-кластера

```dot
digraph G {
    rankdir=TB;
    bgcolor="white";
    fontname="Arial";
    fontcolor="black";
    splines=ortho;
    compound=true;
    node [shape=box, style="filled,rounded", fontname="Arial", fontcolor="black"];
    edge [fontname="Arial", fontcolor="black"];

    clients [label="Клиенты\n(RBD, RGW, CephFS)", fillcolor="#F3E5F5"];
    cli_net [label="Сеть клиентов\n10.0.0.0/24", shape=box, style="filled", fillcolor="#E1BEE7"];

    haproxy [label="Балансировщик / HAProxy\n(RGW, Dashboard)", fillcolor="#E8EAF6"];

    pub_net [label="Public Network (front)\nVLAN 101: 10.0.1.0/24\n25GbE LACP bonds", shape=box, style="filled", fillcolor="#BBDEFB"];

    mon1 [label="MON-1\n.1.10", fillcolor="#90CAF9"];
    mon2 [label="MON-2\n.1.11", fillcolor="#90CAF9"];
    mon3 [label="MON-3\n.1.12", fillcolor="#90CAF9"];

    clu_net [label="Cluster Network (back)\nVLAN 102: 10.0.2.0/24\n25/100GbE LACP", shape=box, style="filled", fillcolor="#C8E6C9"];

    osd1 [label="OSD-1\n.2.20\n12× HDD\n2× NVMe", fillcolor="#A5D6A7"];
    osd2 [label="OSD-2\n.2.21\n12× HDD\n2× NVMe", fillcolor="#A5D6A7"];
    osd3 [label="OSD-3\n.2.22\n12× HDD\n2× NVMe", fillcolor="#A5D6A7"];

    mgmt_net [label="Management\nVLAN 999: 172.16.0.0/24", shape=box, style="filled", fillcolor="#FFE0B2"];

    idrac1 [label="iDRAC/IPMI\nOSD-1", fillcolor="#FFCC80"];
    idrac2 [label="iDRAC/IPMI\nOSD-2", fillcolor="#FFCC80"];
    idrac3 [label="iDRAC/IPMI\nOSD-3", fillcolor="#FFCC80"];

    {rank=same; mon1; mon2; mon3;}
    {rank=same; osd1; osd2; osd3;}
    {rank=same; idrac1; idrac2; idrac3;}

    clients -> cli_net [dir=none];
    cli_net -> haproxy;
    haproxy -> pub_net [dir=none];

    pub_net -> mon1 [dir=none];
    pub_net -> mon2 [dir=none];
    pub_net -> mon3 [dir=none];

    mon1 -> clu_net [dir=none];
    mon2 -> clu_net [dir=none];
    mon3 -> clu_net [dir=none];

    clu_net -> osd1 [dir=none];
    clu_net -> osd2 [dir=none];
    clu_net -> osd3 [dir=none];

    osd1 -> mgmt_net [dir=none];
    osd2 -> mgmt_net [dir=none];
    osd3 -> mgmt_net [dir=none];

    mgmt_net -> idrac1 [dir=none];
    mgmt_net -> idrac2 [dir=none];
    mgmt_net -> idrac3 [dir=none];
}
```

**VLAN-ы:**
1. **VLAN 101 (public):** клиентский трафик, MON-коммуникации, RGW, Dashboard
2. **VLAN 102 (cluster):** OSD-репликация, recovery, backfill, heartbeat
3. **VLAN 999 (management):** iDRAC/IPMI, SSH (опционально)

**Почему отдельный management VLAN:**
- Трафик IPMI не должен конкурировать с Ceph-трафиком
- Безопасность: management-сеть изолирована от production
- При проблемах с Ceph-сетью вы не теряете управление

#### Конфигурация коммутатора (Cisco Nexus / Arista / Juniper)

Пример для Cisco Nexus 9000 (25/100GbE):

```
! VLAN definitions
vlan 101
  name CEPH-PUBLIC
vlan 102
  name CEPH-CLUSTER
vlan 999
  name MGMT

! Port-channel (LACP) to Ceph Node 1
interface Ethernet1/1
  description CEPH-NODE1-PUBLIC-eth0
  switchport mode trunk
  switchport trunk allowed vlan 101
  channel-group 10 mode active
  mtu 9216    ! Jumbo Frames

interface Ethernet1/2
  description CEPH-NODE1-PUBLIC-eth1
  switchport mode trunk
  switchport trunk allowed vlan 101
  channel-group 10 mode active
  mtu 9216

interface Ethernet1/3
  description CEPH-NODE1-CLUSTER-eth2
  switchport mode trunk
  switchport trunk allowed vlan 102
  channel-group 20 mode active
  mtu 9216

! QoS: приоритет cluster-трафика
class-map type qos match-all CEPH-CLUSTER
  match cos 4    ! CoS 4 = высокий приоритет

policy-map type qos CEPH-QOS
  class CEPH-CLUSTER
    set qos-group 4
    bandwidth percent 60    ! Гарантированная полоса
  class class-default
    bandwidth percent 40
```

#### Расчёт пропускной способности

**Формула для расчёта необходимой пропускной способности сети:**

```
BW_public ≥ (суммарный_write_throughput × replication_factor) / oversubscription_ratio
BW_cluster ≥ суммарный_recovery_rate + backfill_rate
```

**Пример расчёта для кластера из 3 OSD-узлов (12 HDD × 20 ТБ):**

```
Сырая ёмкость:      3 × 12 × 20 = 720 ТБ
Полезная (×3 rep):  720 / 3 = 240 ТБ

Исходные данные:
- Один HDD = 250 MB/s последовательно
- Recovery rate (по умолчанию): 1 OSD может восстанавливаться на скорости ~50 MB/s
- При отказе одного узла: 12 OSD восстанавливаются → 12 × 50 = 600 MB/s = 4.8 Gbps

Пиковый recovery трафик: ~5 Gbps
+ Нормальный клиентский трафик: ~2 Gbps
─────────────────────────────────────────
Cluster network: минимум 10 Gbps, рекомендовано 25 Gbps

Public network:
- Клиентский трафик: 4 Gbps (пиковый)
- Рекомендовано: 10 Gbps (с запасом 2.5×)
```

**Правила выбора скорости сети:**

| Размер кластера | Public network | Cluster network |
|-----------------|---------------|-----------------|
| PoC (3 узла) | 10GbE | 10GbE (или общая) |
| Малый (3–5 узлов, HDD) | 10GbE | 25GbE |
| Средний (5–10 узлов, HDD) | 25GbE | 25GbE |
| Средний (SSD/NVMe) | 25GbE | 100GbE |
| Крупный (10+ узлов, NVMe) | 100GbE | 100GbE (или 2×100GbE) |

---

### 6.7. Стратегии CRUSH map для реальных сценариев *(4 стр.)*

CRUSH map — это «скелет», на котором держится размещение данных. Правильная CRUSH map обеспечивает отказоустойчивость на уровне серверов, стоек, дата-центров.

#### Scenario 1: Один DC, отказоустойчивость на уровне серверов

```bash
# CRUSH map: 3 копии, каждая на разном хосте
ceph osd crush rule create-replicated \
    ssd_replicated_host \
    default ssd host

# Убедиться, что bucket type = host
ceph osd crush tree
# -1  root default
# -3      host ceph-osd1
#  0          osd.0
#  1          osd.1
# -5      host ceph-osd2
#  2          osd.2
#  3          osd.3
```

**Проверка:** при отказе одного сервера все PG остаются доступными (копии на других серверах).

#### Scenario 2: Отказоустойчивость на уровне стоек (rack-aware)

```
DC1
├── Rack A
│   ├── ceph-osd-a1 (12 OSD)
│   └── ceph-osd-a2 (12 OSD)
├── Rack B
│   ├── ceph-osd-b1 (12 OSD)
│   └── ceph-osd-b2 (12 OSD)
└── Rack C
    ├── ceph-osd-c1 (12 OSD)
    └── ceph-osd-c2 (12 OSD)
```

```bash
# 1. Создать rack bucket-ы
ceph osd crush add-bucket Rack-A rack
ceph osd crush add-bucket Rack-B rack
ceph osd crush add-bucket Rack-C rack

# 2. Разместить их под default root
ceph osd crush move Rack-A root=default
ceph osd crush move Rack-B root=default
ceph osd crush move Rack-C root=default

# 3. Переместить хосты в свои стойки
ceph osd crush move ceph-osd-a1 rack=Rack-A
ceph osd crush move ceph-osd-a2 rack=Rack-A
ceph osd crush move ceph-osd-b1 rack=Rack-B
# ...

# 4. CRUSH rule: выбираем разные стойки
ceph osd crush rule create-replicated \
    rack_aware_rule \
    default ssd rack
```

**Эффект:** CRUSH гарантирует, что 3 копии размещены в **разных стойках**. При отказе питания всей стойки — данные доступны.

#### Scenario 3: Два дата-центра (site-aware)

```bash
# CRUSH hierarchy
ceph osd crush add-bucket DC1 datacenter
ceph osd crush add-bucket DC2 datacenter
ceph osd crush move DC1 root=default
ceph osd crush move DC2 root=default

# Размещаем стойки по DC
ceph osd crush move Rack-A datacenter=DC1
ceph osd crush move Rack-B datacenter=DC1
ceph osd crush move Rack-C datacenter=DC2

# Создаём rule (2 копии в DC1, 1 в DC2)
ceph osd crush rule create-replicated \
    site_aware_dc \
    default ssd datacenter
# Правило: выбираем 3 разных datacenter (но их всего 2!)
# Нужно кастомное CRUSH rule с take-действиями
```

**Кастомное CRUSH rule с гарантией размещения:**

```bash
# Получить текущий CRUSH map и декомпилировать
ceph osd getcrushmap -o crushmap.bin
crushtool -d crushmap.bin -o crushmap.txt

# Добавить правило (в crushmap.txt):
# rule site_aware {
#     id 10
#     type replicated
#     step take DC1
#     step chooseleaf firstn 2 type host
#     step emit
#     step take DC2
#     step chooseleaf firstn 1 type host
#     step emit
# }

# Скомпилировать и применить
crushtool -c crushmap.txt -o crushmap_new.bin
ceph osd setcrushmap -i crushmap_new.bin
```

**Итог:** 2 копии в DC1 (разные хосты), 1 копия в DC2. При падении DC1 — данные в DC2, но в read-only (нет кворума из 2 копий). Нужен третий DC или arbitrator для full HA.

#### Scenario 4: Erasure Coding с быстрым восстановлением

```bash
# EC 4+2: 4 data chunks + 2 parity, может пережить потерю 2 чанков
ceph osd erasure-code-profile set EC42 \
    k=4 m=2 \
    crush-failure-domain=host

# Пул с EC профилем
ceph osd pool create cold-storage erasure EC42
```

**Экономия места:** 4+2 означает overhead 50% (вместо 200% для ×3 репликации).
**Подходит для:** холодных данных, бэкапов, архивов, где latency не критична.

---

### 6.8. Контрольные вопросы к главе 6

1. **Почему для Ceph важно количество ядер, а не частота CPU?**
2. **Рассчитайте необходимый объём RAM для сервера с 24 HDD-OSD и выделенными MON-узлами.**
3. **Сравните Dell R750xs и Supermicro 6029P: в каком сценарии вы выберете каждую платформу и почему?**
4. **Почему рекомендуется разделять public и cluster сети? Что произойдёт, если этого не сделать?**
5. **Спроектируйте топологию для кластера из 5 узлов с 12 HDD на каждом, SSD-only пулом и общим объёмом 200 ТБ полезных данных. Укажите: CPU, RAM, сеть, CRUSH rule.**
6. **Зачем нужен Jumbo Frames (MTU 9000) в Ceph-сети? В каком случае Jumbo Frames НЕ дадут выигрыша?**
7. **Опишите стратегию WAL/DB для сервера с 24 HDD. Сколько NVMe-дисков нужно, какого размера?**
8. **Спроектируйте rack-aware CRUSH map для 3 стоек по 2 сервера в каждой. Почему rack-aware важнее host-aware?**
9. **Какие требования к сети для синхронной репликации между двумя дата-центрами? Что будет при превышении latency?**
10. **Рассчитайте стоимость кластера из 3 Dell R750xs с 12 HDD и 2 NVMe каждый. Сравните со стоимостью аналогичного на Supermicro. Какая конфигурация «дешевле за терабайт»?**

---

### 6.9. Практикум: проектируем кластер *(3 стр.)*

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

#### Полный каталог флагов `cephadm bootstrap`

`cephadm bootstrap` принимает десятки параметров. Знание каждого флага критически важно для production-развёртывания, особенно в нестандартных средах (замкнутый контур, кастомные сети, корпоративные политики безопасности).

**Основные (обязательные и часто используемые):**

| Флаг | Значение | Пример |
|------|----------|--------|
| `--mon-ip` | IP-адрес первого MON (ОБЯЗАТЕЛЬНЫЙ) | `--mon-ip 10.0.1.10` |
| `--mon-addrv` | Адрес MON в формате `[v2:][v1:]` (IPv6) | `--mon-addrv '[v2:10.0.1.10:3300,v1:10.0.1.10:6789]'` |
| `--fsid` | Задать конкретный FSID (для восстановления) | `--fsid 51fa3f5c-...` |
| `--cluster-network` | Подсеть для cluster (OSD-OSD) трафика | `--cluster-network 10.0.2.0/24` |
| `--output-dir` | Куда сохранять ключи и конфиги | `--output-dir /root/ceph-bootstrap` |
| `--output-keyring` | Путь для admin keyring | `--output-keyring /etc/ceph/admin.keyring` |
| `--output-config` | Путь для ceph.conf | `--output-config /etc/ceph/ceph.conf` |

**Сеть и адреса:**

| Флаг | Значение |
|------|----------|
| `--public-network` | CIDR public-сети |
| `--msbind-ipv4` | Привязаться только к IPv4 (по умолчанию true) |
| `--msbind-ipv6` | Привязаться только к IPv6 |
| `--skip-mon-network` | Не проверять доступность сети MON |
| `--skip-dashboard` | Не разворачивать Dashboard |
| `--dashboard-password-noupdate` | Не менять пароль dashboard (оставить из старой инсталляции) |

**Контейнеры и образы:**

| Флаг | Значение |
|------|----------|
| `--image` | Образ Ceph (локальный registry) | `--image registry.local:5000/ceph/ceph:v19.2.4` |
| `--no-container-init` | НЕ скачивать образ при bootstrap (для air-gapped) |
| `--container-init` | Принудительно скачать образ (по умолчанию) |
| `--docker` | Использовать Docker вместо Podman |
| `--no-minimize-config` | Не минимизировать ceph.conf (сохранить все секции) |

**Безопасность и аутентификация:**

| Флаг | Значение |
|------|----------|
| `--ssh-user` | Пользователь для SSH на узлах (по умолчанию root) |
| `--ssh-private-key` | Путь к приватному SSH-ключу |
| `--ssh-public-key` | Путь к публичному SSH-ключу |
| `--initial-dashboard-user` | Имя пользователя Dashboard (по умолчанию admin) |
| `--initial-dashboard-password` | Задать пароль Dashboard вручную |
| `--ssl-dashboard-port` | Порт Dashboard (по умолчанию 8443) |

**Восстановление и обслуживание:**

| Флаг | Значение |
|------|----------|
| `--skip-prepare-host` | Пропустить подготовку хоста (установку пакетов) |
| `--skip-pull` | Не тянуть образы (уже есть локально) |
| `--skip-firewalld` | Не настраивать firewalld |
| `--allow-fqdn-hostname` | Разрешить FQDN-имена узлов |
| `--orphan-initial-daemons` | Не добавлять начальные демоны в оркестратор (для тестов) |
| `--single-host-defaults` | Оптимизировать настройки для одноузлового кластера |

**Примеры использования:**

```bash
# Минимальный (лабораторный) bootstrap
cephadm bootstrap --mon-ip 10.0.1.10

# Production bootstrap с раздельными сетями, кастомным registry и dashboard
cephadm --image registry.local:5000/ceph/ceph:v19.2.4 \
  bootstrap \
  --mon-ip 10.0.1.10 \
  --cluster-network 10.0.2.0/24 \
  --initial-dashboard-user cephadmin \
  --initial-dashboard-password 'Str0ngP@ssw0rd!' \
  --allow-fqdn-hostname \
  --skip-firewalld

# Air-gapped (замкнутый контур)
cephadm --image registry.local:5000/ceph/ceph:v19.2.4 \
  --no-container-init \
  bootstrap \
  --mon-ip 10.0.1.10 \
  --skip-mon-network \
  --skip-dashboard \
  --output-dir /etc/ceph

# Восстановление кластера (есть старый FSID и ключи)
cephadm bootstrap \
  --mon-ip 10.0.1.10 \
  --fsid 51fa3f5c-a1b2-... \
  --output-dir /etc/ceph
```

#### Глубокий разбор: что именно делает bootstrap внутри

**Фаза 0: Preflight (проверки до запуска)**

```python
# Упрощённая логика cephadm bootstrap (Python):
def _check_preflight():
    # 1. Проверить права (нужен root)
    if os.geteuid() != 0:
        raise Error("bootstrap requires root privileges")

    # 2. Проверить контейнерный движок
    container_engine = detect_container_engine()  # podman или docker
    assert container_engine is not None

    # 3. Проверить синхронизацию времени
    assert time_sync_is_active()  # chrony или ntpd

    # 4. Проверить разрешение имён
    assert hostname_is_resolvable()

    # 5. Проверить IP (должен быть локальным)
    assert mon_ip_is_local(mon_ip)

    # 6. Проверить порты (не заняты ли 3300, 6789, 8443)
    assert ports_are_free()
```

**Фаза 1: Генерация криптографии**

```bash
# FSID — UUID4, уникальный идентификатор кластера
# Генерируется: uuidgen или python -c "import uuid; print(uuid.uuid4())"

# Monitor keyring — секретный ключ для MON-коммуникаций
ceph-authtool --create-keyring /etc/ceph/ceph.mon.keyring \
    --gen-key -n mon.

# Admin keyring — клиентский ключ с полными правами
ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring \
    --gen-key -n client.admin \
    --cap mon 'allow *' \
    --cap osd 'allow *' \
    --cap mds 'allow *' \
    --cap mgr 'allow *'

# Bootstrap keyrings — для добавления новых узлов
# cephadm использует их при ceph orch host add
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-osd/ceph.keyring \
    --gen-key -n client.bootstrap-osd
ceph-authtool --create-keyring /var/lib/ceph/bootstrap-rgw/ceph.keyring \
    --gen-key -n client.bootstrap-rgw
# ... и ещё ~10 bootstrap-ключей для разных сервисов
```

**Фаза 2: Первый Monmap и MON**

```bash
# Monmap — карта мониторов (кто, где, какая эпоха выборов)
monmaptool --create --addv mon1 '[v2:10.0.1.10:3300,v1:10.0.1.10:6789]' \
    --fsid 51fa3f5c-... \
    --clobber /etc/ceph/monmap

# Запуск первого MON (внутри контейнера):
# podman run ... quay.io/ceph/ceph:v19.2.4 \
#     ceph-mon --mkfs -i mon1 --monmap /etc/ceph/monmap \
#              --keyring /etc/ceph/ceph.mon.keyring
# Это создаёт mon-хранилище в /var/lib/ceph/mon/ceph-mon1/
```

**Фаза 3: Конфигурация кластера**

```ini
# /etc/ceph/ceph.conf (после bootstrap):
[global]
fsid = 51fa3f5c-a1b2-c3d4-e5f6-1234567890ab
mon_host = [v2:10.0.1.10:3300,v1:10.0.1.10:6789]
mon_initial_members = mon1
public_network = 10.0.1.0/24
auth_cluster_required = cephx
auth_service_required = cephx
auth_client_required = cephx

# Если указан --cluster-network:
cluster_network = 10.0.2.0/24

# Менеджер:
[mgr]
mgr/cephadm/container_image_ceph = quay.io/ceph/ceph:v19.2.4
```

**Фаза 4: Первый MGR и модули**

```bash
# Запуск MGR:
# podman run ... ceph-mgr -n mgr.mon1.abc123 -f

# Активация модулей:
ceph mgr module enable prometheus
ceph mgr module enable dashboard
ceph mgr module enable restful
ceph mgr module enable iostat
ceph mgr module enable crash
ceph mgr module enable devicehealth
ceph mgr module enable orchestrator

# Настройка Dashboard:
ceph dashboard create-self-signed-cert
ceph dashboard set-login-credentials admin <password>
```

**Фаза 5: Systemd units и финализация**

```bash
# Для каждого сервиса создаётся systemd unit:
# /etc/systemd/system/ceph-51fa3f5c@.service     (шаблон)
# /etc/systemd/system/ceph-51fa3f5c@mon.mon1.service
# /etc/systemd/system/ceph-51fa3f5c@mgr.mon1.abc123.service

# Unit-файл содержит команду запуска контейнера:
# ExecStart=/usr/bin/podman run --rm --name ceph-51fa3f5c-mon.mon1 ...
#     -v /var/lib/ceph/...:/var/lib/ceph:z
#     quay.io/ceph/ceph:v19.2.4

# После создания — enable и start:
systemctl enable ceph-51fa3f5c@mon.mon1.service
systemctl start ceph-51fa3f5c@mon.mon1.service
```

**Чек-лист: что проверить после bootstrap:**

```bash
# 1. MON работает
ceph mon stat
# e1: 1 mons at {mon1=[v2:10.0.1.10:3300,v1:10.0.1.10:6789]}

# 2. MGR работает
ceph mgr stat
# {"active_name":"mon1.abc123"}

# 3. HEALTH_WARN (нормально — кластер ещё не готов)
ceph health
# HEALTH_WARN: OSD count 0 < osd_pool_default_size 3

# 4. Ключи на месте
ls -la /etc/ceph/ceph.client.admin.keyring
ls -la /var/lib/ceph/bootstrap-*/

# 5. Dashboard доступен
curl -k https://10.0.1.10:8443

# 6. Systemd units
systemctl list-units 'ceph-*'
```

#### Секреты bootstrap: что нельзя терять

После bootstrap создаются файлы, потеря которых делает невозможным восстановление/расширение кластера:

| Файл | Назначение | Что будет при потере |
|------|-----------|---------------------|
| `/etc/ceph/ceph.client.admin.keyring` | Полный доступ к кластеру | Не сможете выполнять `ceph` команды |
| `/var/lib/ceph/bootstrap-osd/ceph.keyring` | Ключ для добавления OSD | Не сможете добавлять новые OSD через cephadm |
| `/var/lib/ceph/bootstrap-rgw/ceph.keyring` | Ключ для RGW | Не сможете развернуть RGW |
| `/var/lib/ceph/mon/ceph-mon1/keyring` | Ключ MON | MON не запустится после перезагрузки |
| `/etc/ceph/ceph.conf` | Конфигурация кластера | ceph-команды не смогут подключиться |

**Рекомендация:** скопируйте `/etc/ceph/` и `/var/lib/ceph/bootstrap-*/` в защищённое хранилище (Vault, зашифрованный архив) сразу после bootstrap. Лучше автоматизировать через Ansible:

```yaml
- name: Бэкап Ceph-ключей после bootstrap
  hosts: bootstrap_node
  tasks:
    - name: Создать архив ключей
      archive:
        path:
          - /etc/ceph
          - /var/lib/ceph/bootstrap-*
        dest: /root/ceph-keys-{{ ansible_date_time.date }}.tar.gz
        format: gz

    - name: Скопировать архив в безопасное хранилище
      fetch:
        src: /root/ceph-keys-{{ ansible_date_time.date }}.tar.gz
        dest: /backup/ceph-keys/
        flat: yes
```

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

#### Расширенные примеры Service Spec

**Пример 1: RadosGW multi-site**

```yaml
service_type: rgw
service_id: rgw.dc1
placement:
  hosts:
    - ceph-rgw1
    - ceph-rgw2
spec:
  rgw_realm: multisite
  rgw_zonegroup: production
  rgw_zone: dc1
  rgw_frontend_port: 80
  rgw_frontend_ssl_port: 443
  rgw_frontend_ssl_certificate: |
    -----BEGIN CERTIFICATE-----
    MIIDXTCCAkWgAwIBAgIJAL...
    -----END CERTIFICATE-----
  rgw_frontend_ssl_key: |
    -----BEGIN PRIVATE KEY-----
    MIIEvgIBADANBgkqhki...
    -----END PRIVATE KEY-----
networks:
  - 10.0.1.0/24   # public-only, не слушать на cluster
---
service_type: rgw
service_id: rgw.dc2
placement:
  hosts:
    - ceph-rgw3
    - ceph-rgw4
spec:
  rgw_realm: multisite
  rgw_zonegroup: production
  rgw_zone: dc2
  rgw_frontend_port: 80
```

**Пример 2: NFS-Ganesha (экспорт CephFS через NFS)**

```yaml
service_type: nfs
service_id: nfs-cluster
placement:
  count: 2
  label: nfs
spec:
  port: 2049
  # Создаём NFS-экспорт для CephFS
---
# Затем создать экспорт:
# ceph nfs export create cephfs \
#   --cluster-id nfs-cluster \
#   --pseudo-path /cephfs \
#   --fsname cephfs
```

**Пример 3: SNMP-мониторинг**

```yaml
service_type: snmp-gateway
service_id: snmp
placement:
  hosts:
    - ceph-mon1
spec:
  snmp_destination: 172.16.0.100:162
  snmp_version: V2c
  snmp_community: public
  snmp_engine_id: "80001f8880a1b2c3d4"
```

**Пример 4: Prometheus + Grafana stack**

```yaml
service_type: prometheus
service_id: prometheus
placement:
  count: 1
spec:
  retention_time: "90d"
  retention_size: "100GB"
---
service_type: grafana
service_id: grafana
placement:
  count: 1
spec:
  protocol: https
  initial_admin_password: "admin123"
---
service_type: node-exporter
service_id: node-exporter
placement:
  host_pattern: "*"   # на всех узлах
---
service_type: alertmanager
service_id: alertmanager
placement:
  count: 1
spec:
  user_data: |
    route:
      receiver: 'default'
    receivers:
      - name: 'default'
        webhook_configs:
          - url: 'http://alerts.local:9093/alert'
```

**Пример 5: OSD с фильтрацией по размеру и модели диска**

```yaml
service_type: osd
service_id: hdd_osds
placement:
  host_pattern: '*'
spec:
  data_devices:
    rotational: 1          # только вращающиеся (HDD)
    size: '10TB:'          # только диски от 10 ТБ и более
  db_devices:
    model: 'INTEL SSDPE2KE032T8'   # конкретная модель NVMe для DB/WAL
  filter_logic: AND
  osds_per_device: 1
---
service_type: osd
service_id: ssd_osds
placement:
  host_pattern: '*'
spec:
  data_devices:
    rotational: 0          # только SSD/NVMe
    size: ':'              # все размеры
---
service_type: osd
service_id: nvme_perf
placement:
  hosts:
    - ceph-osd-perf1
    - ceph-osd-perf2
spec:
  data_devices:
    model: 'SAMSUNG MZWLR3T8HBLS'   # конкретные NVMe для high-perf
  db_devices:
    size: '400GB:'
  osds_per_device: 1
```

**Пример 6: MDS с настройками кеша**

```yaml
service_type: mds
service_id: cephfs
placement:
  count: 2
spec:
  mds_cache_memory_limit: 8589934592   # 8 ГБ кеша метаданных
  mds_cache_size: 500000               # максимум inode в кеше
  mds_standby_replay: true             # standby replay для быстрого failover
```

**Пример 7: Ingress (HAProxy + Keepalived) для RGW**

```yaml
service_type: ingress
service_id: rgw.ingress
placement:
  hosts:
    - ceph-rgw1
    - ceph-rgw2
spec:
  backend_service: rgw.rgw.dc1
  frontend_port: 80
  monitor_port: 1967
  virtual_ip: 10.0.1.100/24
  virtual_interface_networks:
    - 10.0.1.0/24
```

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

### 8.6. Устранение неисправностей cephadm *(6 стр.)*

cephadm — это сложный оркестратор с множеством движущихся частей. Типичные проблемы и методы их решения сэкономят вам часы отладки.

#### Проблема 1: `ceph orch host add` завершается ошибкой SSH

```bash
# Симптом:
# Error: Failed to connect to ceph-osd1 (10.0.1.20): ...
# Permission denied (publickey,password)

# Причины и решения:
# 1. SSH-ключ не скопирован
ssh-copy-id root@10.0.1.20

# 2. SSH работает не на 22 порту
ceph orch host add ceph-osd1 10.0.1.20 --labels osd
# cephadm всегда использует порт 22!

# 3. Проблема с правами
# cephadm копирует свой pubkey:
cat /etc/ceph/ceph.pub
# Должен быть в /root/.ssh/authorized_keys на целевом узле

# 4. Проверка вручную:
ssh -i /etc/ceph/ceph.pub root@10.0.1.20 'echo OK'
```

#### Проблема 2: Контейнер не запускается (systemd unit в failed)

```bash
# Симптом:
# ceph orch ps | grep -E 'error|stopped'
# osd.5  ceph-osd2  error  <unknown>

# Диагностика:
# 1. Посмотреть юнит
systemctl status ceph-$(ceph fsid)@osd.5.service

# 2. Логи контейнера
journalctl -u ceph-$(ceph fsid)@osd.5.service -n 200 --no-pager

# 3. Логи внутри контейнера
cephadm logs --name osd.5

# Типичные причины:
# - Диск не найден (после замены/переподключения)
# - Проблема с LVM (vg/lv отсутствует)
# - OOM kill (проверить dmesg)
# - Образ контейнера повреждён

# Решения:
# 1. Перезапустить
ceph orch daemon restart osd.5

# 2. Пересоздать (если диск исправен)
ceph orch daemon rm osd.5 --force
ceph orch daemon add osd ceph-osd2:/dev/sdf

# 3. Проверить LVM
pvs; vgs; lvs  # На OSD-узле
```

#### Проблема 3: Оркестратор «завис» (reconciliation loop не работает)

```bash
# Симптом:
# ceph orch ps показывает устаревшую информацию
# Сервисы не создаются, хотя spec применён

# Диагностика:
# 1. Статус оркестратора
ceph orch status
# Backend: cephadm
# Available: True
# Paused: False

# 2. Принудительный запуск reconciliation
ceph orch device ls            # Обновить список устройств
ceph mgr module disable cephadm
ceph mgr module enable cephadm

# 3. Перезапустить MGR
ceph orch daemon restart mgr.$(hostname).*

# 4. Проверить логи MGR
cephadm logs --name mgr.$(hostname).*
# Или на хосте:
journalctl -u ceph-$(ceph fsid)@mgr.* -f

# 5. Если ничего не помогает — пауза/снятие с паузы
ceph orch pause
sleep 30
ceph orch resume
```

#### Проблема 4: Образ контейнера не загружается

```bash
# Симптом:
# Error: unable to pull image quay.io/ceph/ceph:v19.2.4

# Причины:
# 1. Нет доступа в интернет (возможно, нужен прокси)
# 2. Registry заблокирован (корпоративный firewall)
# 3. Не хватает места на диске

# Решения:
# 1. Установить прокси для Podman
mkdir -p /etc/systemd/system/podman.service.d/
cat > /etc/systemd/system/podman.service.d/http-proxy.conf <<EOF
[Service]
Environment="HTTP_PROXY=http://proxy.corp:8080"
Environment="HTTPS_PROXY=http://proxy.corp:8080"
Environment="NO_PROXY=localhost,127.0.0.1,10.0.0.0/8"
EOF
systemctl daemon-reload
systemctl restart podman

# 2. Использовать локальный registry (если настроен)
ceph config set mgr mgr/cephadm/container_image_ceph \
    registry.local:5000/ceph/ceph:v19.2.4

# 3. Проверить свободное место
df -h /var/lib/containers

# 4. Очистить старые образы
podman image prune -a -f
```

#### Проблема 5: `HEALTH_WARN: clock skew detected on mon.X`

```bash
# Симптом:
# mon.ceph-mon2 clock skew 0.08s > max 0.05s

# Причины:
# - chrony/ntpd не настроен или остановился
# - Разные NTP-серверы на разных узлах
# - Виртуальные машины с разным временем хоста

# Решения:
# 1. Проверить chrony на ВСЕХ узлах
ansible all -m shell -a 'chronyc tracking | grep -E "Offset|Stratum"'

# 2. Временное решение (увеличить допуск)
ceph config set mon mon_clock_drift_allowed 0.100  # 100ms

# 3. Правильное: синхронизировать время
# На всех узлах:
systemctl restart chrony
chronyc -a makestep

# 4. В замкнутом контуре — настроить локальный NTP
# На bootstrap-узле:
cat > /etc/chrony/chrony.conf <<EOF
local stratum 10
allow 10.0.0.0/8
EOF
systemctl restart chrony
# На остальных:
echo 'server 10.0.1.10 iburst' >> /etc/chrony/chrony.conf
systemctl restart chrony
```

#### Проблема 6: Dashboard недоступен (SSL/сеть)

```bash
# Симптом:
# curl -k https://ceph-mon1:8443 → Connection refused

# Диагностика:
# 1. Проверить, что сервис запущен
ceph orch ps --daemon-type mgr

# 2. Проверить порт
ss -tlnp | grep 8443

# 3. Проверить SSL-сертификат
ceph dashboard get-ssl-certificate

# 4. Сбросить пароль
ceph dashboard set-login-credentials admin NewP@ssw0rd!

# 5. Отключить SSL (для отладки)
ceph config set mgr mgr/dashboard/ssl false
ceph mgr module disable dashboard
ceph mgr module enable dashboard
# Порт станет 8080 (HTTP)

# 6. Проверить iptables/firewalld
iptables -L -n | grep 8443
# Или временно отключить:
systemctl stop firewalld
```

#### Проблема 7: Добавление OSD зависает (LVM/zap)

```bash
# Симптом:
# ceph orch daemon add osd ceph-osd1:/dev/sdb
# Висит > 5 минут, потом ошибка

# Причины:
# - На диске есть существующая ФС/LVM
# - Диск используется (смонтирован)
# - Диск в состоянии hardware error

# Решения:
# 1. Проверить, что диск не смонтирован
lsblk /dev/sdb
mount | grep sdb

# 2. Принудительно очистить (ZAP!)
# ВНИМАНИЕ: это уничтожит все данные
cephadm shell -- ceph-volume lvm zap --destroy /dev/sdb

# 3. Если ZAP не помогает — вручную:
wipefs -a /dev/sdb
dd if=/dev/zero of=/dev/sdb bs=1M count=100

# 4. Проверить smart
smartctl -a /dev/sdb | grep -i error
```

#### Проблема 8: Узлы в статусе `OFFLINE` / `MAINTENANCE`

```bash
# Симптом:
# ceph orch host ls
# ceph-osd1  10.0.1.20  osd  Offline

# Проверка:
# 1. Узел доступен по SSH?
ssh ceph-osd1 'cephadm ls'

# 2. Сервис cephadm-agent запущен?
ssh ceph-osd1 'systemctl status cephadm-agent'

# 3. Время синхронизировано?
ssh ceph-osd1 'chronyc tracking'

# Решения:
# 1. Перезапустить агент
ssh ceph-osd1 'systemctl restart cephadm-agent'

# 2. Принудительно вернуть в online
ceph orch host maintenance enter ceph-osd1 --force
ceph orch host maintenance exit ceph-osd1

# 3. Переустановить cephadm на узле (крайний случай)
ssh ceph-osd1 'cephadm rm-cluster --fsid $(ceph fsid) --force'
# Затем снова добавить:
ceph orch host add ceph-osd1 10.0.1.20 --labels osd
```

#### Чек-лист отладки любой проблемы cephadm

```
1. [ ] ceph health detail           — полное состояние кластера
2. [ ] ceph -s                      — краткий статус
3. [ ] ceph orch ps                 — все сервисы и их статусы
4. [ ] ceph orch host ls            — все узлы
5. [ ] ceph orch ls                 — все применённые specs
6. [ ] ceph orch status             — статус оркестратора
7. [ ] ceph config log              — последние изменения конфигурации
8. [ ] ceph crash stat              — crash-дампы
9. [ ] cephadm logs --name <svc>    — логи конкретного сервиса
10.[ ] journalctl -u ceph-* -n 200   — логи systemd
```

---

### 8.7. Миграция кластеров: сценарии и процедуры *(6 стр.)*

Миграция между разными инструментами развёртывания или между мажорными версиями — одна из сложнейших задач в эксплуатации Ceph.

#### Сценарий 1: ceph-deploy → cephadm

Исходное состояние: кластер развёрнут через `ceph-deploy` (пакетная установка, без контейнеров), Ceph Octopus 15.2.x.

```bash
# Шаг 1: Обновить до Pacific 16.2.x через ceph-deploy (последний релиз, где он работает)
# На admin-узле:
ceph-deploy install --release pacific ceph-mon1 ceph-mon2 ...
ceph-deploy upgrade ceph-mon1 ceph-mon2 ...

# Шаг 2: Перейти на cephadm
# На одном из MON-узлов:
cephadm adopt \
  --style legacy \
  --name mon.$(hostname -s) \
  --cluster ceph

# Шаг 3: Принять (adopt) остальные сервисы
cephadm adopt --style legacy --name mon.mon2
cephadm adopt --style legacy --name mgr.mon1
# Для OSD:
cephadm adopt --style legacy --name osd.0
cephadm adopt --style legacy --name osd.1
# ...

# Шаг 4: Проверить
ceph orch ps
ceph orch host ls

# Шаг 5: Удалить старые systemd unit-ы
systemctl stop ceph-mon@mon1
systemctl disable ceph-mon@mon1
systemctl stop ceph-osd@0
systemctl disable ceph-osd@0
# ... все старые юниты

# Шаг 6: Обновить до Quincy/Reef/Squid
ceph orch upgrade start --ceph-version 19.2.4
```

#### Сценарий 2: ceph-ansible → cephadm

Исходное состояние: кластер через ceph-ansible, Quincy 17.2.x.

```bash
# Рекомендация: сначала обновить ceph-ansible до Reef, затем переходить на cephadm.
# Прямой переход сложен и рискован.

# Шаг 1: Оценить состояние
ceph versions
ceph health

# Шаг 2: Установить cephadm на все узлы
ansible all -m shell -a 'apt install cephadm -y'
# Или скопировать скрипт:
curl --silent --remote-name --location \
    https://download.ceph.com/rpm-reef/el9/noarch/cephadm
chmod +x cephadm

# Шаг 3: Создать временный кластер cephadm с новым FSID
# Не делайте bootstrap — используйте adopt

# Шаг 4: Сгенерировать конфигурацию cephadm на базе существующей
cephadm adopt \
  --style legacy \
  --name mon.$(hostname -s)

# Шаг 5: Мигрировать хосты и сервисы
# Этот процесс интерактивный и зависит от конкретной установки.
# Лучше планировать с инженером Ceph.

# Альтернатива (менее рискованная): дождаться EOL текущей версии,
# развернуть НОВЫЙ кластер на cephadm и мигрировать данные через RBD mirror
# или rados cppool.
```

#### Сценарий 3: Миграция данных между кластерами (пул → пул)

Используется при переходе на новое оборудование или новую архитектуру.

```bash
# Метод 1: RADOS copy-pool (блочная миграция)
# На старом кластере:
ceph osd pool create target-pool 128 128
# Настроить репликацию pool-level
# (Исторический метод, сейчас редко используется)

# Метод 2: RBD mirror (для виртуализации)
# На старом кластере:
rbd mirror pool enable source-pool image
rbd mirror pool peer add source-pool client.bootstrap@new-cluster

# Создать образы на новом кластере
rbd mirror image enable source-pool/image1 snapshot

# После синхронизации — переключить клиентов на новый кластер

# Метод 3: Ручной экспорт/импорт (малые объёмы)
# Экспорт
rbd export pool/image1 /mnt/export/image1.raw
rados get -p pool obj1 /mnt/export/obj1.dat

# Импорт на новом кластере
rbd import /mnt/export/image1.raw pool/image1
rados put -p pool obj1 /mnt/export/obj1.dat

# Метод 4: RGW multisite sync (S3-данные)
# Настройка multisite между старым и новым кластером
# После полной синхронизации — отключить старый zone
```

#### Сценарий 4: Миграция на новое оборудование без простоя

```bash
# План (0 downtime, требует запасных слотов в стойке):

# Фаза 1: Добавить новые узлы в кластер
ceph orch host add new-osd1 10.0.1.30 --labels osd
ceph orch host add new-osd2 10.0.1.31 --labels osd
ceph orch host add new-osd3 10.0.1.32 --labels osd

# Развернуть OSD на новых узлах
ceph orch apply osd --all-available-devices

# Фаза 2: Дождаться перебалансировки
ceph -s  # HEALTH_OK

# Фаза 3: Установить noout (чтобы не было recovery при выводе старых OSD)
ceph osd set noout

# Фаза 4: Вывести старые OSD по одному
for osd in $(ceph osd ls-tree old-osd1); do
    ceph osd out $osd
    sleep 60
done
# Дождаться HEALTH_OK после каждого OSD

# Фаза 5: Удалить старые OSD
for osd in $(ceph osd ls-tree old-osd1); do
    ceph orch daemon rm osd.$osd --force
done

# Фаза 6: Удалить старые узлы
ceph orch host rm old-osd1
ceph orch host rm old-osd2
ceph orch host rm old-osd3

# Фаза 7: Снять noout
ceph osd unset noout

# Итог: данные переехали на новое оборудование без потери доступности.
# Время миграции зависит от объёма данных и скорости recovery.
```

#### Сценарий 5: Экстренное восстановление MON (потеря кворума)

```bash
# Ситуация: 2 из 3 MON упали безвозвратно. Кластер в режиме readonly.

# Шаг 1: Определить живые MON
ceph mon stat  # Ответит только если кворум есть
# Если нет — смотреть на узлах: systemctl status ceph-*@mon.*

# Шаг 2: Если живых MON нет (все упали) — извлечь monmap из OSD
# На любом узле с OSD:
cephadm shell -- ceph-monstore-tool /var/lib/ceph/mon/ceph-mon1/ get monmap \
    --keyring /var/lib/ceph/osd/ceph-0/keyring \
    --osd-fsid $(cat /var/lib/ceph/osd/ceph-0/fsid)

# Шаг 3: Собрать кворум вручную
# Создать новый monmap с одним MON
monmaptool --create --addv mon1 10.0.1.10:3300 --fsid <FSID> /tmp/monmap
# Скопировать на узел
ceph-mon --mkfs -i mon1 --monmap /tmp/monmap --keyring /etc/ceph/ceph.mon.keyring
# Запустить
systemctl start ceph-<FSID>@mon.mon1

# Шаг 4: Добавить новые MON
ceph mon add mon2 10.0.1.11
ceph mon add mon3 10.0.1.12
```

---

### 8.8. Обновление: `cephadm upgrade` — углублённый разбор *(4 стр.)*

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

### 9.5. Полные Ansible playbook для air-gapped *(6 стр.)*

#### Playbook 1: Подготовка мастер-узла (с интернетом)

Этот playbook выполняется **один раз** на машине, имеющей доступ в Интернет. Создаёт локальные зеркала пакетов и контейнеров для переноса в замкнутый контур.

```yaml
# ansible/prepare-air-gapped-master.yml
- name: Подготовка мастер-узла для air-gapped Ceph
  hosts: airgap-master
  become: yes
  vars:
    ceph_version: "19.2.4"
    ubuntu_release: "noble"
    registry_port: 5000
    aptly_port: 8080
    repo_base: "/srv/repos"
    registry_base: "/opt/registry"

  tasks:
    - name: Установка базовых пакетов
      apt:
        name:
          - aptly
          - podman
          - nginx
          - createrepo-c
          - wget
          - curl
        state: present
        update_cache: yes

    - name: Создание структуры каталогов
      file:
        path: "{{ item }}"
        state: directory
        mode: '0755'
      loop:
        - "{{ repo_base }}/ubuntu"
        - "{{ repo_base }}/ceph"
        - "{{ registry_base }}/data"
        - "{{ registry_base }}/certs"

    # === Зеркало DEB-пакетов через Aptly ===
    - name: Создание зеркала Ubuntu 24.04
      command:
        cmd: aptly mirror create
          -architectures=amd64
          ubuntu-{{ ubuntu_release }}
          http://archive.ubuntu.com/ubuntu {{ ubuntu_release }} main universe restricted
      args:
        creates: /root/.aptly/db  # Пропустить, если уже создано

    - name: Обновление зеркала Ubuntu
      command: aptly mirror update ubuntu-{{ ubuntu_release }}
      async: 3600
      poll: 30
      register: mirror_update
      # Это долгая операция (~20-30 минут)

    - name: Создание зеркала Ceph
      command:
        cmd: aptly mirror create
          -architectures=amd64
          ceph-squid
          https://download.ceph.com/debian-squid {{ ubuntu_release }} main
      args:
        creates: /root/.aptly/db

    - name: Обновление зеркала Ceph
      command: aptly mirror update ceph-squid

    - name: Создание снапшотов
      command: "aptly snapshot create {{ item }} from mirror {{ item }}"
      loop:
        - "ubuntu-{{ ubuntu_release }}"
        - ceph-squid

    - name: Публикация снапшотов (HTTP-репозиторий)
      command: >
        aptly publish snapshot
        -distribution={{ ubuntu_release }}
        ubuntu-{{ ubuntu_release }} ceph-squid
      args:
        creates: /root/.aptly/public

    - name: Настройка Nginx для раздачи репозитория
      copy:
        dest: /etc/nginx/sites-available/aptly
        content: |
          server {
              listen {{ aptly_port }};
              server_name _;
              root /root/.aptly/public;
              location / {
                  autoindex on;
              }
          }

    - name: Активация сайта Nginx
      file:
        src: /etc/nginx/sites-available/aptly
        dest: /etc/nginx/sites-enabled/aptly
        state: link

    - name: Перезапуск Nginx
      systemd:
        name: nginx
        state: restarted
        enabled: yes

    # === Локальный Container Registry ===
    - name: Запуск локального registry (Podman)
      containers.podman.podman_container:
        name: registry
        image: docker.io/library/registry:2
        state: started
        restart_policy: always
        ports:
          - "{{ registry_port }}:5000"
        volumes:
          - "{{ registry_base }}/data:/var/lib/registry:z"

    - name: Скачивание и загрузка образов Ceph
      shell: |
        IMAGES=(
          quay.io/ceph/ceph:v{{ ceph_version }}
          quay.io/ceph/keepalived:2.1.5
          quay.io/prometheus/prometheus:v2.51.0
          quay.io/prometheus/node-exporter:v1.7.0
          quay.io/prometheus/alertmanager:v0.27.0
          quay.io/ceph/grafana:10.4.0
          quay.io/ceph/haproxy:2.3
        )
        for img in "${IMAGES[@]}"; do
          echo "Pulling: $img"
          podman pull "$img"
          short_name="${img#quay.io/}"
          echo "Tagging: localhost:{{ registry_port }}/$short_name"
          podman tag "$img" "localhost:{{ registry_port }}/$short_name"
          echo "Pushing: localhost:{{ registry_port }}/$short_name"
          podman push "localhost:{{ registry_port }}/$short_name"
        done
      args:
        executable: /bin/bash
      register: image_push
      changed_when: "'Pushing' in image_push.stdout"

    - name: Создание архива registry
      archive:
        path: "{{ registry_base }}"
        dest: "/tmp/ceph-registry-{{ ceph_version }}.tar.gz"
        format: gz

    - name: Создание архива Aptly
      archive:
        path: /root/.aptly
        dest: "/tmp/ceph-aptly-{{ ceph_version }}.tar.gz"
        format: gz

    - name: Вывод инструкций по переносу
      debug:
        msg: |
          ====== AIR-GAP TRANSFER INSTRUCTIONS ======
          1. Скопируйте файлы на изолированную сеть:
             scp /tmp/ceph-registry-{{ ceph_version }}.tar.gz user@jump-host:/media/usb/
             scp /tmp/ceph-aptly-{{ ceph_version }}.tar.gz user@jump-host:/media/usb/

          2. На jump-хосте изолированной сети:
             scp /media/usb/*.tar.gz root@airgap-master-internal:/tmp/

          3. Запустите playbook 2: deploy-air-gapped-services.yml

          Размер архивов:
          - registry: ~3-5 ГБ
          - aptly: ~30-50 ГБ
```

#### Playbook 2: Развёртывание сервисов в замкнутом контуре

Выполняется на «мастер-узле» внутри изолированной сети (после переноса архивов).

```yaml
# ansible/deploy-air-gapped-services.yml
- name: Развёртывание сервисов в замкнутом контуре
  hosts: airgap-master-internal
  become: yes
  vars:
    ceph_version: "19.2.4"
    registry_port: 5000
    aptly_port: 8080
    master_ip: "192.168.0.1"  # IP этого узла в изолированной сети
    registry_base: "/opt/registry"
    network_cidr: "192.168.0.0/24"

  tasks:
    - name: Установка Podman и Nginx
      apt:
        name:
          - podman
          - nginx
          - chrony
        state: present

    - name: Распаковка registry
      unarchive:
        src: "/tmp/ceph-registry-{{ ceph_version }}.tar.gz"
        dest: /
        remote_src: yes
      notify: restart registry

    - name: Запуск registry
      containers.podman.podman_container:
        name: registry
        image: docker.io/library/registry:2
        state: started
        restart_policy: always
        ports:
          - "{{ registry_port }}:5000"
        volumes:
          - "{{ registry_base }}/data:/var/lib/registry:z"

    - name: Распаковка Aptly
      unarchive:
        src: "/tmp/ceph-aptly-{{ ceph_version }}.tar.gz"
        dest: /root
        remote_src: yes

    - name: Настройка Nginx для репозитория
      copy:
        dest: /etc/nginx/sites-available/aptly
        content: |
          server {
              listen {{ aptly_port }};
              server_name _;
              root /root/.aptly/public;
              location / {
                  autoindex on;
              }
          }

    - name: Активация Nginx
      file:
        src: /etc/nginx/sites-available/aptly
        dest: /etc/nginx/sites-enabled/aptly
        state: link
      notify: restart nginx

    - name: Удаление default site
      file:
        path: /etc/nginx/sites-enabled/default
        state: absent
      notify: restart nginx

    - name: Запуск Nginx
      systemd:
        name: nginx
        state: started
        enabled: yes

    # === Локальный NTP-сервер ===
    - name: Настройка chrony как локального NTP-сервера
      copy:
        dest: /etc/chrony/chrony.conf
        content: |
          # Локальный NTP-сервер (stratum 10)
          local stratum 10
          # Разрешить запросы из локальной сети
          allow {{ network_cidr }}
          # Запретить внешние соединения
          bindcmdaddress 127.0.0.1
          bindcmdaddress ::1

    - name: Перезапуск chrony
      systemd:
        name: chrony
        state: restarted
        enabled: yes

    - name: Фиксация текущего времени как эталонного
      command: chronyc -a makestep
      ignore_errors: yes

    - name: Вывод информации о сервисах
      debug:
        msg: |
          ====== AIR-GAP SERVICES READY ======
          Package repo:  http://{{ master_ip }}:{{ aptly_port }}
          Registry:      {{ master_ip }}:{{ registry_port }}
          NTP server:    {{ master_ip }}
          ====================================

  handlers:
    - name: restart nginx
      systemd:
        name: nginx
        state: restarted

    - name: restart registry
      systemd:
        name: podman
        state: restarted
```

#### Playbook 3: Подготовка узлов Ceph в замкнутом контуре

Выполняется на всех узлах кластера.

```yaml
# ansible/prepare-ceph-nodes.yml
- name: Подготовка узлов Ceph (air-gapped)
  hosts: ceph_nodes
  become: yes
  vars:
    master_ip: "192.168.0.1"
    aptly_port: 8080
    registry_port: 5000
    ubuntu_release: "noble"
    public_network: "10.0.1.0/24"
    cluster_network: "10.0.2.0/24"

  tasks:
    - name: Настройка hostname
      hostname:
        name: "{{ inventory_hostname }}"
      when: ansible_hostname != inventory_hostname

    - name: Настройка /etc/hosts
      lineinfile:
        path: /etc/hosts
        line: "{{ hostvars[item].ansible_host }} {{ item }}"
        state: present
      loop: "{{ groups['ceph_nodes'] }}"

    # === Локальный репозиторий ===
    - name: Очистка стандартных репозиториев
      file:
        path: "{{ item }}"
        state: absent
      loop:
        - /etc/apt/sources.list
        - /etc/apt/sources.list.d/ubuntu.sources

    - name: Добавление локального репозитория
      copy:
        dest: /etc/apt/sources.list.d/local.list
        content: |
          deb [trusted=yes] http://{{ master_ip }}:{{ aptly_port }}/ {{ ubuntu_release }} main

    - name: Обновление кеша пакетов
      apt:
        update_cache: yes
      retries: 3
      delay: 10

    - name: Установка необходимых пакетов
      apt:
        name:
          - podman
          - lvm2
          - chrony
          - python3
          - smartmontools
        state: present

    # === Локальный container registry ===
    - name: Настройка доступа к локальному registry
      copy:
        dest: /etc/containers/registries.conf.d/local-registry.conf
        content: |
          [[registry]]
          location = "{{ master_ip }}:{{ registry_port }}"
          insecure = true
          [[registry.mirror]]
          location = "{{ master_ip }}:{{ registry_port }}"
          insecure = true

    - name: Проверка доступности registry
      command: "curl -s http://{{ master_ip }}:{{ registry_port }}/v2/_catalog"
      register: registry_check
      retries: 5
      delay: 5
      until: registry_check.rc == 0

    # === Локальный NTP-клиент ===
    - name: Настройка chrony на мастер-узел
      copy:
        dest: /etc/chrony/chrony.conf
        content: |
          server {{ master_ip }} iburst
          driftfile /var/lib/chrony/drift
          makestep 1.0 3
          rtcsync

    - name: Перезапуск chrony
      systemd:
        name: chrony
        state: restarted
        enabled: yes

    - name: Принудительная синхронизация времени
      command: chronyc -a makestep
      register: chrony_result
      changed_when: "'System clock wrong' in chrony_result.stdout or 'is not running' not in chrony_result.stderr"
      ignore_errors: yes

    # === Проверки ===
    - name: Проверка доступности репозитория
      command: "curl -s --connect-timeout 5 http://{{ master_ip }}:{{ aptly_port }}"
      register: repo_test

    - name: Проверка отсутствия доступа в Интернет
      command: "curl --connect-timeout 5 http://archive.ubuntu.com"
      register: internet_test
      failed_when: internet_test.rc == 0
      ignore_errors: yes

    - name: Вывод результатов проверки
      debug:
        msg:
          - "Local repo: {{ 'OK' if repo_test.rc == 0 else 'FAILED' }}"
          - "Internet access: {{ 'OFFLINE (good)' if internet_test.rc != 0 else 'ONLINE (bad!)' }}"
          - "Registry: {{ 'OK' if registry_check.rc == 0 else 'FAILED' }}"

    # === Сеть ===
    - name: Проверка сетевых интерфейсов
      shell: |
        ip addr show | grep -E "{{ public_network.split('.')[0:2] | join('.') }}|{{ cluster_network.split('.')[0:2] | join('.') }}"
      register: network_check

    - name: Настройка MTU 9000 для cluster-интерфейсов
      sysctl:
        name: "net.ipv4.conf.{{ item }}.mtu"
        value: 9000
      loop:
        - eth1
        - eth2
      when: cluster_network != ''
      ignore_errors: yes
```

#### Playbook 4: Bootstrap Ceph в замкнутом контуре

```yaml
# ansible/bootstrap-air-gapped-ceph.yml
- name: Bootstrap Ceph в замкнутом контуре
  hosts: ceph_monitors[0]   # Первый MON-узел
  become: yes
  vars:
    master_ip: "192.168.0.1"
    registry_port: 5000
    mon_ip: "{{ ansible_default_ipv4.address }}"
    public_network: "10.0.1.0/24"
    cluster_network: "10.0.2.0/24"
    ceph_version: "19.2.4"

  tasks:
    - name: Проверка cephadm
      command: which cephadm
      register: cephadm_check
      ignore_errors: yes

    - name: Установка cephadm
      shell: |
        apt install -y cephadm
      when: cephadm_check.rc != 0

    - name: Проверка доступности образов
      command: "podman pull {{ master_ip }}:{{ registry_port }}/ceph/ceph:v{{ ceph_version }}"
      register: pull_result
      retries: 3
      delay: 10
      until: pull_result.rc == 0

    - name: Bootstrap кластера
      command: >
        cephadm
        --image {{ master_ip }}:{{ registry_port }}/ceph/ceph:v{{ ceph_version }}
        --no-container-init
        bootstrap
        --mon-ip {{ mon_ip }}
        --cluster-network {{ cluster_network }}
        --skip-mon-network
        --skip-dashboard
        --output-dir /etc/ceph
        --allow-fqdn-hostname
      args:
        creates: /etc/ceph/ceph.conf
      register: bootstrap_result

    - name: Вывод результатов bootstrap
      debug:
        var: bootstrap_result.stdout_lines

    - name: Бэкап ключей
      archive:
        path:
          - /etc/ceph
          - /var/lib/ceph/bootstrap-*
        dest: /root/ceph-keys-airgap.tar.gz
        format: gz

    - name: Сохранить admin keyring локально
      fetch:
        src: /etc/ceph/ceph.client.admin.keyring
        dest: ./fetched-keyrings/
        flat: yes
```

#### Playbook 5: Расширение кластера в замкнутом контуре

```yaml
# ansible/expand-air-gapped-ceph.yml
- name: Расширение Ceph-кластера (air-gapped)
  hosts: ceph_monitors[0]   # Выполняется с admin-узла
  become: yes
  vars:
    registry_server: "192.168.0.1:5000"

  tasks:
    - name: Настройка глобального образа контейнера
      command: >
        ceph config set mgr
        mgr/cephadm/container_image_ceph
        {{ registry_server }}/ceph/ceph:v19.2.4

    - name: Добавление MON-узлов
      command: "ceph orch host add {{ item }} {{ hostvars[item].ansible_host }} --labels _admin,mon,mgr"
      loop: "{{ groups['ceph_monitors'][1:] }}"
      when: groups['ceph_monitors'] | length > 1

    - name: Добавление OSD-узлов
      command: "ceph orch host add {{ item }} {{ hostvars[item].ansible_host }} --labels osd"
      loop: "{{ groups['ceph_osds'] }}"
      when: groups['ceph_osds'] is defined

    - name: Развёртывание MON
      command: "ceph orch apply mon --placement='{{ groups['ceph_monitors'] | join(',') }}'"
      when: groups['ceph_monitors'] | length > 1

    - name: Развёртывание MGR
      command: "ceph orch apply mgr --placement='{{ groups['ceph_monitors'] | join(',') }}'"

    - name: Развёртывание OSD (все доступные диски)
      command: "ceph orch apply osd --all-available-devices"
      when: groups['ceph_osds'] is defined

    - name: Ожидание HEALTH_OK
      shell: |
        for i in $(seq 1 120); do
          STATUS=$(ceph health -f json | python3 -c "import sys,json; print(json.load(sys.stdin)['status'])")
          if [ "$STATUS" = "HEALTH_OK" ]; then
            echo "CLUSTER IS HEALTH_OK"
            exit 0
          fi
          sleep 30
        done
        echo "TIMEOUT: cluster not HEALTH_OK after 60 minutes"
        exit 1
      args:
        executable: /bin/bash
      register: health_wait
      changed_when: false

    - name: Итоговый статус
      command: ceph status
      register: final_status

    - name: Вывод статуса
      debug:
        var: final_status.stdout_lines

    - name: Проверка отсутствия доступа в Интернет (окончательная)
      command: "curl --connect-timeout 5 http://archive.ubuntu.com"
      register: final_internet_test
      failed_when: final_internet_test.rc == 0
      ignore_errors: yes

    - name: Подтверждение изоляции
      debug:
        msg: |
          ====== AIR-GAP CEPH CLUSTER VERIFIED ======
          ✓ Ceph cluster HEALTH_OK
          ✓ All services running from local registry
          ✓ Packages installed from local repository
          ✓ NTP synced to local server
          ✓ Internet access BLOCKED (verified)
          ==============================================
```

---

### 9.6. Проверка изоляции и отладка *(3 стр.)*

После развёртывания нужно убедиться, что:
1. Кластер работает корректно в изоляции
2. Ни один узел не «пробивается» в Интернет
3. Локальные сервисы (registry, repo, NTP) доступны

```bash
# === Тест 1: Проверка изоляции ===
# На каждом узле:
curl --connect-timeout 5 http://8.8.8.8 && echo "INTERNET: доступен (ПЛОХО!)" || echo "INTERNET: заблокирован ✓"
curl --connect-timeout 5 http://archive.ubuntu.com && echo "UBUNTU REPO: доступен (ПЛОХО!)" || echo "UBUNTU REPO: заблокирован ✓"
curl --connect-timeout 5 https://quay.io && echo "QUAY.IO: доступен (ПЛОХО!)" || echo "QUAY.IO: заблокирован ✓"

# === Тест 2: Локальные сервисы ===
# Репозиторий:
curl -s http://192.168.0.1:8080/ | head -20

# Registry:
curl -s http://192.168.0.1:5000/v2/_catalog
# Должен вернуть: {"repositories":["ceph/ceph","ceph/grafana",...]}

# NTP:
chronyc sources
# Должен показывать мастер-узел как источник

# === Тест 3: Ceph-кластер ===
ceph status
ceph osd tree
ceph df

# === Тест 4: Функциональное тестирование ===
# Создать тестовый RBD-образ
rbd create test-airgap --size 1G
rbd map test-airgap
mkfs.ext4 /dev/rbd0
mount /dev/rbd0 /mnt/test
echo "Air-gapped Ceph работает!" > /mnt/test/verify.txt
cat /mnt/test/verify.txt
umount /mnt/test
rbd unmap test-airgap
rbd rm test-airgap

# === Тест 5: Перезагрузка узла ===
# После перезагрузки все сервисы должны автоматически запуститься:
systemctl list-units 'ceph-*' --state=running
ceph orch ps | grep -c running
```

---

### 9.7. Обновление в замкнутом контуре *(2 стр.)*

Обновление в air-gapped — это многоэтапный процесс, так как новый образ нужно сначала доставить в изолированную сеть.

```bash
# === На машине с Интернетом ===

# 1. Скачать новый образ Ceph
CEPH_NEW="19.2.5"
podman pull quay.io/ceph/ceph:v${CEPH_NEW}
podman tag quay.io/ceph/ceph:v${CEPH_NEW} localhost:5000/ceph/ceph:v${CEPH_NEW}
podman push localhost:5000/ceph/ceph:v${CEPH_NEW}

# 2. Скачать обновлённые пакеты
aptly mirror update ceph-squid
aptly snapshot create ceph-squid-${CEPH_NEW} from mirror ceph-squid
aptly publish switch noble ceph-squid ceph-squid-${CEPH_NEW}

# 3. Упаковать
tar czf ceph-update-${CEPH_NEW}-registry.tar.gz -C /opt registry/data
tar czf ceph-update-${CEPH_NEW}-aptly.tar.gz -C /root .aptly

# 4. Перенести на изолированную сеть (USB/временная сеть)

# === На мастер-узле изолированной сети ===

# 5. Распаковать обновления
cd /
tar xzf /tmp/ceph-update-${CEPH_NEW}-registry.tar.gz
# Обновить репозиторий
tar xzf /tmp/ceph-update-${CEPH_NEW}-aptly.tar.gz -C /root

# 6. Перезапустить Nginx
systemctl restart nginx

# 7. Обновить пакеты на узлах
ansible ceph_nodes -m apt -a "update_cache=yes"
ansible ceph_nodes -m apt -a "name=cephadm state=latest"

# 8. Обновить образ по умолчанию
ceph config set mgr mgr/cephadm/container_image_ceph \
    ${MASTER_IP}:5000/ceph/ceph:v${CEPH_NEW}

# 9. Запустить rolling upgrade
ceph orch upgrade start --image ${MASTER_IP}:5000/ceph/ceph:v${CEPH_NEW}

# 10. Мониторинг
watch -n 10 'ceph orch upgrade status; ceph -s'
```

---

### 9.8. Практикум: полное развёртывание без интернета *(4 стр.)*

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

## Контрольные вопросы к Части III *(итоговые, по всем главам)*

### Глава 6 (Планирование кластера)

1. **Почему для OSD рекомендуется ~1 ядро CPU на процесс? Что произойдёт, если ядер меньше, чем OSD?**
2. **Рассчитайте полную конфигурацию (CPU, RAM, диски, сеть) для кластера из 5 узлов с 24 HDD на каждом. Опишите каждый компонент.**
3. **Сравните стоимость 1 ТБ полезной ёмкости на Dell R750xs (12×20TB HDD, ×3 rep) и Supermicro 6029P (12×16TB HDD, ×3 rep). Какая платформа дешевле и почему?**
4. **Зачем нужен Jumbo Frames в Ceph-сети? Опишите механизм: почему MTU 9000 снижает загрузку CPU.**
5. **Спроектируйте rack-aware CRUSH map для 4 стоек по 2 сервера. Что гарантирует эта топология?**
6. **Какие требования к latency для синхронной репликации между двумя дата-центрами? Что делать, если latency > 2 мс?**
7. **Опишите стратегию WAL/DB placement для сервера с 24 HDD и 3 NVMe. Рассчитайте размеры WAL и DB.**
8. **Спроектируйте VLAN-схему для production-кластера: public, cluster, management. Обоснуйте каждый VLAN.**
9. **Рассчитайте необходимую пропускную способность сети для кластера из 10 узлов с 12 NVMe каждый.**
10. **EC 4+2 vs ×3 репликация: сравните overhead по месту, latency, отказоустойчивость. В каком сценарии вы выберете EC, а в каком — репликацию?**

### Глава 7 (Ручное развёртывание)

11. **Почему синхронизация времени критична для Ceph? Что произойдёт при clock skew > 0.05s?**
12. **Перечислите все фазы `cephadm bootstrap` и опишите, что происходит в каждой из них (Preflight, генерация ключей, monmap, MON, конфигурация, MGR, systemd).**
13. **Какие bootstrap-ключи создаются при bootstrap и зачем нужен каждый? Что будет при потере `/var/lib/ceph/bootstrap-osd/ceph.keyring`?**
14. **Опишите разницу между `--mon-ip` и `--mon-addrv`. В каком случае использовать `--mon-addrv`?**
15. **Как выполнить bootstrap в замкнутом контуре? Перечислите все необходимые флаги.**
16. **Какие проверки выполняет cephadm на фазе Preflight? Что делать, если проверка не проходит?**
17. **Опишите, как cephadm хранит и использует SSH-ключи при добавлении узлов. Какой файл содержит публичный ключ?**
18. **Что такое monmap и какую роль он играет в кластере? Как восстановить monmap при потере всех MON?**
19. **Опишите процесс создания OSD: от команды `ceph orch daemon add osd` до запущенного контейнера.**
20. **Как проверить, что bootstrap прошёл успешно? Напишите чек-лист из 6 пунктов.**

### Глава 8 (cephadm и оркестрация)

21. **Что такое декларативная модель в cephadm? Приведите пример: вы описали 3 MON в spec, оркестратор видит 1 — что он сделает?**
22. **Напишите полный service spec для RadosGW с SSL, размещением на 2 узлах и отдельной сетью.**
23. **Как работает reconciliation loop в cephadm? Каков интервал между итерациями?**
24. **Что произойдёт, если OSD-контейнер упадёт (OOM kill)? Опишите автоматическое восстановление.**
25. **Опишите фазы rolling upgrade через `ceph orch upgrade start`. Почему MON обновляются по одному?**
26. **Как откатить незавершённое обновление? Какие риски при откате?**
27. **Напишите service spec для OSD с фильтрацией: только SSD от 1.92 ТБ, DB на NVMe Intel P4610.**
28. **Как проверить, что оркестратор «жив» и работает? Какие команды использовать?**
29. **Узел показывает статус Offline в `ceph orch host ls`. Опишите алгоритм диагностики и решения.**
30. **Как мигрировать кластер с ceph-deploy на cephadm? Опишите последовательность с ключевыми командами.**

### Глава 9 (Замкнутый контур)

31. **Перечислите 4 сервиса, которые нужно развернуть локально для air-gapped Ceph. Опишите назначение каждого.**
32. **Какие образы контейнеров нужны для полноценного air-gapped кластера? Перечислите минимум 5.**
33. **Как настроить локальный NTP-сервер на базе chrony для изолированной сети? Приведите конфигурацию.**
34. **Опишите процесс переноса образов контейнеров в изолированную сеть: от скачивания до использования в cephadm bootstrap.**
35. **Как проверить, что узел действительно изолирован от Интернета? Напишите 3 теста.**
36. **Что произойдёт, если в изолированной сети не настроен DNS? Как обойти эту проблему?**
37. **Опишите процедуру обновления Ceph в замкнутом контуре (новая минорная версия). Какие файлы нужно перенести?**
38. **Какие риски связаны с использованием insecure registry (`insecure = true`)? Как их минимизировать?**
39. **Напишите Ansible playbook для проверки изоляции всех узлов перед bootstrap.**
40. **Спроектируйте полный процесс развёртывания air-gapped кластера из 5 узлов: от мастер-узла с интернетом до HEALTH_OK. Опишите каждый этап.**

### Практические задания повышенной сложности

41. **Спроектируйте бюджет для кластера:** У вас бюджет $50,000. Нужно 100 ТБ полезных данных с ×3 репликацией. Выберите оборудование (Dell, Supermicro, или сборка), рассчитайте стоимость, обоснуйте выбор.

42. **Напишите Ansible playbook** для полного развёртывания Ceph на 3 узлах (lab), включая: установку пакетов, настройку сети, bootstrap, добавление MON/MGR/OSD, настройку dashboard.

43. **Спроектируйте план аварийного восстановления:** все 3 MON-узла потеряны безвозвратно (пожар в стойке). OSD-узлы целы. Опишите пошаговую процедуру восстановления кластера.

44. **Миграция между дата-центрами:** нужно перенести данные из DC1 (старый кластер, ceph-deploy, Octopus) в DC2 (новый, cephadm, Squid). Сеть между DC: 1 Gbps, latency 10 мс. Предложите стратегию.

45. **Отказоустойчивость критичного пула:** спроектируйте CRUSH map для пула с данными, которые должны пережить одновременный отказ 2 стоек (из 5). Опишите конфигурацию.

---

| Навигация | |
|-----------|---|
| ← Часть II | [part-II.md](part-II.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Часть IV | [part-IV.md](part-IV.md) |
