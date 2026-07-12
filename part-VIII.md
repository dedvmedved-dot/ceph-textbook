# Часть VIII. Kubernetes и CSI *(85 стр., 6 кейсов)*

> **Цель:** освоить интеграцию Ceph с Kubernetes через CSI — от архитектуры Rook до диагностики отказов и тюнинга производительности.
> **После этой части вы сможете:** развернуть Rook + Ceph в K8s, настроить RBD- и CephFS-тома для подов, измерить и улучшить производительность CSI, диагностировать 6 типовых кейсов отказа.

---

## Глава 26. Ceph + Kubernetes: архитектура *(16 стр.)*

### 26.1. CSI (Container Storage Interface) *(3 стр.)*
- Зачем CSI: стандартный интерфейс между K8s и системами хранения
- Компоненты CSI: controller plugin, node plugin, sidecar-контейнеры
- DOT-схема: как K8s говорит с хранилищем через CSI

### 26.2. Rook *(4 стр.)*
- Rook: оператор Ceph для Kubernetes
- CRD: CephCluster, CephBlockPool, CephFilesystem, CephObjectStore
- Reconciler: как оператор приводит реальность к желаемому состоянию
- DOT-схема: компоненты Rook (operator, mon, osd, mgr, mds, rgw)

### 26.3. Ceph-CSI *(3 стр.)*
- RBD plugin: provisioner, attacher, resizer, snapper
- CephFS plugin: provisioner, controller, node
- NFS plugin: provisioner для NFS-томов поверх CephFS (Squid+, экспериментально)

### 26.4. StorageClass, PVC, PV *(4 стр.)*
- DOT-схема: жизненный цикл — PVC → StorageClass → CSI provisioner → PV → Pod
- StorageClass: provisioner, parameters (pool, fsName, imageFeatures)
- PVC: accessModes (RWO, ROX, RWX), resources.requests.storage
- Binding: PVC → PV, reclaimPolicy (Retain/Delete)

### 26.5. Практикум: Rook + Ceph *(2 стр.)*
- `kubectl apply -f rook-operator.yaml`
- `kubectl apply -f cluster.yaml`
- Через 10 минут: работающий Ceph в K8s

---

## Глава 27. CSI RBD: блочные тома *(18 стр.)*

### 27.1. RBD CSI: компоненты *(4 стр.)*
- DOT-схема: provisioner → attacher → resizer → snapper
- Provisioner: создаёт RBD-образ
- Attacher: мапит RBD на узел (`rbd map`)
- Resizer: расширяет образ
- Snapper: создаёт/восстанавливает снапшоты

### 27.2. StorageClass *(3 стр.)*
- Параметры: `pool`, `imageFeatures`, `imageFormat`, `csi.storage.k8s.io/fstype`
- `imageFeatures: layering` — минимальный набор
- Режимы томов: `Block`, `Filesystem`
- Пример StorageClass для SSD-пула с шифрованием

### 27.3. Снапшоты и клоны *(3 стр.)*
- VolumeSnapshotClass: привязка к CSI-драйверу
- VolumeSnapshot: создание снапшота PVC
- PVC clone: `dataSource` → существующий PVC
- Восстановление: создание PVC из снапшота

### 27.4. Расширение тома *(2 стр.)*
- `allowVolumeExpansion: true` в StorageClass
- `kubectl edit pvc <name>` → увеличить `storage`
- Онлайн-расширение: под не перезапускается

### 27.5. Практикум *(6 стр.)*
- PVC (10 Gi) → запись данных → VolumeSnapshot
- Клон PVC из снапшота → новый Pod
- Расширение тома до 20 Gi
- Проверка: данные сохранены

---

## Глава 28. CSI CephFS: RWX-тома *(14 стр.)*

### 28.1. CephFS CSI *(3 стр.)*
- Provisioner: создаёт subvolume в CephFS
- `ReadWriteMany` vs `ReadWriteOnce`: когда RWX
- kernel driver vs ceph-fuse: что выбрать

