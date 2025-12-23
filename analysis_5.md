# Анализ err16 - Контексты перепутаны местами

## Дата: 2025-12-24

## Проблема

Звонки в обе стороны через VPN заканчиваются ошибкой **404 Not Found**.

### Логи показывают:

**На EU сервере:**
```
ru_asterisk_vpn: Call (UDP:10.0.0.2:5060) to extension '+37127873106' rejected 
because extension not found in context 'from-eu-vpn'.
```

**На RU сервере:**
```
eu_asterisk_vpn: Call (UDP:10.0.0.1:5060) to extension '+79164688596' rejected 
because extension not found in context 'from-ru-vpn'.
```

## Диагноз

**Контексты в `pjsip_custom.conf` перепутаны местами на обоих серверах!**

| Сервер | Endpoint | Сейчас | Надо |
|--------|----------|--------|------|
| EU | `ru_asterisk_vpn` | `context=from-eu-vpn` ❌ | `context=from-ru-vpn` ✅ |
| RU | `eu_asterisk_vpn` | `context=from-ru-vpn` ❌ | `context=from-eu-vpn` ✅ |

---

## Исправления

### На EU сервере

```bash
# Редактируем pjsip_custom.conf
nano /etc/asterisk/pjsip_custom.conf

# Найдите секцию [ru_asterisk_vpn] и измените:
# context=from-eu-vpn  <-- БЫЛО
# context=from-ru-vpn  <-- НАДО
```

**Правильная конфигурация `[ru_asterisk_vpn]` на EU:**
```ini
[ru_asterisk_vpn]
type=endpoint
context=from-ru-vpn    ; <-- Исправить!
disallow=all
allow=ulaw,alaw
aors=ru_asterisk_vpn_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes
```

---

### На RU сервере

```bash
# Редактируем pjsip_custom.conf
nano /etc/asterisk/pjsip_custom.conf

# Найдите секцию [eu_asterisk_vpn] и измените:
# context=from-ru-vpn  <-- БЫЛО
# context=from-eu-vpn  <-- НАДО
```

**Правильная конфигурация `[eu_asterisk_vpn]` на RU:**
```ini
[eu_asterisk_vpn]
type=endpoint
context=from-eu-vpn    ; <-- Исправить!
disallow=all
allow=ulaw,alaw
aors=eu_asterisk_vpn_aor
direct_media=no
rtp_symmetric=yes
force_rport=yes
rewrite_contact=yes
```

---

## Применение изменений

**На EU:**
```bash
fwconsole reload
# или
asterisk -rx "module reload res_pjsip.so"
```

**На RU:**
```bash
fwconsole reload
# или
asterisk -rx "module reload res_pjsip.so"
```

---

## Проверка после исправления

```bash
# Проверить что контексты правильные:
asterisk -rx "pjsip show endpoint ru_asterisk_vpn"  # на EU - должен быть from-ru-vpn
asterisk -rx "pjsip show endpoint eu_asterisk_vpn"  # на RU - должен быть from-eu-vpn
```

---

## Быстрый фикс через sed

**На EU:**
```bash
sed -i 's/context=from-eu-vpn/context=from-ru-vpn/' /etc/asterisk/pjsip_custom.conf
fwconsole reload
```

**На RU:**
```bash
sed -i 's/context=from-ru-vpn/context=from-eu-vpn/' /etc/asterisk/pjsip_custom.conf
fwconsole reload
```

---

## Итог

После исправления контекстов:
- Звонки от RU (`10.0.0.2`) на EU попадут в контекст `[from-ru-vpn]` → далее в Retell
- Звонки от EU (`10.0.0.1`) на RU попадут в контекст `[from-eu-vpn]` → далее в Mango

