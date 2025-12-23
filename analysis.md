# 🔍 Анализ конфигурации Asterisk EU/RU

## 📋 Найденные проблемы

### ❌ Критическая ошибка #1 — RU: Endpoints не найдены!

В логе `log_12` видны ошибки:
```
WARNING: Identify 'eu_asterisk_vpn_identify' points to endpoint 'eu_asterisk_vpn' but endpoint could not be found
WARNING: Identify 'mango_telecom_identify' points to endpoint 'mango_telecom' but endpoint could not be found
```

**Причина:** В `pjsip_custom.conf` на RU секции endpoint и aor имеют **одинаковые имена**, что вызывает конфликт:
```ini
[eu_asterisk_vpn]    # ← endpoint
type=endpoint
...

[eu_asterisk_vpn]    # ← aor (КОНФЛИКТ!)
type=aor
```

### ❌ Критическая ошибка #2 — EU: Нет auth для VPN

```
ERROR: ru_asterisk_vpn:10.0.0.2: There were no auth ids available
```

RU отвечает `401 Unauthorized`, но у EU нет credentials для авторизации.

### ❌ Ошибка #3 — Отсутствует aors= в endpoint mango_telecom

Endpoint `[mango_telecom]` не имеет ссылки на AOR.

---

## ✅ Исправленная конфигурация

### 🇪🇺 EU Server — `/etc/asterisk/pjsip_custom.conf`

```ini
; ===== EU -> RU (VPN) — БЕЗ АУТЕНТИФИКАЦИИ =====
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
match=130.162.187.104/32
match=143.223.88.0/21
match=161.115.160.0/19
match=18.98.16.0/24
match=89.168.121.54/32
```

---

### 🇷🇺 RU Server — `/etc/asterisk/pjsip_custom.conf`

```ini
; ===== RU -> EU (VPN) — БЕЗ АУТЕНТИФИКАЦИИ =====
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
match=81.88.86.35/32
match=81.88.88.0/24
```

---

## 📝 Исправленный extensions_custom.conf

### 🇪🇺 EU Server

```asterisk
[from-retell]
exten => _X.,1,NoOp(=== IN Retell: exten=${EXTEN} clid=${CALLERID(all)} ===)
 same => n,Set(NUM_RAW=${FILTER(0-9,${EXTEN})})
 same => n,Set(NUM10=${NUM_RAW:-10})
 same => n,Set(NUM_E164=+7${NUM10})
 same => n,NoOp(=== To RU via VPN: ${NUM_E164} ===)
 same => n,Dial(PJSIP/${NUM_E164}@ru_asterisk_vpn,60)
 same => n,Hangup()

exten => _+X.,1,NoOp(=== IN Retell: exten=${EXTEN} clid=${CALLERID(all)} ===)
 same => n,Set(NUM_RAW=${FILTER(0-9,${EXTEN})})
 same => n,Set(NUM10=${NUM_RAW:-10})
 same => n,Set(NUM_E164=+7${NUM10})
 same => n,NoOp(=== To RU via VPN: ${NUM_E164} ===)
 same => n,Dial(PJSIP/${NUM_E164}@ru_asterisk_vpn,60)
 same => n,Hangup()

[from-ru-vpn]
; Входящие от RU — передаём CallerID как есть
exten => _+X.,1,NoOp(=== IN from RU VPN: exten=${EXTEN} clid=${CALLERID(all)} ===)
 same => n,Dial(PJSIP/${EXTEN}@retell_trunk2,60)
 same => n,Hangup()

exten => _X.,1,NoOp(=== IN from RU VPN: exten=${EXTEN} clid=${CALLERID(all)} ===)
 same => n,Dial(PJSIP/${EXTEN}@retell_trunk2,60)
 same => n,Hangup()
```

### 🇷🇺 RU Server

