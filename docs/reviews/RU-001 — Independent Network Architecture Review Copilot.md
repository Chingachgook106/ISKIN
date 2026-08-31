### Исполнительное заключение

**Предпочтительный класс архитектуры:** **System-level VPN** (force-tunnel, OS-integrated virtual adapter + WFP enforcement).  
**Второй по качеству:** **Overlay-network** (с виртуальным адаптером на уровне ОС).  
**Отклонён:** **Proxy-based** (недостаточно для требования fail-closed).

**Коротко почему:** System-level VPN позволяет установить чёткую границу безопасности на уровне сетевого стека ОС, обеспечить системное (не прикладное) управление маршрутизацией и DNS и реализовать поведение «fail-closed» для всех приложений без их перенастройки. Overlay может дать похожие свойства, но добавляет сложность и дополнительные векторы (control plane, NAT traversal). Proxy не даёт системной гарантии — многие приложения и системные службы обходят proxy и могут молча выйти в Интернет.

**Ключевые проверки на EMS-001 перед развёртыванием:** возможность установить виртуальный адаптер как единственный default route; применение и сохранение WFP/Windows Firewall правил, блокирующих egress вне туннеля; принудительная маршрутизация и/или шифрование DNS через туннель; поведение при появлении IPv6; поведение при перезагрузках, sleep/wake и смене сети.

---

### Текущее состояние (кратко, факты из baseline)

**Цитата из AttachedDocument:**  
- *“Active Interface: Ethernet 4; IP Address: 192.168.0.102; IPv4 Default Gateway: 192.168.0.1.”*  
- *“Active DNS Servers (Ethernet 4): 8.8.8.8, 1.1.1.1 (Public resolvers).”*

Вывод: EMS-001 — single-homed IPv4 клиент с plaintext DNS к публичным резолверам; IPv6 глобально отсутствует; нет установленного VPN/proxy/overlay.

---

### System-level VPN — анализ

**Описание:** виртуальный сетевой адаптер (TUN/TAP или WFP-based) + зашифрованный туннель к удалённому шлюзу; устанавливается 0.0.0.0/0 (и ::/0 при необходимости) через виртуальный адаптер; WFP/Windows Firewall блокирует egress вне туннеля.

**Граница безопасности:** на уровне OS network stack → virtual adapter → encrypted tunnel → remote gateway.

**Как реализовать FAIL-CLOSED (технически):**
- виртуальный адаптер должен быть единственным источником default route; физический default route удалён или имеет более низкий приоритет;
- WFP/Windows Firewall: глобальный deny для исходящего трафика, не привязанного к виртуальному адаптеру;
- DNS: резолверы доступны только через туннель (или DoH/DoT через туннель); DHCP DNS на физическом интерфейсе подавляется;
- не устанавливать default route через виртуальный адаптер до успешной аутентификации туннеля.

**Как верифицировать эмпирически:**
- снимки `Get-NetRoute`, `Get-DnsClientServerAddress`, дампы правил Windows Firewall/WFP;
- packet capture на physical interface: отсутствие UDP/53, TCP/443 к внешним хостам при падении туннеля;
- проверка публичного IP (api.ipify.org) — показывает egress IP туннеля при поднятом туннеле и отсутствие egress при падении (fail-closed).

**DNS:** при корректной конфигурации DNS-запросы исходят через virtual adapter к резолверам туннеля; риск обхода низкий при блокировке прямого доступа к публичным резолверам и подавлении DHCP DNS на физическом интерфейсе; Windows-specific векторы (LLMNR, NetBIOS, DoH) нужно отключить/контролировать.

**IPv4/IPv6:** IPv4 — полный контроль. IPv6 — требует явной обработки: установить ::/0 через туннель или блокировать физический ::/0; иначе IPv6 может обойти туннель.

**Приложения:** Git, VS Code, PowerShell, SSH, HTTPS/API клиенты и системные службы будут работать без per-app конфигурации, если маршрутизация и firewall настроены правильно.

