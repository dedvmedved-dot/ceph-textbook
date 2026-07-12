# Приложения *(55 стр.)*

---

## Приложение A. Словарь терминов *(10 стр.)*

120+ терминов с переводом, расшифровкой аббревиатур и русским пояснением.

| Термин | Расшифровка | Пояснение |
|--------|-------------|-----------|
| **BlueStore** | — | Движок хранения OSD, заменивший FileStore с версии Luminous. Хранит объекты напрямую на блочном устройстве, RocksDB для метаданных |
| **CephFS** | Ceph File System | Распределённая POSIX-совместимая файловая система поверх RADOS |
| **CephX** | Ceph eXtended Authentication | Протокол аутентификации и авторизации в Ceph |
| **CRUSH** | Controlled Replication Under Scalable Hashing | Алгоритм псевдослучайного размещения данных без центральной таблицы |
| **MDS** | Metadata Server | Сервер метаданных, обслуживает CephFS |
| **MGR** | Manager | Менеджер — сбор метрик, Dashboard, модули |
| **MON** | Monitor | Монитор — хранит карту кластера, обеспечивает кворум |
| **OSD** | Object Storage Daemon | Демон хранения объектов — хранит данные на дисках |
| **PG** | Placement Group | Группа размещения — логическая группа объектов |
| **RADOS** | Reliable Autonomic Distributed Object Store | Фундамент Ceph — распределённое объектное хранилище |
| **RBD** | RADOS Block Device | Блочное устройство поверх RADOS |
| **RGW** | RADOS Gateway | Шлюз объектного хранилища с S3/Swift API |
| **SDS** | Software-Defined Storage | Программно-определяемое хранилище — логика отделена от железа |
| ... | ... | *(полный список из 120+ терминов — раскрывается при написании главы)* |

---

## Приложение B. Командная шпаргалка *(8 стр.)*

### `ceph` — управление кластером
```bash
ceph status                      # состояние кластера
ceph health detail               # детали здоровья
ceph osd tree                    # дерево OSD
ceph osd df                      # использование OSD
ceph osd pool ls                 # список пулов
ceph osd pool create <name> <pg> # создать пул
ceph osd pool set <pool> size 3  # репликация
ceph mon stat                    # статус MON
ceph mon dump                    # monmap
ceph pg dump                     # все PG
ceph pg <pgid> query             # детали PG
ceph auth ls                     # пользователи
ceph auth add client.<name> ...  # создать пользователя
ceph log last 100                # последние 100 записей
```

### `cephadm` — оркестрация
```bash
cephadm bootstrap --mon-ip <ip>  # развернуть bootstrap
ceph orch host add <host> <ip>   # добавить узел
ceph orch host ls                # список узлов
ceph orch apply mon/osd/mds/rgw  # развернуть сервис
ceph orch ps                     # статус контейнеров
ceph orch ls                     # список сервисов
ceph orch upgrade start          # обновление
cephadm shell                    # контейнер с конфигами
```

### `rados` — работа с объектами
```bash
rados lspools                    # список пулов
rados ls -p <pool>               # объекты в пуле
rados get <obj> -p <pool>        # скачать объект
rados put <obj> -p <pool>        # загрузить объект
rados bench -p <pool> 60 write   # бенчмарк
```

### `rbd` — блочные устройства
```bash
rbd create -s 10G <name>         # создать образ
rbd ls -p <pool>                 # список образов
rbd map <pool>/<name>            # подключить
rbd snap create <img>@<snap>     # снапшот
rbd snap rollback <img>@<snap>   # откат
rbd clone <src>@<snap> <dst>     # клон
```

### `ceph-volume` — управление дисками
```bash
ceph-volume inventory            # список дисков
ceph-volume lvm create --data /dev/sdb  # создать OSD
```

---

## Приложение C. Чек-листы *(6 стр.)*

### Чек-лист развёртывания
- [ ] chrony синхронизирован (offset < 0.05s)
- [ ] hostname FQDN, /etc/hosts
- [ ] Docker/Podman установлен и запущен
- [ ] Порты открыты (3300, 6789, 6800-7300)
- [ ] `cephadm bootstrap` завершён без ошибок
- [ ] MON: 3 узла, кворум 3/3
- [ ] MGR: active + standby
- [ ] OSD: все up + in
- [ ] PG: все active+clean
- [ ] Тестовый пул создан, `rados put/get` работает

### Чек-лист обновления
- [ ] `ceph health` = HEALTH_OK
- [ ] PG: все active+clean
- [ ] Бэкап monmap, osdmap, crushmap, auth export
- [ ] `ceph orch upgrade check` — совместимость
- [ ] `ceph orch upgrade start`
- [ ] `ceph orch upgrade status` — мониторинг
- [ ] HEALTH_OK после обновления
- [ ] Клиентские подключения работают

