---

# ТЗ ведущему инженеру Qwen

## RU-001 — проектирование и экспериментальная верификация сетевого защитного слоя EMS-001

**Дата постановки:** 13.08.2026
**Объект:** EMS-001 / «Малыш»
**Роль исполнителя:** ведущий инженер / network security architect
**Режим:** исследование → проектирование → безопасный эксперимент → отчёт
**Приоритет:** корректность и доказуемость выше простоты реализации

---

## 0. Главная задача

Спроектировать и подготовить к реализации **RU-001 Network Security Layer** для EMS-001 — Windows 11 Pro 25H2.

RU-001 должен создать единый системный сетевой защитный слой **ниже уровня приложений**, обеспечивающий:

1. системное покрытие сетевого трафика;
2. защищённый исходящий Internet path;
3. fail-closed;
4. отсутствие незаметного direct-Internet fallback;
5. контроль DNS;
6. контроль IPv4;
7. явную политику IPv6;
8. сохранение политики при reboot / sleep-wake / смене сети;
9. отсутствие необходимости конфигурировать Git, VS Code, SSH, Terminal и другие приложения по отдельности;
10. возможность последующего расширения архитектуры для Compute Node.

**Необходимо проектировать свойства системы, а не конкретный VPN-клиент.**

Конкретный продукт/протокол выбирается только после технического сравнения кандидатов и проверки на EMS-001.

---

# 1. Авторитетный baseline

Использовать в качестве исходного технического состояния:

**`RU-001-Network-Baseline-001.md`**

Baseline фиксирует:

* Windows 11 Pro Build 26200 / 25H2;
* x64 Beelink 5800H;
* активный Ethernet 4;
* IPv4 `192.168.0.102/24`;
* default gateway `192.168.0.1`;
* DNS `8.8.8.8`, `1.1.1.1`;
* отсутствие глобального IPv6;
* отсутствие IPv6 default route;
* отсутствие VPN / Proxy / Overlay;
* наличие прямого Internet;
* работоспособность HTTPS/GitHub.   

Baseline считать **фактом**, а не предположением.

Не изменять baseline задним числом.

---

# 2. Архитектурный вывод внешней экспертизы

Три независимых экспертных заключения дали одинаковый основной результат:

### Preferred

**System-level VPN**

### Second

**System-level Overlay**

### Rejected as primary perimeter

**Proxy**

Copilot рекомендует System VPN с force-tunnel, virtual adapter и WFP; Sakana даёт аналогичный вывод; Perplexity также ставит System VPN на первое место и подчёркивает необходимость независимой проверки fail-closed.   

### Следовательно

Не тратить основной инженерный ресурс на Proxy как на кандидат RU-001 perimeter.

Proxy допускается только как **дополнительный application-layer механизм**, если он потребуется в будущем.

---

# 3. Ключевой архитектурный принцип

Не использовать следующее равенство:

```text
VPN = connected
        ↓
system = secure
```

Оно неверно.

Требуется:

```text
VPN / tunnel state
        +
routing state
        +
firewall/WFP state
        +
DNS state
        +
IPv6 state
        +
actual observed egress
        ↓
RU-001 compliant
```

То есть **источником истины является фактическое выполнение security policy**, а не индикатор VPN-клиента.

---

# 4. Главный инвариант — FAIL-CLOSED

Это **P0 requirement**.

Определение:

> При отсутствии работоспособного и авторизованного защищённого сетевого пути EMS-001 не должен иметь возможности осуществлять неразрешённый исходящий Internet traffic.

Следовательно:

```text
Protected path UP
        ↓
Internet traffic = ALLOWED
        ↓
ONLY through protected path


Protected path DOWN
        ↓
Internet traffic = BLOCKED
```

Запрещено состояние:

```text
VPN DOWN
   ↓
Windows restores ordinary Internet route
   ↓
Traffic continues normally
```

Это считается **критическим дефектом**.

---

# 5. Не навязывать конкретный механизм fail-closed

Инженер должен сравнить минимум следующие механизмы:

### Branch A — Native Windows VPN

Проверить возможность комбинации:

* Windows built-in VPN;
* Force Tunnel;
* Always On;
* VPN LockDown;
* DNS/NRPT;
* traffic filtering;
* Windows Firewall/WFP.

Microsoft документирует Force Tunnel для Windows VPN и отдельно VPN LockDown, который при недоступном VPN блокирует outbound traffic; LockDown для встроенного VPN ограничен IKEv2. ([Microsoft Learn][1])

### Branch B — Third-party system VPN