**Основной риск:** неправильная конфигурация маршрутов или firewall (особенно отсутствие контроля IPv6) — silent fallback.

---

### Proxy-based — анализ

**Описание:** application-layer proxy (HTTP/HTTPS/SOCKS) или system proxy settings; локальный агент пересылает трафик к удалённому прокси.

**Граница безопасности:** на уровне приложения; нет единой системной границы.

**Почему proxy не удовлетворяет fail-closed:**
- многие приложения и системные службы не используют системный proxy (или используют отдельные настройки WinHTTP/WinINET);
- Windows name-resolution и raw sockets остаются на физическом интерфейсе;
- при недоступности proxy приложения часто пытаются прямое соединение — silent leak.

**DNS:** обычно разрешение имён выполняется локально; proxy не гарантирует, что DNS пойдёт через прокси; высокий риск утечек.

**IPv4/IPv6:** proxy не контролирует raw sockets; IPv6 легко обходит proxy.

**Приложения:** работает только для proxy-aware приложений; Git, SSH и многие CLI требуют ручной настройки.

**Операционная сложность:** низкая установка, но высокая сложность диагностики и высокий риск silent bypass.

---

### Overlay-network — анализ

**Описание:** виртуальная сеть L3/L2 (виртуальный адаптер) с инкапсуляцией; может быть mesh (peer-to-peer) или hub-and-spoke.

**Граница безопасности:** при наличии virtual adapter — на уровне OS network stack (как у VPN); если overlay user-mode и не стартует рано — гарантий меньше.

**FAIL-CLOSED:** достижимо при установке default routes через overlay и блокировке физического egress; риски — control-plane failure, NAT traversal (UDP hole punching) и delayed startup.

**DNS:** аналогично system VPN — можно направлять DNS через overlay, но нужно блокировать прямой доступ к резолверам.

**IPv6:** зависит от реализации overlay; нужно явное управление IPv6.

**Приложения:** при system-level overlay — большинство приложений защищены; при user-mode overlay — возможны утечки до старта.

**Основной риск:** контроль-плэйн и NAT traversal создают дополнительные пути обхода.

---

### Сравнительная матрица (кратко)

| Критерий | System VPN | Proxy | Overlay |
|---|---:|---:|---:|
| Full system coverage | **Strong** | **Weak** | **Adequate** |
| DNS protection | **Strong** | **Weak** | **Adequate** |
| IPv4 protection | **Strong** | **Weak** | **Adequate** |
| IPv6 protection | **Adequate** (if configured) | **Weak** | **Adequate** |
| Fail-closed | **Strong** | **Unsuitable** | **Adequate** |
| Routing control | **Strong** | **Weak** | **Adequate** |
| Application independence | **Strong** | **Weak** | **Adequate** |
| Windows integration | **Adequate** | **Adequate** | **Weak–Adequate** |
| Reboot behaviour | **Adequate** | **Weak** | **Weak–Adequate** |
| Network mobility | **Adequate** | **Weak** | **Adequate** |
| Failure recovery | **Adequate** | **Weak** | **Adequate** |
| Local LAN compatibility | **Adequate** | **Strong** | **Adequate** |
| Security boundary clarity | **Strong** | **Weak** | **Adequate** |
| Operational complexity | **Moderate** | **Low** | **Moderate–High** |
| Future Compute Node path | **Strong** | **Weak** | **Adequate** |
| Verification difficulty | **Moderate** | **Low** | **Moderate–High** |
| Main failure mode | misconfiguration / IPv6 leak | silent direct-Internet fallback | control-plane / NAT traversal leakage |

---

### Критический (adversarial) обзор

**Скрытые допущения и опасные кейсы:**
- допущение, что VPN/overlay стартует раньше приложений; на Windows это не гарантировано — нужно тестировать startup timing;
- допущение, что IPv6 отсутствует или контролируется; на многих сетях IPv6 включён по умолчанию;
- локальный администратор может изменить/удалить firewall/WFP правила;
- DoH/встроенные резолверы в приложениях могут резолвить вне туннеля;
- NAT traversal в overlay может создать прямые пути.

