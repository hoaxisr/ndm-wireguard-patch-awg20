# AmneziaWG 2.0 для Keenetic — drop-in замена wireguard.ko

Патчи и готовые модули для замены встроенного `wireguard.ko` на Keenetic роутерах модулем AmneziaWG v2.0 с поддержкой обфускации трафика (DPI bypass).

## Что это

AmneziaWG v2.0 — форк WireGuard с обфускацией: динамические заголовки (H1-H4 диапазоны), junk-пакеты (Jc/Jmin/Jmax), размеры мусора (S1-S4), пакеты маскировки (I1-I5).

Этот проект адаптирует upstream AmneziaWG v2.0 для работы как drop-in замена нативного `wireguard.ko` в Keenetic NDMS:
- NDMS видит модуль как свой — статус соединения, handshake, трафик отображаются корректно
- Управление через web-интерфейс Keenetic работает штатно
- AWG-параметры настраиваются через утилиту `awg`

## Поддерживаемые модели (75 моделей, 8 платформ)

| Модуль | Платформа | Архитектура | Утилита | Модели |
|---|---|---|---|---|
| `wireguard-mt7621.ko` | MT7621 | mipsel | `awg-mipsel` | KN-1010/1011/1810/1910/1913/2310/2311/2610/2810/2910/2911/3010/3013/3410/3510 |
| `wireguard-mt7628.ko` | MT7628 | mipsel | `awg-mipsel` | KN-1110/1111/1112/1121/1210/1211/1212/1213/1221/1310/1311/1410/1510/1511/1610/1611/1613/1621/1710/1711/1713/1714/1721/2210/2211/2212/3210/3211/3310/3311/4910 |
| `wireguard-en7528.ko` | EN7528 | mipsel | `awg-mipsel` | KN-1912/3012/3710/3810 |
| `wireguard-en7512.ko` | EN7512 | mips BE | `awg-mips` | KN-2010/2011/2012/2110/2111 |
| `wireguard-en7516.ko` | EN7516 | mips BE | `awg-mips` | KN-2112/2410/2510/3610 |
| `wireguard-mt7622.ko` | MT7622 | aarch64 | `awg-aarch64` | KN-1811/2710 |
| `wireguard-mt7981.ko` | MT7981 | aarch64 | `awg-aarch64` | KN-1012/2312/3411/3711/3712/3811/3812/3910/3911/4010/4410 |
| `wireguard-mt7988.ko` | MT7988 | aarch64 | `awg-aarch64` | KN-1812 |

## Быстрая установка

### 1. Определите платформу роутера

Зайдите в CLI роутера (SSH) и выполните:
```
cat /proc/cpuinfo | grep -i "system type\|Hardware"
```

### 2. Скопируйте файлы на роутер

```bash
# Модуль ядра (выберите свой)
scp releases/modules/wireguard-mt7621.ko root@router:/opt/tmp/wireguard.ko

# Утилиту awg (выберите архитектуру)
scp releases/tools/awg-mipsel root@router:/opt/tmp/awg
chmod +x /opt/tmp/awg
```

### 3. Замените модуль

```bash
# На роутере через SSH:
rmmod wireguard
insmod /opt/tmp/wireguard.ko
```

### 4. Настройте AWG-параметры

Создайте конфиг `/opt/tmp/tunnel.conf`:
```ini
[Interface]
PrivateKey = <your_private_key>
Jc = 4
Jmin = 40
Jmax = 70
S1 = 0
S2 = 0
H1 = 1
H2 = 2
H3 = 3
H4 = 4

[Peer]
PublicKey = <peer_public_key>
Endpoint = <server>:<port>
AllowedIPs = 0.0.0.0/0
PersistentKeepalive = 25
```

Примените:
```bash
/opt/tmp/awg setconf <interface_name> /opt/tmp/tunnel.conf
```

Где `<interface_name>` — системное имя WireGuard интерфейса (например `nwg0`).

## Сборка из исходников

### Требования

