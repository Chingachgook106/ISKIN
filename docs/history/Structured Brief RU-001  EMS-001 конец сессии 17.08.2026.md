---

# 📋 Structured Brief: RU-001 / EMS-001 (конец сессии 17.08.2026)

## ✅ Что сделано в этой сессии

**1. Физический инвентарь EMS-001 (Малыш):**
- Wi-Fi модуль: **MediaTek MT7920 Wi-Fi 6** (ifIndex 12, статус Disconnected)
- Ethernet: **Realtek PCIe GbE #2** (ifIndex 16, MAC B0-41-6F-0C-78-2C, активен)
- DNS leak зафиксирован: `{8.8.8.8, 1.1.1.1}` → физический интерфейс

**2. Архитектурные решения (ADR-001 обновлён):**
- ❌ Oracle Cloud — отклонён окончательно
- ❌ Compute Node как Default Gateway — запрещён
- ✅ Compute Node = **Local VPS endpoint** только для Windows+Microsoft трафика
- ✅ Fallback A = **Psiphon на телефоне** + USB Tethering/Wi-Fi
- ✅ Gov RU / Банки = **Out of Scope** (Малыш закрыт для них)
- ✅ Split Tunnel: 3 outbound → External VPS / Local VPS / Block

**3. Уточнённая формулировка Q1 (Zero Trust):**
> *"Compute Node не должен иметь возможности самостоятельно перехватить или маршрутизировать трафик, не предназначенный для явно разрешённого Windows + Microsoft traffic path"*

**4. Стратегия запуска:** Стратегия B (Staged Rollout) — WFP Base Layer с временным глобальным исключением для Microsoft traffic.

**5. Создана точка восстановления:**
- `RU-001-BASELINE-PATCHED` (Sequence 6, 17.08.2026 01:42)
- Служба VSS запущена, System Restore на C:\ включён

**6. Phase 2.2 — Minimal Prototype (ЧАСТИЧНО ВЫПОЛНЕН):**
- ✅ sing-box 1.13.18 + wintun.dll установлены на **обоих** ПК
- ✅ Сервер на Большом ПК: слушает порт 8443, inbound vless-in работает
- ✅ Клиент на Малыше: **TUN-адаптер создан** (tun0, ifIndex 48)
- ✅ `auto_route: true` работает — трафик заворачивается в TUN
- ❌ VLESS Reality-handshake падает

---

## ⚠️ Текущее состояние (БЛОКЕР)

**Симптом:** каждое VLESS-соединение рвётся с `EOF` (клиент) / `REALITY: processed invalid connection` (сервер).

**Что проверено:**
- ✅ `www.microsoft.com:443` доступен с Большого ПК (`TcpTestSucceeded: True`)
- ✅ Часы на обоих ПК синхронизированы (рассинхрон был до ~минуты)
- ❌ После синхронизации времени EOF продолжается — проблема глубже
- ❌ `server.log` на Большом ПК отсутствует — возможно, сервер упал или лог идёт в stdout

**Следствие:** У Малыша сейчас НЕТ интернета (весь трафик уходит в TUN → vless-out → EOF). Нужно либо наладить VLESS, либо остановить клиент (Ctrl+C) для восстановления обычного интернета.

---

## 🗝 Ключевые секреты (сохранено!)

Буратино

---

## 📁 Структура файлов

**Большой ПК (192.168.0.100):**
```
C:\sing-box\sing-box-1.13.18-windows-amd64\
├── sing-box.exe      (45481472 байт)
├── wintun.dll        (427552 байт)
├── libcronet.dll
├── LICENSE
└── server.json
```

**Малыш (192.168.0.102, пользователь `indeez`):**
```
C:\sing-box\sing-box-1.13.18-windows-amd64\
├── sing-box.exe
├── wintun.dll
├── libcronet.dll
├── LICENSE
├── sing-box.json     ← КОНФИГ КЛИЕНТА (имя нестандартное!)
└── client.log        ← логи клиента пишутся сюда
```

⚠️ **Важно:** конфиг клиента называется `sing-box.json`, а не `client.json`. Все команды: `.\sing-box.exe run -c sing-box.json`

---

## 🎯 Следующие шаги в новой сессии

1. **Проверить, жив ли сервер** на Большом ПК (`Get-Process sing-box`, порт 8443). Если нет — перезапустить.
2. **Разобраться с Reality-handshake:**
   - Проверить server.log (или stdout сервера) на предмет точной причины `invalid connection`
   - Возможные действия: пересоздать ключи Reality, сменить `server_name` на другой доступный домен (Samsung/Apple/Cloudflare), попробовать другой `fingerprint` utls (firefox/safari)
3. **После успешного handshake:** проверить egress IP на Малыше (должен стать IP Большого ПК)
4. **Phase 2.3:** Настройка WFP Base Layer на Малыше (блокировка физических интерфейсов, разрешение TUN + временное исключение Microsoft)

---

Конфиги `server.json` и `sing-box.json` (client) отправляю отдельным сообщением — чтобы можно было скопировать целиком без разбора по блоку.