Рассмотреть только решения, которые:

* создают системный virtual network interface;
* поддерживают system-wide routing;
* имеют проверяемый kill-switch/fail-closed;
* корректно работают на Windows 11;
* допускают автоматический старт;
* позволяют диагностировать routing/firewall state.

### Branch C — System-level Overlay

Рассмотреть только если overlay:

* создаёт системный virtual interface;
* позволяет управлять default routing;
* допускает fail-closed;
* позволяет контролировать IPv4/IPv6;
* имеет приемлемый control plane;
* не создаёт неожиданный direct egress через NAT traversal.

**Если Branch C оказывается технически равным или превосходящим A по требованиям RU-001, представить это как отдельный архитектурный вариант, а не отвергать Overlay по названию.**

---

# 6. Предпочтительная ветка

На первом проходе:

> **исследовать Native Windows VPN + LockDown/Force Tunnel как baseline implementation.**

Причина: Windows уже имеет системные механизмы VPN-профилей, forced tunneling, traffic filters и LockDown. ([Microsoft Learn][2])

Но:

> **не считать Native Windows VPN победителем до экспериментальной проверки на EMS-001.**

Если Native Windows VPN не обеспечивает весь набор требований, перейти к Branch B.

---

# 7. Routing requirements

Не требовать буквально удаления физического default route.

Требовать:

> При активном защищённом состоянии Internet traffic должен маршрутизироваться через защищённый интерфейс; при неактивном защищённом состоянии direct Internet egress должен быть невозможен.

Windows Force Tunnel может реализовываться через VPN IPv4/IPv6 default routes с более низкой метрикой, чем маршруты других интерфейсов. ([Microsoft Learn][1])

Инженер обязан определить:

* routing table до VPN;
* routing table при VPN UP;
* routing table при VPN DOWN;
* routing table после reboot;
* routing table после sleep/wake;
* routing table после network change.

---

# 8. Firewall / WFP

Firewall/WFP рассматривать как **второй уровень enforcement**, а не как декоративное дополнение.

Windows Firewall фильтрует входящий и исходящий трафик, а WFP предоставляет системную инфраструктуру фильтрации на разных уровнях сетевого стека. ([Microsoft Learn][3])

Проверить возможность построения политики:

```text
ALLOW:
    protected tunnel traffic
    required tunnel endpoint traffic
    explicitly approved local traffic

DENY:
    unprotected Internet egress
```

Особое внимание:

* IPv4;
* IPv6;
* UDP;
* TCP;
* ICMP;
* DNS;
* raw sockets;
* системные службы.

WFP/ALE способен контролировать создание outbound connections и socket operations, поэтому использовать его возможности там, где это действительно повышает доказуемость политики. ([Microsoft Learn][4])

---

# 9. Важное ограничение модели угроз

Не обещать защиту от локального администратора.

Если пользователь обладает полными административными правами, он потенциально способен изменить локальную сетевую политику.

Поэтому разделить:

### Threat model A

Обычный пользователь / приложение.

→ RU-001 должен обеспечить fail-closed.

### Threat model B

Администратор EMS-001.

→ отдельно определить, какие механизмы могут быть им изменены и какие дополнительные меры необходимы.

Не выдавать A за B.

WFP использует системную модель безопасности Windows, а локальный администратор имеет возможности управления соответствующими объектами. ([Microsoft Learn][5])

---

# 10. DNS

DNS является частью RU-001 security boundary.

Требование:

> При отсутствии защищённого пути DNS не должен тихо откатываться к физическому интерфейсу и публичному resolver.

Проверить:

* DNS через VPN;
* DNS через физический interface;
* fallback DNS;
* NRPT;
* Windows DNS Client;
* Smart Multi-Homed Name Resolution;
* LLMNR;
* NetBIOS;
* DoH;
* application-embedded resolvers.

Эксперты отдельно указали DNS fallback и embedded DoH как потенциальные обходы.  

### Предпочтительная политика

Если приложение/система требует DNS:

```text
DNS
 ↓
protected path
```

Если protected path отсутствует:

```text
DNS
 ↓
BLOCK
```

а не:

```text
DNS
 ↓
physical interface
 ↓
ISP
```

---

# 11. IPv6

Это обязательное требование.

Текущее отсутствие глобального IPv6 **не считать основанием для игнорирования IPv6**. Baseline прямо фиксирует, что IPv6 stack включён, но сейчас не имеет global address/default route. 

Необходимо выбрать одну из двух веток:

### Preferred

IPv6 полностью проходит через защищённый путь.

