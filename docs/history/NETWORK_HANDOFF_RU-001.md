# 📦 STRUCTURED BRIEF: ISKIN / RU-001
**Session Handoff → New Chat**
**Дата:** 13.08.2026
**Роли:** Манни (Заказчик / Со-архитектор) ↔ Асси (Ведущий инженер)
**Общение:** на «ты», строго по фактам, без непроверенных данных.
**Инженерный протокол:** `Почему / Что / Зачем / Проверка`. Никаких изменений «на всякий случай».

---

## 1. PROJECT ARCHITECTURE (ISKIN ADR-001)

| Узел | Роль | Железо / ОС | Сеть |
|---|---|---|---|
| **EMS-001 «Малыш»** | Control / Orchestration | Beelink 5800H, Win 11 Pro 25H2 (Build 26200.9168) | `192.168.0.102/24` (Ethernet 4) |
| **Compute Node (Большой ПК)** | Heavy GPU/LLM | Отдельное железо | `192.168.0.100` |
| **CR-001** | SMB Share | `\\192.168.0.102\iskin-exchange` ↔ `C:\iskin-exchange` | Только в локалке |
| **RU-001** | Network Security Layer | Текущий объект проектирования | — |

**Правило:** EMS-001 автономен и легковесен. Docker и тяжелые серверные компоненты на него не ставим без явного ADR.

---

## 2. COMPLETED WORK

### ✅ 2.1 Network Baseline (RU-001-NB-001) — ЗАВЕРШЕНО
Снимок сети до начала работ зафиксирован в `RU-001-Network-Baseline-001.md`:
- DNS hardcoded: `8.8.8.8`, `1.1.1.1` (высокий риск утечки).
- IPv6 stack включён, но global address / default route отсутствуют.
- Единственный маршрут по умолчанию через `192.168.0.1`.
- Зафиксированы утечки DNS на физический интерфейс (Google/Cloudflare).
- Зафиксирован workaround SMB-глюка Windows 11 (WMP Prompt) через UNC-пути без установки WMP.

### ✅ 2.2 External Expert Review (Sakana + Perplexity Comet) — ЗАВЕРШЕНО
Проведены две независимые экспертизы (stress-test архитектуры и real-world DPI РФ 2026). Ключевые выводы зафиксированы в секции 4.

### ✅ 2.3 System Snapshots — СОЗДАНЫ
- `RU-001-BASELINE-BEFORE-VPN` — точка до обновлений.
- Обновление Windows **KB5121003** применено, EMS-001 перезагружен.
- **Следующий шаг:** создать снапшот `RU-001-BASELINE-PATCHED` после успешной загрузки — это будет истинная точка отката для Phase 2.

---

## 3. TZ RU-001 — КРАТКАЯ ВЫЖИМКА (P0 REQUIREMENTS)

Полное ТЗ лежит в файле `ТЗ для защиты RU-001 for Qwen.md`. Ниже — инварианты:

1. **System-wide protected path** ниже уровня приложений (Git, VS Code, SSH, PowerShell должны работать без per-app конфигурации).
2. **FAIL-CLOSED** — главный инвариант. При отсутствии защищённого пути интернет **полностью недоступен**. Состояние `VPN unavailable → Internet unavailable` считается нормой.
3. **DNS containment** — DNS не должен тихо уходить на физический интерфейс.
4. **IPv6 policy** — либо полностью через туннель, либо полностью заблокирован (выбрали второе).
5. **Persist through reboot / sleep-wake / network change.**
6. **Источник истины** — не зелёная галочка VPN-клиента, а фактическое выполнение политики (routing + WFP + DNS + egress).
7. **Модель угроз:** защищаемся от приложений/служб/фоновых процессов. **Не защищаемся** от локального администратора (Манни).
8. **Emergency Override:** ручная «красная кнопка» для временного снятия периметра по решению админа.
9. **Compute Node:** прямо сейчас connectivity не закладываем, но архитектурно не отрезаем.

---

## 4. ARCHITECTURAL DECISIONS (Принятые)

### 🎯 Vector Alpha Refined — УТВЕРЖДЁН
**Выбрано:** Branch B — Third-party System VPN (`sing-box` + `wintun` + **VLESS/Reality**) + собственный WFP enforcement layer + event-driven watchdog.

**Отклонено:**
- **Branch A (Native Windows VPN + LockDown):** требует IKEv2, который легко детектируется ТСПУ. (REJECTED, reason: DPI survival).
- **Branch C (Overlay типа Tailscale/ZeroTier):** недостаточный контроль над default route и WFP для RU-001.
- **Proxy (Browsec/Telegram-прокси):** только per-app, не даёт system-wide coverage. Оставлен как application-layer fallback.
- **GoodbyeDPI / zapret / ByeDPI:** не создают TUN-адаптер, не шифруют трафик, провайдер видит destination IP. Категория «DPI circumvention», не «Protected Path».
- **Cloudflare WARP:** VERIFIED — системно троттлится/блокируется в РФ с 09.06.2025 (~16 КБ/с, packet loss). MASQUE/QUIC не спасают.
- **Amnezia Free:** не system-wide (только социально значимые сайты).

### 🛡 WFP Architecture (Hardened)
```
Base Layer (boot-time, persistent, lowest weight):
  BLOCK all outbound IPv4/IPv6 on physical interfaces
  ALLOW DHCP/ARP (локалка)
  ALLOW TCP to VPS endpoint (Bootstrap exception)
  BLOCK LLMNR (UDP 5355), NetBIOS (UDP 137-139), mDNS (UDP 5353)
  BLOCK multicast/broadcast (224.0.0.0/4, ff02::/16)

Dynamic Layer (runtime, managed by watchdog):
  ALLOW all outbound if InterfaceIndex == CURRENT_VPN_TUN_INDEX
```