- [Keenetic SDK 4.03](https://github.com/nickstenning/keenetic-sdk) или аналог
- GCC кросс-компилятор (входит в SDK)

### Сборка модуля ядра

```bash
# 1. Скопируйте пакет в SDK
cp -r kernel-module/ <SDK>/package/kernel/amneziawg/

# 2. Сконфигурируйте SDK под модель роутера
cd <SDK>
./configure.sh KN-1810   # или другая модель

# 3. Включите модуль
echo 'CONFIG_PACKAGE_kmod-amneziawg=m' >> .config
make defconfig

# 4. Исправьте constexpr (GCC 14+)
# После первой неудачной сборки ядра:
sed -i 's/\bconstexpr\b/is_constexpr/g' \
  build_dir/target-*/linux-*/linux-4.9/scripts/unifdef.c

# 5. Соберите ядро (один раз для каждого таргета)
make target/linux/compile V=s

# 6. Соберите модуль
make package/kernel/amneziawg/compile V=s

# 7. Результат
ls build_dir/target-*/linux-*/amneziawg-*/src/wireguard.ko
```

### Сборка утилиты awg

```bash
# Клонируйте amneziawg-tools
git clone https://github.com/amnezia-vpn/amneziawg-tools.git
cd amneziawg-tools

# Примените патчи
cd src
patch -p2 < /path/to/awg-tools/patches/001-always-send-magic-header-as-string.patch
patch -p2 < /path/to/awg-tools/patches/002-fix-peer-uapi-compat.patch

# Соберите (пример для mipsel)
CC=<SDK>/staging_dir/toolchain-mipsel-linux-musl/bin/mipsel-ndms-linux-musl-gcc \
LDFLAGS="-static" make -j$(nproc) wg

# Результат
file wg   # ELF 32-bit LSB executable, MIPS...
```

## Патчи для модуля ядра

Применяются к [amneziawg-linux-kernel-module v1.0.20251104](https://github.com/amnezia-vpn/amneziawg-linux-kernel-module/tree/v1.0.20251104):

| # | Патч | Описание |
|---|---|---|
| 001 | `fix-zinc-include` | Добавляет `#include "crypto/zinc.h"` в main.c для kernel < 5.10 |
| 002 | `force-icmp-compat` | Принудительно использует compat-обёртки `__compat_icmp_ndo_send` для kernel < 4.10 (NDM ядро не экспортирует символ) |
| 003 | `fix-ndo-get-stats64` | Исправляет баг upstream: `#ifndef` → `#ifdef` COMPAT_CANNOT_USE_PCPU_STAT_TYPE для корректной статистики /sys/class/net/*/statistics/ |
| 004 | `keenetic-compat-rename-wireguard` | Переименовывает модуль: GENL_NAME "amneziawg"→"wireguard", GENL_VERSION 2→1, Kbuild amneziawg.o→wireguard.o |
| 005 | `keenetic-compat-guard-awg-attrs` | Убирает ВСЕ AWG-атрибуты из GET_DEVICE ответа (JC/S1-S4/H1-H4/I1-I5, peer FLAGS/ADVANCED_SECURITY) — NDMS ожидает только стандартные WG-атрибуты |
| 006 | `keenetic-peer-uapi-compat` | Добавляет FWMARK(11)/CLIENT_ID(12) в peer enum, сдвигает ADVANCED_SECURITY на позицию 13 — исправляет ERANGE(34) при SET_DEVICE от NDMS |

## Патчи для утилиты awg

Применяются к [amneziawg-tools v1.0.20250903](https://github.com/amnezia-vpn/amneziawg-tools/tree/v1.0.20250903):

| # | Патч | Описание |
|---|---|---|
| 001 | `always-send-magic-header-as-string` | H1-H4 всегда отправляются как NLA_NUL_STRING (не NLA_U32), т.к. GENL_VERSION=1 но ядро ожидает строки AWG v2.0 |
| 002 | `fix-peer-uapi-compat` | Добавляет FWMARK/CLIENT_ID в peer enum, сдвигает AWG на позицию 13 — должен совпадать с ядром |

## Версии

- **AmneziaWG**: v1.0.20251104 (AWG 2.0)
- **amneziawg-tools**: v1.0.20250903
- **Keenetic SDK**: 4.03
- **Kernel**: 4.9-ndm-5

## Зависимости модуля

```
depends: udp_tunnel, ip6_udp_tunnel
```

Эти модули обычно уже загружены если на роутере установлен WireGuard.

## Известные ограничения

- `awg show` не отображает AWG-параметры (JC, S1, H1...) — они убраны из GET_DEVICE для совместимости с NDMS. Параметры применены и активны, просто не видны через netlink.
- Для настройки AWG-параметров используйте `awg setconf` — NDMS настраивает только стандартные WG-параметры.

## Лицензия

GPL-2.0, как и оригинальные WireGuard/AmneziaWG.
