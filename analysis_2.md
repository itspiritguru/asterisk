# 🔍 Анализ log13 — Endpoints НЕ создаются!

## 📊 Что показывает лог

### ✅ WireGuard работает
```
PING 10.0.0.1 (10.0.0.1) - 3 packets transmitted, 3 received, 0% packet loss
PING 10.0.0.2 (10.0.0.2) - 3 packets transmitted, 3 received, 0% packet loss
```

### ❌ КРИТИЧЕСКАЯ ПРОБЛЕМА: Endpoints не видны!

```bash
asterisk -rx "pjsip show endpoints" | grep -E "eu_asterisk|mango"
# ПУСТОЙ ВЫВОД!
```

Команда выполнена **дважды** после reload — и оба раза **пусто**!

### ✅ Identifies создаются нормально

**На RU:**
```
Identify:  eu_asterisk_vpn_identify/eu_asterisk_vpn
     Match: 10.0.0.1/32

Identify:  mango_telecom_identify/mango_telecom  
     Match: 81.88.86.35/32
     Match: 81.88.88.0/24
```

**На EU:**
```
Identify:  ru_asterisk_vpn_identify/ru_asterisk_vpn
     Match: 10.0.0.2/32
```

---

## 🔴 Диагноз

**Identifies существуют и указывают на endpoints, но сами endpoints НЕ СОЗДАНЫ!**

Это значит:
1. Файл `pjsip_custom.conf` читается (identify создаются)
2. Но секции `[endpoint]` игнорируются или имеют синтаксическую ошибку

---

## 🔧 Решение — Проверить и исправить

### Шаг 1: Проверить что endpoint-ы не создались

```bash
# На RU:
asterisk -rx "pjsip show endpoints"

# На EU:
asterisk -rx "pjsip show endpoints"
```

### Шаг 2: Проверить синтаксис файла

```bash
# На RU:
cat -A /etc/asterisk/pjsip_custom.conf | head -50
```
Если видите `^M` в конце строк — файл имеет Windows line endings!

### Шаг 3: Исправить line endings (если есть ^M)

```bash
sed -i 's/\r$//' /etc/asterisk/pjsip_custom.conf
fwconsole reload
```

### Шаг 4: Проверить что FreePBX включает pjsip_custom.conf

```bash
grep -r "pjsip_custom" /etc/asterisk/pjsip.conf
```

Должна быть строка:
```
#include pjsip_custom.conf
```

### Шаг 5: Проверить ошибки в логе Asterisk

```bash
asterisk -rx "core reload"
tail -100 /var/log/asterisk/full | grep -i error
```

---

## 📝 Правильный pjsip_custom.conf для RU

**ВАЖНО:** Убедитесь что нет лишних пробелов и переносов строк!

```ini
; ===== RU -> EU (VPN) =====
[eu_asterisk_vpn]
type=endpoint
context=from-ru-vpn
disallow=all
allow=ulaw,alaw
aors=eu_asterisk_vpn_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[eu_asterisk_vpn_aor]
type=aor
contact=sip:10.0.0.1:5060

[eu_asterisk_vpn_identify]
type=identify
endpoint=eu_asterisk_vpn
match=10.0.0.1

; ===== Mango Telecom =====
[mango_telecom]
type=endpoint
transport=0.0.0.0-udp
context=from-mango-retell
disallow=all
allow=ulaw,alaw,g729
aors=mango_telecom_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[mango_telecom_aor]
type=aor
contact=sip:81.88.86.35:5060

[mango_telecom_identify]
type=identify
endpoint=mango_telecom
match=81.88.86.35
match=81.88.88.0/24
```

---

## 📝 Правильный pjsip_custom.conf для EU

```ini
; ===== EU -> RU (VPN) =====
[ru_asterisk_vpn]
type=endpoint
context=from-eu-vpn
disallow=all
allow=ulaw,alaw
aors=ru_asterisk_vpn_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[ru_asterisk_vpn_aor]
type=aor
contact=sip:10.0.0.2:5060

[ru_asterisk_vpn_identify]
type=identify
endpoint=ru_asterisk_vpn
match=10.0.0.2

; ===== Retell Incoming =====
[retell_in]
type=endpoint
transport=0.0.0.0-udp
context=from-retell
disallow=all
allow=alaw,ulaw
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[retell_in_aor]
type=aor
contact=sip:5t4n6j0wnrl.sip.livekit.cloud

[retell_in_identify]
type=identify
endpoint=retell_in
match=130.162.187.104
match=143.223.88.0/21
match=161.115.160.0/19
match=18.98.16.0/24
match=89.168.121.54
```

---

## 🔄 Быстрое исправление (копипаст)

### На RU сервере:

```bash
# Бэкап
cp /etc/asterisk/pjsip_custom.conf /etc/asterisk/pjsip_custom.conf.bak2

# Создать чистый файл
cat > /etc/asterisk/pjsip_custom.conf << 'EOF'
[eu_asterisk_vpn]
type=endpoint
context=from-ru-vpn
disallow=all
allow=ulaw,alaw
aors=eu_asterisk_vpn_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[eu_asterisk_vpn_aor]
type=aor
contact=sip:10.0.0.1:5060

[eu_asterisk_vpn_identify]
type=identify
endpoint=eu_asterisk_vpn
match=10.0.0.1

[mango_telecom]
type=endpoint
transport=0.0.0.0-udp
context=from-mango-retell
disallow=all
allow=ulaw,alaw,g729
aors=mango_telecom_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[mango_telecom_aor]
type=aor
contact=sip:81.88.86.35:5060

[mango_telecom_identify]
type=identify
endpoint=mango_telecom
match=81.88.86.35
match=81.88.88.0/24
EOF

# Применить
fwconsole reload

# Проверить
asterisk -rx "pjsip show endpoints" | grep -E "eu_asterisk|mango"
```

### На EU сервере:

```bash
# Бэкап
cp /etc/asterisk/pjsip_custom.conf /etc/asterisk/pjsip_custom.conf.bak2

# Создать чистый файл
cat > /etc/asterisk/pjsip_custom.conf << 'EOF'
[ru_asterisk_vpn]
type=endpoint
context=from-eu-vpn
disallow=all
allow=ulaw,alaw
aors=ru_asterisk_vpn_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[ru_asterisk_vpn_aor]
type=aor
contact=sip:10.0.0.2:5060

[ru_asterisk_vpn_identify]
type=identify
endpoint=ru_asterisk_vpn
match=10.0.0.2

[retell_in]
type=endpoint
transport=0.0.0.0-udp
context=from-retell
disallow=all
allow=alaw,ulaw
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes

[retell_in_aor]
type=aor
contact=sip:5t4n6j0wnrl.sip.livekit.cloud

[retell_in_identify]
type=identify
endpoint=retell_in
match=130.162.187.104
match=143.223.88.0/21
match=161.115.160.0/19
match=18.98.16.0/24
match=89.168.121.54
EOF

# Применить
fwconsole reload

# Проверить
asterisk -rx "pjsip show endpoints" | grep -E "ru_asterisk|retell_in"
```

---

## ✅ Ожидаемый результат после исправления

**На RU:**
```
asterisk -rx "pjsip show endpoints" | grep -E "eu_asterisk|mango"
 Endpoint:  eu_asterisk_vpn              Unavailable   0 of inf
 Endpoint:  mango_telecom                Unavailable   0 of inf
```

**На EU:**
```
asterisk -rx "pjsip show endpoints" | grep -E "ru_asterisk|retell"
 Endpoint:  ru_asterisk_vpn              Unavailable   0 of inf
 Endpoint:  retell_in                    Unavailable   0 of inf
```