**Возможные обходы:** IPv6 bypass, misconfigured split-tunnel, приложения с embedded DoH, hole punching.

> **Сильнейший аргумент против preferred архитектуры:** System-level VPN даёт сильные гарантии только при безупречной конфигурации и хост-харденинге; малейшая ошибка (особенно в управлении IPv6 или в правилах WFP) может привести к тихому обходу и ложному ощущению безопасности.

---

### Рекомендация (подробно)

1. **Preferred:** System-level VPN (force-tunnel, virtual adapter + WFP).  
2. **Second:** Overlay (если требуется mesh/Compute Node сценарий и если control plane безопасен).  
3. **Rejected:** Proxy-only.  
4. **Почему:** System VPN даёт ясную, проверяемую границу на уровне ОС и обеспечивает application-independence; proxy не даёт системной гарантии; overlay может подойти, но требует больше контроля.  
5. **Когда не выбирать System VPN:** если node должен работать в сетях, где VPN-трафик блокируется и нет альтернативного транспорта; если локальные пользователи имеют права изменять firewall; если организация не готова провести тщательное тестирование (особенно IPv6).  
6. **Что проверить на EMS-001 перед развёртыванием:** возможность установки virtual adapter как единственного default route; устойчивость и сохранение WFP/Firewall правил; отсутствие DNS/IPv6 утечек; поведение при reboot/sleep/wake и смене сети; доступность protected endpoint из целевых сетей.

---

### План экспериментальной верификации (шаги для EMS-001)

1. Сохранить baseline: `Get-NetIPConfiguration`, `Get-NetRoute`, `Get-DnsClientServerAddress`, packet capture на physical interface.  
2. Установить тестовый VPN client, настроить virtual adapter без установки default route; проверить отсутствие egress через адаптер до аутентификации.  
3. Fail-closed boot test: настроить так, чтобы физический default route был удалён/заблокирован и VPN сервис стартовал, но аутентификация намеренно проваливалась; перезагрузить и убедиться, что внешние соединения блокируются и нет DNS/HTTP трафика на physical interface.  
4. Tunnel up test: поднять туннель, проверить `Get-NetRoute` (0.0.0.0/0 через virtual adapter), проверить публичный IP (api.ipify.org) и DNS резолвинг через туннель.  
5. Tunnel drop test: убить туннель; убедиться, что нет silent fallback (packet capture, firewall logs).  
6. IPv6 test: подключиться к IPv6-enabled сети; убедиться, что ::/0 не проходит по физическому интерфейсу и либо идёт через туннель, либо блокируется.  
7. Mobility tests: смена сетей (home/office/hotspot/public); проверка восстановления и поведения в сетях, блокирующих VPN.  
8. Application independence: запуск Git, VS Code, SSH и проверка, что трафик идёт через туннель без per-app настроек.  
9. Adversarial tests: попытки создать raw sockets, попытки изменить firewall как non-admin и admin; проверка устойчивости.

**Артефакты верификации:** дампы маршрутов, дампы правил firewall/WFP, packet captures на physical и virtual интерфейсах, логи VPN клиента, результаты публичного IP и DNS тестов.

---

### Открытые вопросы (требуют тестирования)

- Может ли выбранный VPN/overlay клиент гарантировать ранний старт и установку маршрутов до запуска пользовательских приложений на EMS-001?  
- Будут ли встроенные DoH реализации приложений резолвить вне туннеля в каких-то failure-mode?  
- Какие системные службы на EMS-001 выполняют сетевые операции до поднятия системных сервисов и могут ли они утечь?  
- Насколько control plane overlay безопасен и не создаёт ли он неожиданных обходов через NAT traversal?

---

### Источники

- Baseline: RU-001-Network-Baseline-001.md (приложен).  
- Microsoft documentation по Windows Filtering Platform, DNS client behavior, interface metrics (рекомендуется использовать при реализации и верификации).  
- RFC и технические материалы по tunnelling/overlay (для проектирования control plane и маршрутизации).

---