### 28.2. StorageClass *(3 стр.)*
- Параметры: `fsName`, `pool`, `csi.storage.k8s.io/provisioner-secret`
- `mounter: kernel` (быстрее) vs `mounter: fuse` (совместимее)
- `subvolumegroup`, `path`

### 28.3. Shared volume *(3 стр.)*
- DOT-схема: несколько подов → один CephFS-том (RWX)
- Пример: Nginx (несколько реплик) с общей статикой
- Проблемы: блокировки, консистентность, производительность

### 28.4. Практикум *(5 стр.)*
- Deployment Nginx × 3 реплики + общий RWX-том
- Запись файла из одного пода → чтение из другого
- Замер производительности: `fio` на CephFS RWX

---

## Глава 29. Производительность CSI *(20 стр.)*

### 29.1. CSI performance model *(4 стр.)*
- DOT-схема: путь I/O K8s → Ceph
  Pod → Filesystem (ext4/xfs) → RBD device → KRBD → network → OSD
- Где добавляется latency: каждый слой
- Накладные расходы CSI vs bare-metal RBD

### 29.2. Бенчмаркинг *(4 стр.)*
- `fio` в поде: `direct=1`, `ioengine=libaio`, `bs=4k`
- Сравнение: RBD CSI vs CephFS CSI vs bare-metal RBD
- Интерпретация: IOPS, MB/s, latency p99

### 29.3. Тюнинг CSI *(4 стр.)*
- `rbd_op_threads`: управление от CSI-драйвера
- `rbd_cache`: включение/выключение, `rbd_cache_max_dirty`
- `readahead`: `rbd_readahead_max_bytes`
- `blk-mq`: multi-queue на уровне ядра

### 29.4. Rook vs external Ceph *(3 стр.)*
- Rook-managed: Ceph внутри K8s — overhead оркестрации
- External Ceph: кластер вне K8s — меньше overhead
- Сравнение latency: Rook-managed vs external

### 29.5. Практикум *(5 стр.)*
- Снять производительность CSI RBD и CephFS: `fio` в поде
- Сравнить с bare-metal RBD на том же кластере
- Grafana-дашборд для CSI latency

---

## Глава 30. Диагностика и отказы в CSI: 6 кейсов *(17 стр.)*

### 30.1. Кейс 1: PVC в Pending *(3 стр.)*
- Симптом: `kubectl get pvc` → STATUS=Pending
- Диагностика: `kubectl describe pvc`, events, `csi-rbdplugin` логи
- Причины: не найден StorageClass, pool не существует, Ceph недоступен

### 30.2. Кейс 2: том не монтируется *(3 стр.)*
- Симптом: Pod в ContainerCreating, events: `failed to mount`
- `volumeMount`, `DEVICEFEATURE` mismatch
- Диагностика: `kubectl describe pod`, `/var/log/syslog` на узле

### 30.3. Кейс 3: потеря связи с Ceph *(3 стр.)*
- Симптом: Pod в ContainerCreating, CSI timeout
- Причина: сеть K8s → Ceph недоступна
- Диагностика: `ceph status` из пода CSI-provisioner

### 30.4. Кейс 4: отказ OSD под RBD-томом *(3 стр.)*
- Симптом: латенция в поде, I/O errors
- Диагностика: `ceph osd tree`, PG degraded
- Влияние на поды: зависание I/O до восстановления

### 30.5. Кейс 5: восстановление из снапшота *(2 стр.)*
- Симптом: случайное удаление данных в поде
- Решение: VolumeSnapshot → новый PVC → подмонтировать в новый Pod
- Проверка: данные восстановлены

### 30.6. Кейс 6: DR в K8s *(3 стр.)*
- Сценарий: потерян кластер K8s, остались снапшоты PVC
- Восстановление: новый K8s → VolumeSnapshot → PVC → Pod
- Полный цикл: проверка, что данные и приложение работают

---

| Навигация | |
|-----------|---|
| ← Часть VII | [part-VII.md](part-VII.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Часть IX | [part-IX.md](part-IX.md) |