### Alternative

IPv6 полностью блокируется, если защищённый IPv6 path не реализован.

Недопустимо:

```text
IPv4 → VPN
IPv6 → physical network
```

---

# 12. Local LAN

Не разрешать автоматически весь LAN traffic.

Сначала определить, нужен ли EMS-001 доступ к:

* router;
* NAS;
* printers;
* local services;
* other LAN nodes.

Если доступ нужен, создать **явный список исключений**.

Если не нужен:

> default policy = block.

Это особенно важно, поскольку Force Tunnel допускает exclusion routes, а значит локальные исключения должны быть сознательным архитектурным решением, а не побочным эффектом. ([Microsoft Learn][6])

---

# 13. Mobility

RU-001 является relocation profile.

Проверить минимум:

1. Home network
2. Office network
3. Public Wi-Fi
4. Mobile hotspot
5. Network with IPv6
6. Network with restrictive outbound UDP
7. Network where VPN endpoint is inaccessible

Ключевое правило:

> Недоступность VPN endpoint не должна превращаться в direct Internet access.

В этом случае допустимое состояние:

```text
VPN unavailable
      ↓
Internet unavailable
```

---

# 14. Boot / Startup

Это отдельный P0 test.

Проверить:

```text
Power ON
   ↓
Windows starts
   ↓
Network initializes
   ↓
VPN service/profile initializes
   ↓
Authentication
   ↓
Protected path
```

Особенно определить:

> может ли любое приложение или Windows service отправить Internet traffic до того, как RU-001 policy стала активной?

Эксперты независимо указали startup timing как потенциальный leakage window. 

---

# 15. Sleep / Wake

Проверить:

```text
VPN UP
 ↓
Sleep
 ↓
Wake
 ↓
Network reconnect
```

Во всех состояниях должно выполняться:

```text
Protected path UP → traffic allowed

Protected path not yet UP → traffic blocked
```

Не допускается временный direct Internet window.

---

# 16. Reboot

Проверить минимум:

### Test R1

VPN available during boot.

### Test R2

VPN endpoint unavailable during boot.

### Test R3

Authentication fails.

### Test R4

Network comes up later than VPN service.

### Test R5

Network changes during boot.

Expected result:

> отсутствие защищённого состояния = отсутствие Internet egress.

---

# 17. Tunnel failure

Обязательный тест:

```text
Traffic running
      ↓
VPN forcibly terminated
      ↓
observe
```

Expected:

```text
existing connection → terminates/stalls
new connections → BLOCKED
DNS → BLOCKED
direct HTTPS → BLOCKED
direct IPv6 → BLOCKED
```

Не принимать:

```text
VPN terminated
↓
Windows reconnects physical route
↓
traffic continues
```

---

# 18. Application independence

Проверить без per-app network configuration:

* Git;
* VS Code;
* PowerShell;
* CMD;
* SSH;
* HTTPS;
* ordinary Windows services.

Expected:

```text
Application
   ↓
Windows network stack
   ↓
RU-001
```

а не:

```text
Application
   ↓
special proxy/VPN configuration
```

Это требование подтверждено всеми тремя внешними экспертизами.  

---

# 19. Verification architecture

Не ограничиваться проверкой:

```text
VPN UI says Connected
```

Создать **три уровня проверки**.

### Level 1 — Control Plane

Проверить:

* VPN state;
* service state;
* routing;
* DNS configuration;
* firewall/WFP policy.

### Level 2 — Data Plane

Проверить фактические пакеты.

Использовать:

* packet capture;
* physical interface;
* virtual interface.

### Level 3 — External observation

Проверить:

* public egress IPv4;
* IPv6 egress;
* DNS visibility;
* external HTTPS connectivity.

WFP предоставляет штатные средства диагностики, включая `netsh wfp` capture/dump/show. ([Microsoft Learn][7])

---

# 20. Обязательная таблица состояний

Создать state matrix:

| State | VPN             | IPv4 Internet | IPv6 Internet | DNS | Expected                 |
| ----- | --------------- | ------------- | ------------- | --- | ------------------------ |
| S0    | OFF             | ?             | ?             | ?   | BLOCK                    |
| S1    | CONNECTING      | ?             | ?             | ?   | BLOCK                    |
| S2    | UP              | ?             | ?             | ?   | ALLOW via protected path |
| S3    | AUTH FAIL       | ?             | ?             | ?   | BLOCK                    |
| S4    | TUNNEL DROP     | ?             | ?             | ?   | BLOCK                    |
| S5    | RECONNECTING    | ?             | ?             | ?   | BLOCK                    |
| S6    | IPv6 appears    | UP            | UP            | ?   | Protected or BLOCK       |
| S7    | Sleep/Wake      | ?             | ?             | ?   | Protected or BLOCK       |
| S8    | Network changed | ?             | ?             | ?   | Protected or BLOCK       |