### 🧠 Event-Driven Watchdog (Critical Findings от Sakana)
Никакого 200ms polling. Только реакции на события:
- `NotifyIpInterfaceChange` (ловим смену IfIndex TUN-адаптера).
- `RegisterPowerSettingNotification` (PBT_APMRESUMEAUTOMATIC для sleep/wake).
- **L3 health check** (ICMP/HTTP через TUN каждые ~5s) — проверяем не наличие адаптера, а реальную связность.
- При любом подозрительном событии → немедленно сносим Dynamic ALLOW (Fail-Closed).

---

## 5. CUSTOMER ANSWERS Q1–Q5 (Зафиксировано)

| # | Вопрос | Решение |
|---|---|---|
| **Q1** | Local LAN | **Вариант 2 (Allow-list)**. Доступ к Большому ПК `192.168.0.100` (Compute workflow) + роутеру. Жёсткий запрет использовать его как Default Gateway. |
| **Q2** | IPv6 | **Полная блокировка** (до доказательства безопасного IPv6-over-tunnel). |
| **Q3** | VPN unavailable | **Fail-Closed норма.** Ручной Emergency Override. Комбинация: Redundancy + LAN Lifeline + Red Button. |
| **Q4** | Administrator threat | **Не защищаемся** от Манни. |
| **Q5** | Compute Node | **Не закладываем сейчас**, не блокируем в будущем. |

---

## 6. 🌍 ОТДЕЛЬНАЯ ВЕТКА: FREE INTERNET ACCESS STRATEGY

**Задача:** обеспечить EMS-001 свободным исходящим интернетом **из любого региона планеты**, начиная с РФ. Транспортно-независимо от периметра RU-001.

### Текущая оценка ландшафта (август 2026)

| Tier | Транспорт | Цена | DPI (РФ) | System-wide | Статус |
|---|---|---|---|---|---|
| **Tier 0** | Cloudflare WARP | Free | ❌ DEAD (VERIFIED) | ✅ | **REJECTED для RU** |
| **Tier 1** | VLESS+Reality (Oracle Free / сервер друга / микро-VPS за крипту) | Free / ~$2/мес | ✅ Сильный | ✅ | **PRIMARY TARGET** |
| **Tier 2** | Премиум VPS | $5-10/мес | ✅ Сильный | ✅ | Для тяжёлых задач |
| **Fallback A** | Psiphon (VPN mode) | Free | ⚠️ Средний | ✅ | Временная затычка |
| **Fallback B** | Tor + tun2socks | Free | ✅ с мостами | ✅ | Аварийный канал, высокая латентность |
| **Out of scope** | GoodbyeDPI / zapret | Free | ⚠️ Средний | ❌ | Не создаёт TUN |
| **Out of scope** | Telegram-конфиги | Free | ⚠️ | ⚠️ | Риск MITM/honeypot |

### Что предстоит проработать в новом чате

1. **Oracle Cloud Always Free** — есть ли у Манни зарубежная карта?
2. **Микро-VPS за крипту/рубли** (Aeza, PQ Hosting, Timeweb Cloud) — выбор локации НЕ в РФ.
3. **Peer-to-peer** с другом за границей.
4. **Мобильность EMS-001:** как меняется архитектура при переезде в Европу/Азию (исчезает угроза DPI, остаётся угроза публичных Wi-Fi).

---

## 7. NEXT STEPS (Что делаем сразу в новом чате)

1. **Подтвердить снапшот** `RU-001-BASELINE-PATCHED` (winver → 26200.9168, сеть OK).
2. **Запустить Phase 2.2 (Minimal Prototype):**
   - Определиться с Tier 1 endpoint (VPS).
   - Скачать `sing-box` + `wintun.dll` на EMS-001.
   - Поднять VLESS+Reality сервер (если выбран путь self-hosted).
   - Запустить в консоли, проверить TUN-адаптер и смену egress IP.
3. **Запустить Phase 2.3 (WFP Perimeter):**
   - Настроить WFAS (boot-time filters + bootstrap exception).
   - Проверить fail-closed (без sing-box → интернета нет).
4. **Phase 3 (Adversarial Testing):**
   - Boot leak test (Wireshark).
   - Sleep/Wake leak test.
   - IfIndex volatility test (5 reboots).
   - Tunnel crash test (`taskkill /F`).

---

## 8. RULES TO CARRY OVER

- **Security ≠ Privacy ≠ Anonymization.**
- **Baseline — абсолютный источник истины** для сетевой диагностики.
- **Правило безопасного эксперимента:** `BASELINE → SNAPSHOT → CHANGE → TEST → VERIFY → ROLLBACK or ACCEPT`.
- **Классификация утверждений:**
  - `FACT` — наблюдалось непосредственно на EMS-001.
  - `DOCUMENTED` — подтверждено документацией Microsoft/вендора.
  - `INFERENCE` — следует из архитектуры.
  - `UNKNOWN` — требует эксперимента (говорим «у меня нет этой информации», а не выдумываем).
- **Deliverables (согласно ТЗ п. 27):** D1-D8 (ADR-001, Implementation Spec, Threat Matrix, Test Plan, Acceptance Matrix, Evidence Package).

---