### Чек-лист аварии
(См. §21.1 — чек-лист журнала аварии на 20 пунктов)

### Чек-лист DR
- [ ] Бэкап monmap, osdmap, crushmap, auth
- [ ] RBD Mirror active (для критичных образов)
- [ ] CephFS снапшоты по расписанию
- [ ] Проверка восстановления: раз в квартал

### Чек-лист post-mortem
(См. §21.4 — шаблон post-mortem)

---

## Приложение D. Атлас DOT-схем *(12 стр.)*

Все ~47 схем с миниатюрами и ссылками на исходный DOT-код.

| № | Схема | Глава |
|----|-------|-------|
| 1 | Эволюция хранилищ (RAID → DAS → NAS → SAN → SDS) | 1.2 |
| 2 | Горизонтальное vs вертикальное масштабирование | 1.3 |
| 3 | Три интерфейса Ceph (RBD, CephFS, RGW) | 2.3 |
| 4 | RADOS — компоненты и связи | 3.6 |
| 5 | Кворум MON (Paxos) | 3.2 |
| 6 | Слои BlueStore | 3.3 |
| 7 | CRUSH map — иерархия (DC → rack → host → osd) | 4.3 |
| 8 | CRUSH — вычисление OSD (take → choose → emit) | 4.4 |
| 9 | Путь данных: клиент → OSD | 5.2 |
| 10 | Запись: временна́я диаграмма (WAL → replication → ack) | 5.3 |
| 11 | Scrubbing: light vs deep | 5.5 |
| 12 | Топологии: 3 узла, 5 узлов, 10+ узлов | 6.2 |
| 13 | Сетевая сегментация Ceph | 6.3 |
| 14 | cephadm bootstrap — что создаётся | 7.3 |
| 15 | cephadm declarative state | 8.3 |
| 16 | Air-gapped bootstrap | 9.4 |
| 17 | Дерево решений диагностики | 11.1 |
| 18 | Admin socket — схема взаимодействия | 13.1 |
| 19 | Инструменты бенчмаркинга — схема тестов | 14.2 |
| 20 | BlueStore: путь байта | 15.1 |
| 21 | Сетевой QoS: полосы трафика (client/recovery/scrub) | 16.4 |
| 22 | Домены отказа: вложенные уровни | 18.1 |
| 23 | Цепочка отказа OSD: хронометраж | 19.1 |
| 24 | Split-brain: две половинки кластера | 19.5 |
| 25 | MON синхронизация monmap | 19.9 |
| 26 | План восстановления (мини-DR) | 19.13 |
| 27 | RBD Mirror топология | 20.2 |
| 28 | RGW Multi-site топология | 20.4 |
| 29 | Post-mortem блок-схема | 21.3 |
| 30 | RBD слои (striping → objects) | 23.1 |
| 31 | RBD Mirror failover | 23.3 |
| 32 | CephFS: файл → inode → объекты | 24.1 |
| 33 | NFS-Ganesha экспорт CephFS | 24.3 |
| 34 | RGW Multi-site: зоны, группы, realm | 25.2 |
| 35 | CSI: как K8s говорит с хранилищем | 26.1 |
| 36 | Rook: компоненты оператора | 26.2 |
| 37 | PVC → PV → Pod: жизненный цикл | 26.4 |
| 38 | RBD CSI: provisioner/attacher/resizer/snapper | 27.1 |
| 39 | CephFS RWX: несколько подов → один том | 28.3 |
| 40 | CSI latency: путь I/O K8s → Ceph | 29.1 |
| 41 | CephX handshake | 31.1 |
| 42 | Шифрование at-rest (LUKS) + in-transit (TLS) | 32.1 |

*Каждая схема: ортогональные стрелки, цветная, белый фон (Material Design pastel), чёрные шрифты. Исходный DOT-код в `/schemas/`.*

---

## Приложение E. Ответы к практикумам *(12 стр.)*

Ответы и пояснения ко всем практическим заданиям из глав 1–32.

---

## Приложение F. Лабораторный стенд *(7 стр.)*

### Конфигурация ВМ (Proxmox)
- 8 виртуальных машин
- Ceph Squid 19.2.x
- 3 MON + 5 OSD (110 GiB) + CephFS + RBD

### Vagrantfile
```ruby
# Полный Vagrantfile для развёртывания стенда
```

### Инвентарь Ansible
```yaml
# inventory.yml — описание узлов и групп
```

### Ссылки
- Репозиторий стенда: [ceph-ha-lab](https://github.com/dedvmedved-dot/ceph-ha-lab)
- Репозитории кейсов: `ceph-case-01` — `ceph-case-36`

---

| Навигация | |
|-----------|---|
| ← Часть IX | [part-IX.md](part-IX.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
