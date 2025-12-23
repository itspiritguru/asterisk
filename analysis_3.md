# 🔍 Анализ log14 — Найдена причина!

## 📊 Статус

| Сервер | Endpoints | Причина |
|--------|-----------|---------|
| **EU** | ✅ Работают | `ru_asterisk_vpn` виден, Contact: `10.0.0.2:5060` |
| **RU** | ❌ **No objects found!** | Файл не загружается |

---

## 🔴 ПРОБЛЕМА НАЙДЕНА!

### Строка 55 в логе:
```
; ===== RU -> EU (VPN) M-bM-^@M-^T M-PM-^QM-PM-^UM-PM-^W...
```

**Это UTF-8 кириллица!** (`М-PM-^Q` = русские буквы)

Asterisk **НЕ ПОДДЕРЖИВАЕТ UTF-8** в конфигах! Кириллица в комментариях ломает парсер!

---

## ✅ Решение — убрать кириллицу из pjsip_custom.conf

### На RU сервере:

```bash
# Создать чистый файл БЕЗ кириллицы
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
EOF

# Применить
fwconsole reload

# Проверить
asterisk -rx "pjsip show endpoints"
```

---

## ⚠️ Важно: Конфликт с GUI-транком mango_telecom

В веб-интерфейсе на RU уже есть транк:
```
Name: mango_telecom
Tech: pjsip
CallerID: 78122107151
Status: Enabled
```

**Может быть конфликт имён!** Если GUI создал свой `mango_telecom`, то ваш из `pjsip_custom.conf` может не работать.

### Варианты решения:

**Вариант 1:** Удалить транк из GUI и использовать только `pjsip_custom.conf`

**Вариант 2:** Переименовать endpoint в `pjsip_custom.conf`:
```ini
[mango_telecom_vpn]  ; другое имя
type=endpoint
...
```

**Вариант 3:** Использовать GUI-транк и не создавать свой в `pjsip_custom.conf`
- В этом случае нужно в GUI настроить правильный `context=from-mango-retell`

---

## 📋 Проверка после исправления

```bash
# 1. Проверить endpoints
asterisk -rx "pjsip show endpoints" | grep -E "eu_asterisk|mango"

# Ожидаемый вывод:
# Endpoint:  eu_asterisk_vpn                Not in use    0 of inf
# Endpoint:  mango_telecom                  Not in use    0 of inf

# 2. Проверить identifies  
asterisk -rx "pjsip show identifies"

# 3. Тест звонка
# Позвонить с Mango на user1 и смотреть лог:
asterisk -rvvvvv
pjsip set logger on
```

---

## 📊 Итоговая схема (после исправления)

```
EU Server (10.0.0.1):
  ✅ ru_asterisk_vpn     → 10.0.0.2:5060 (RU)
  ✅ retell_in           → Retell IPs
  ✅ retell_trunk2       → livekit.cloud (Available!)

RU Server (10.0.0.2):
  ⏳ eu_asterisk_vpn     → 10.0.0.1:5060 (EU)  ← нужно исправить
  ⏳ mango_telecom       → 81.88.86.35:5060    ← нужно исправить
```