Qwen должен заполнить эту таблицу фактическими результатами экспериментов.

---

# 21. Acceptance Criteria

RU-001 считается технически принятым только если выполнены **все P0**:

### AC-01

При отсутствии VPN/защищённого пути прямой Internet egress невозможен.

### AC-02

При разрыве VPN прямой egress не возникает.

### AC-03

DNS не уходит напрямую через физический интерфейс.

### AC-04

IPv6 не создаёт обход.

### AC-05

После reboot политика сохраняется.

### AC-06

После sleep/wake политика сохраняется.

### AC-07

После смены сети политика сохраняется.

### AC-08

Git работает без индивидуальной VPN/proxy configuration.

### AC-09

VS Code работает без индивидуальной VPN/proxy configuration.

### AC-10

SSH работает без индивидуальной VPN/proxy configuration.

### AC-11

Фактический packet capture подтверждает отсутствие запрещённого egress.

### AC-12

Внешняя проверка подтверждает правильный egress IP при защищённом состоянии.

### AC-13

При failure state система переходит именно в **BLOCK**, а не в DEGRADED DIRECT INTERNET.

---

# 22. Что НЕ является acceptance criterion

Не считать доказательством безопасности:

* зелёный индикатор VPN;
* наличие VPN adapter;
* наличие `0.0.0.0/0` через VPN;
* отсутствие ошибки VPN-клиента;
* успешный ping;
* успешный DNS lookup;
* работа одного браузера.

Нужна проверка **фактического egress**.

---

# 23. Предпочтительный порядок инженерной работы

Не начинать с установки коммерческого VPN.

### Phase 1 — Architecture

Определить:

* Native Windows VPN;
* third-party system VPN;
* system-level Overlay.

### Phase 2 — Native Windows feasibility

Проверить:

* Force Tunnel;
* LockDown;
* Always On;
* DNS;
* IPv6;
* firewall/WFP;
* reboot.

Microsoft прямо документирует для Windows VPN Force Tunnel, Always On, traffic filters, NRPT и LockDown как параметры VPN-профиля. ([Microsoft Learn][8])

### Phase 3 — Minimal prototype

Только после Phase 1.

### Phase 4 — Adversarial testing

Проверить намеренные отказы.

### Phase 5 — Decision

Сформировать:

**RU-001-ADR-001**

с окончательно выбранной архитектурой.

### Phase 6 — Implementation specification

Только после ADR сформировать:

**RU-001-Implementation-Spec-001**

---

# 24. Что делать, если Native Windows VPN оказывается недостаточным

Не пытаться насильно подгонять требования.

Перейти:

```text
Native Windows VPN
       ↓
FAIL
       ↓
Third-party system VPN
       ↓
FAIL
       ↓
System-level Overlay
```

Но каждый переход должен сопровождаться **конкретным зарегистрированным reason for rejection**.

---

# 25. Что делать, если несколько решений проходят

Использовать следующие приоритеты:

1. **Fail-closed correctness**
2. **Security boundary**
3. **IPv4/IPv6 completeness**
4. **DNS containment**
5. **Boot/reboot reliability**
6. **Network mobility**
7. **Application independence**
8. **Diagnostic transparency**
9. **Low operational complexity**
10. **Future Compute Node compatibility**

Не выбирать решение только по:

* скорости;
* удобству GUI;
* популярности;
* стоимости;
* простоте установки.

---

# 26. Compute Node

Не проектировать RU-001 специально под Compute Node.

Но не закрывать путь к нему.

Предпочтительная будущая модель:

```text
                    EMS-001
                       │
          ┌────────────┴────────────┐
          │                         │
     RU-001 perimeter        Compute Network
          │                         │
   System VPN / WFP              Overlay
          │                         │
       Internet                Compute Node
```

Если в процессе исследования обнаружится, что один механизм естественно решает обе задачи без ухудшения RU-001 — представить это как **вариант оптимизации**, но не ослаблять P0 requirements ради унификации.

---

# 27. Deliverables

Qwen должен вернуть **не один общий ответ**, а следующий комплект.

### D1 — Architecture Decision Report

Сравнение:

* Native Windows VPN;
* third-party System VPN;
* System-level Overlay.

