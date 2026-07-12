# Часть IX. Безопасность *(30 стр.)*

> **Цель:** освоить модель безопасности Ceph — аутентификацию (CephX), авторизацию (caps), шифрование данных (at-rest, in-transit).
> **После этой части вы сможете:** создать пользователя с минимальными правами, ограничить доступ по пулам и подсетям, включить шифрование на дисках и в сети.

---

## Глава 31. CephX: аутентификация и авторизация *(18 стр.)*

### 31.1. CephX: цепочка доверия *(3 стр.)*
- Модель: клиент → MON (аутентификация) → OSD (авторизация)
- Сессионные ключи: временные, автоматическая ротация
- Защита: man-in-the-middle, replay attacks
- DOT-схема: handshake CephX

### 31.2. keyring, caps, profiles *(4 стр.)*
- keyring: формат, где хранится (`/etc/ceph/ceph.client.admin.keyring`)
- Caps (capabilities): синтаксис `allow rwx`, разбор
  - `mon`: `allow r` (читать карты), `allow *` (всё)
  - `osd`: `allow rwx pool=cephfs_data`
  - `mds`: `allow rw`
- Profiles: предустановленные наборы прав (bootstrap-osd, bootstrap-mds)
- Построчный разбор: `ceph auth get client.admin`

### 31.3. Пользователи *(3 стр.)*
- Создание: `ceph auth add client.myuser mon 'allow r' osd 'allow rwx pool=data'`
- Просмотр: `ceph auth ls`
- Отзыв: `ceph auth del client.myuser`
- Аудит: `ceph auth list` — кто и с какими правами

### 31.4. Ограничение доступа *(3 стр.)*
- Пулы: `osd 'allow rwx pool=app_data'`
- Namespaces: `osd 'allow rwx pool=data namespace=tenant_a'`
- Подсети: `ceph auth caps client.myuser mon 'allow r' osd 'allow rwx pool=data' network 10.0.1.0/24`
- Экспорт/импорт: `ceph auth export > auth.bak`

### 31.5. Практикум *(5 стр.)*
- Создать 3 роли: admin (полный доступ), developer (rw в dev-пул), reader (r в prod-пул)
- Проверить: `rados -n client.reader -k reader.keyring ls -p prod`
- Убедиться: reader не может писать, developer не видит prod

---

## Глава 32. Шифрование *(12 стр.)*

### 32.1. At-rest и in-transit *(4 стр.)*
- At-rest: dm-crypt/LUKS на дисках OSD
  - Как включить: `encrypted=True` в OSD spec
  - Влияние на производительность: ~5-10% overhead
  - Управление ключами: `ceph config-key`
- In-transit: MSGR2 + TLS между компонентами
  - Как включить: `ms_mode=secure`
  - mTLS: взаимная аутентификация сертификатами
  - DOT-схема: какие соединения шифруются (MON↔OSD, OSD↔OSD, клиент↔OSD)

### 32.2. SSE-S3 на RGW *(3 стр.)*
- Режимы: SSE-S3 (ключи Ceph), SSE-KMS (внешний KMIP), SSE-C (ключи клиента)
- Настройка: `rgw_crypt_s3_kms_backend`, `rgw_crypt_require_ssl`
- Ограничения: не все S3-клиенты поддерживают SSE

### 32.3. Практикум *(5 стр.)*
- Включить LUKS на тестовом OSD: полный цикл (создание, проверка `lsblk`)
- Включить MSGR2 secure mode: `ceph config set mon ms_mode secure`
- Проверка: `tcpdump` показывает зашифрованный трафик
- Замер производительности: до и после шифрования (fio)

---

| Навигация | |
|-----------|---|
| ← Часть VIII | [part-VIII.md](part-VIII.md) |
| ↑ Оглавление | [TOC.md](TOC.md) |
| → Приложения | [appendix.md](appendix.md) |