```asterisk
[from-mango-retell]
exten => user1,1,NoOp(=== IN from Mango: clid=${CALLERID(all)} ===)
 same => n,Set(ORIGINAL_CLID=${CALLERID(num)})
 same => n,Set(NUM_RAW=${FILTER(0-9,${ORIGINAL_CLID})})
 same => n,Set(NUM10=${NUM_RAW:-10})
 ; Формат +91XXXXXXXXXX для Retell (как работало раньше)
 same => n,Set(CALLERID(all)=+91${NUM10})
 same => n,NoOp(=== Forwarding to EU: CallerID=${CALLERID(all)} ===)
 same => n,Dial(PJSIP/+37127873106@eu_asterisk_vpn,60)
 same => n,Hangup()

[from-eu-vpn]
exten => _+7X.,1,NoOp(=== IN from EU VPN: exten=${EXTEN} ===)
 same => n,Dial(PJSIP/${EXTEN}@mango_telecom,60)
 same => n,Hangup()

exten => _7X.,1,NoOp(=== IN from EU VPN: exten=${EXTEN} ===)
 same => n,Dial(PJSIP/+${EXTEN}@mango_telecom,60)
 same => n,Hangup()
```

---

## 🔧 Инструкция по применению

### Шаг 1: На EU сервере
```bash
# Редактировать pjsip_custom.conf
nano /etc/asterisk/pjsip_custom.conf
# Вставить исправленную конфигурацию EU

# Редактировать extensions_custom.conf
nano /etc/asterisk/extensions_custom.conf
# Вставить исправленную конфигурацию EU

# Применить
fwconsole reload
```

### Шаг 2: На RU сервере
```bash
# Редактировать pjsip_custom.conf
nano /etc/asterisk/pjsip_custom.conf
# Вставить исправленную конфигурацию RU

# Редактировать extensions_custom.conf
nano /etc/asterisk/extensions_custom.conf
# Вставить исправленную конфигурацию RU

# Применить
fwconsole reload
```

### Шаг 3: Проверить что endpoints видны
```bash
# На EU:
asterisk -rx "pjsip show endpoints" | grep -E "ru_asterisk|retell"

# На RU:
asterisk -rx "pjsip show endpoints" | grep -E "eu_asterisk|mango"
```

### Шаг 4: Проверить identify
```bash
# На EU:
asterisk -rx "pjsip show identifies"

# На RU:
asterisk -rx "pjsip show identifies"
```

---

## ⚠️ Почему перестало работать через 15 минут?

Из файла `problems`:
> "спустя минут 15 перестало работать"

Возможные причины:

1. **FreePBX перезаписал pjsip_custom.conf** при Apply Config из GUI
2. **Firewall сбросился** — правила iptables не были сохранены
3. **Asterisk перезапустился** и не загрузил правильные конфиги

**Решение:**
```bash
# Сделать правила iptables постоянными:
apt install iptables-persistent -y
iptables -A INPUT -i wg0 -p udp --dport 5060 -j ACCEPT
iptables -A INPUT -i wg0 -p udp --dport 10000:20000 -j ACCEPT
netfilter-persistent save

# Проверить что WireGuard работает:
wg show
ping -c 3 10.0.0.2  # с EU
ping -c 3 10.0.0.1  # с RU
```

---

## 📊 Схема потока звонков

```
┌─────────────┐    INVITE     ┌─────────────┐   VPN (wg0)   ┌─────────────┐    INVITE    ┌─────────────┐
│   Retell    │ ────────────► │     EU      │ ────────────► │     RU      │ ───────────► │    Mango    │
│             │               │ 10.0.0.1    │               │ 10.0.0.2    │              │             │
│ +37127..    │ ◄──────────── │from-retell  │ ◄──────────── │from-eu-vpn  │ ◄─────────── │ 79164688596 │
└─────────────┘    200 OK     └─────────────┘    200 OK     └─────────────┘   200 OK     └─────────────┘

Обратный путь (Mango → Retell):
┌─────────────┐    INVITE     ┌─────────────┐   VPN (wg0)   ┌─────────────┐    INVITE    ┌─────────────┐
│    Mango    │ ────────────► │     RU      │ ────────────► │     EU      │ ───────────► │   Retell    │
│ 79164688596 │               │from-mango   │               │from-ru-vpn  │              │             │
│             │               │ CallerID:   │               │             │              │ видит:      │
│             │               │ +919164..   │               │             │              │ +919164..   │
└─────────────┘               └─────────────┘               └─────────────┘              └─────────────┘
```