### D2 — Recommended Architecture

Одна выбранная архитектура.

### D3 — Rejected Alternatives

Для каждого:

* почему отвергнут;
* какое требование не выполнено.

### D4 — Threat / Failure Matrix

Минимум:

* boot;
* reboot;
* sleep/wake;
* tunnel drop;
* authentication failure;
* DNS failure;
* IPv6 appearance;
* network change;
* firewall failure;
* VPN service failure.

### D5 — Test Plan

Пошаговый план проверки на EMS-001.

### D6 — Acceptance Matrix

Все AC-01…AC-13 со статусом:

```text
PASS
FAIL
NOT TESTED
NOT APPLICABLE
```

### D7 — Evidence Package

Перечень артефактов:

* routing dumps;
* DNS configuration;
* firewall/WFP state;
* packet captures;
* VPN logs;
* external egress results.

### D8 — ADR

Финальное архитектурное решение:

**RU-001-ADR-001**

---

# 28. Строго запрещено на первом этапе

До утверждения ADR не следует:

* окончательно выбирать коммерческий VPN;
* менять сетевую архитектуру EMS-001 без фиксации baseline;
* отключать IPv6 без отдельной фиксации причины;
* удалять физический default route как самоцель;
* создавать постоянные firewall rules без rollback;
* изменять DNS без возможности восстановления;
* устанавливать Overlay в production state;
* считать эксперимент успешным только на основании GUI.

---

# 29. Правило безопасного эксперимента

Перед любым изменением:

```text
BASELINE
   ↓
SNAPSHOT
   ↓
CHANGE
   ↓
TEST
   ↓
VERIFY
   ↓
ROLLBACK or ACCEPT
```

Каждое изменение должно иметь:

* цель;
* ожидаемый эффект;
* способ проверки;
* способ rollback.

---

# 30. Вопросы, которые инженер обязан вернуть заказчику

Если Qwen обнаруживает неоднозначность, он **не должен самостоятельно придумывать пользовательское требование**.

Сначала задать вопрос.

### Q1 — Local LAN

Нужен ли EMS-001 доступ к другим узлам локальной сети?

**Моя предварительная ветка:** если нет — блокировать; если да — создать минимальный allow-list.

### Q2 — IPv6

Допустимо ли временно полностью блокировать IPv6 до появления полноценного IPv6-over-protected-path?

**Моя предпочтительная ветка:** **да, блокировать**, если полноценная защищённая IPv6-маршрутизация не доказана.

### Q3 — VPN endpoint availability

Допустимо ли состояние:

> VPN недоступен → Internet полностью недоступен?

**Ответ по текущему ТЗ:** **да. Это нормальное fail-closed состояние.**

### Q4 — Administrator threat model

Нужно ли защищаться от локального администратора?

**Предварительная ветка:** нет, если это не будет отдельно заявлено как requirement.

### Q5 — Compute Node

Нужно ли уже сейчас обеспечивать connectivity к Compute Node?

**Ответ:** нет.

Только не блокировать будущую интеграцию.

---

# 31. Особое требование к инженерному выводу

Не писать:

> «VPN работает, значит задача выполнена».

Писать:

> «AC-01…AC-13 проверены следующим образом; получены следующие наблюдения; следующие claims доказаны, следующие остаются предположениями».

Разделить:

### FACT

Наблюдалось непосредственно на EMS-001.

### DOCUMENTED

Подтверждено документацией Microsoft / разработчика.

### INFERENCE

Следует из архитектуры, но непосредственно не проверено.

### UNKNOWN

Требует дополнительного эксперимента.

---

# 32. Финальная формулировка задачи

**Спроектировать не VPN, а доказуемое состояние EMS-001, при котором:**

```text
             ┌─────────────────────┐
             │      APPLICATIONS   │
             │ Git / VS Code / SSH │
             └──────────┬──────────┘
                        │
                        ▼
              ┌──────────────────┐
              │      RU-001      │
              │                  │
              │ Routing          │
              │ Firewall / WFP   │
              │ DNS              │
              │ IPv4 / IPv6      │
              │ Protected Path   │
              └────────┬─────────┘
                       │
               ┌───────┴───────┐
               │               │
           PROTECTED         BLOCKED
             PATH          if unavailable
               │               │
               ▼               X
            Internet        Internet
```

**Главный инженерный критерий:**

> **При любом отказе защищённого пути система должна деградировать в BLOCKED, а не в DIRECT.**

И только после доказательства этого свойства архитектура считается пригодной для RU-001.

---
