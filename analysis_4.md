# 🔍 Анализ err15 — Несоответствие имён!

## 🔴 ОШИБКА В КОНФИГЕ

### В pjsip_custom.conf:
```ini
[mango_telecom_vpn]      ← endpoint называется mango_telecom_VPN
type=endpoint
...

[mango_telecom_identify]
type=identify
endpoint=mango_telecom   ← указывает на mango_telecom (БЕЗ _vpn)!
```

**Identify указывает на несуществующий endpoint!**

---

## ✅ Исправление

### Вариант 1: Исправить identify (если хотите оставить `_vpn`)

```bash
nano /etc/asterisk/pjsip_custom.conf
```

Изменить:
```ini
[mango_telecom_identify]
type=identify
endpoint=mango_telecom_vpn   ← добавить _vpn
match=81.88.86.35
match=81.88.88.0/24
```

### Вариант 2: Убрать `_vpn` из имени endpoint (проще)

```bash
cat > /etc/asterisk/pjsip_custom.conf << 'EOF'
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
EOF

fwconsole reload
```

**НО!** Для Mango лучше использовать GUI-транк который уже есть!

---

## ⚠️ Рекомендация: Использовать GUI-транк для Mango

В GUI уже есть транк `mango_telecom`. Его identify уже работает:
```
Identify:  mango_telecom/mango_telecom
     Match: 81.88.86.35/32
```

**Нужно только проверить context в GUI:**

1. Открыть FreePBX Web UI → Connectivity → Trunks → `mango_telecom`
2. Найти поле **Context** в настройках PJSIP
3. Установить: `from-mango-retell`
4. Submit + Apply Config

Тогда в `pjsip_custom.conf` НЕ НУЖНО создавать mango_telecom — только VPN endpoint:

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
```

---

## 📋 Проверка

После изменений:
```bash
fwconsole reload
asterisk -rx "pjsip show endpoints" | grep -E "eu_asterisk|mango"
```

Должно показать ОБА endpoint'а:
```
Endpoint:  eu_asterisk_vpn              Not in use    0 of inf
Endpoint:  mango_telecom                Not in use    0 of inf
```

