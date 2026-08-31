# ТЗ RU-001-NB-001

## Снятие исходного сетевого baseline EMS-001

**Проект:** ISKIN
**Узел:** EMS-001 / «Малыш»
**Профиль:** RU-001 — Russian Federation Relocation Profile
**Тип работ:** диагностическая / измерительная
**Изменение конфигурации:** запрещено

---

### 1. Цель

Получить воспроизводимый снимок фактического сетевого состояния EMS-001 до разработки и установки сетевого security/privacy слоя RU-001.

Baseline используется как исходная точка для:

* выбора сетевой архитектуры;
* сравнения System VPN / Proxy / Overlay;
* проверки поведения после конфигурационных изменений;
* диагностики сетевых отказов;
* подтверждения отсутствия непредусмотренных маршрутов.

---

### 2. Архитектурное ограничение

На данном этапе **не устанавливать и не настраивать**:

* VPN;
* proxy;
* overlay network;
* kill-switch;
* firewall policy, если она не существует в текущей конфигурации;
* специальные DNS-механизмы;
* маршрутизационные изменения.

Не изменять существующую рабочую конфигурацию EMS-001.

---

### 3. Требуется зафиксировать

#### 3.1 Network interfaces

Зафиксировать:

* список физических и виртуальных интерфейсов;
* состояние каждого интерфейса;
* MAC-адреса;
* link state;
* текущие IPv4-адреса;
* текущие IPv6-адреса;
* prefix / subnet information.

Секретные либо избыточные идентификаторы, предназначенные только для внутреннего использования, в публичную документацию не включать.

---

#### 3.2 Default gateway

Определить:

* IPv4 default gateway;
* IPv6 default gateway, если присутствует;
* интерфейс, через который проходит default route;
* metric / priority маршрута, если применимо.

---

#### 3.3 DNS

Зафиксировать:

* DNS servers;
* порядок их использования;
* IPv4/IPv6 DNS endpoints;
* DNS search domains, если присутствуют;
* фактическую способность разрешать DNS-запросы.

---

#### 3.4 Routing

Снять routing table.

Отдельно отметить:

* default routes;
* локальные сети;
* IPv4 routes;
* IPv6 routes;
* metric;
* интерфейс / gateway каждого существенного маршрута.

---

#### 3.5 External connectivity

Проверить:

* наличие Internet connectivity;
* HTTPS connectivity;
* DNS resolution;
* доступность GitHub по HTTPS;
* доступность иных базовых внешних endpoints, необходимых для дальнейшего тестирования.

Не использовать внешний сервис как единственный источник истины.

---

#### 3.6 Public addressing

Определить текущий:

* public IPv4;
* public IPv6, если доступен.

Зафиксировать только как baseline.

Не предпринимать попыток скрыть или изменить адрес на данном этапе.

---

#### 3.7 IPv6

IPv6 исследовать отдельно.

Определить:

* включён ли IPv6;
* имеется ли global IPv6;
* имеется ли IPv6 default route;
* работает ли IPv6 Internet connectivity;
* используется ли IPv6 DNS.

IPv6 не считать автоматически «неактивным» только потому, что основная связь работает через IPv4.

---

### 4. Проверка утечек

На baseline-этапе зафиксировать исходное состояние:

* DNS exposure;
* IPv4 exposure;
* IPv6 exposure;
* возможное расхождение IPv4/IPv6 маршрутов.

Термины использовать строго:

**security ≠ privacy ≠ anonymization.**

Baseline не должен интерпретироваться как тест анонимности.

---

### 5. GitHub

Проверить:

* DNS resolution GitHub;
* TCP/HTTPS connectivity;
* фактическую доступность HTTPS endpoint.

**Git push/pull на данном этапе не выполнять**, если отдельное распоряжение архитектуры не дано.

---

### 6. Результат

Создать отчёт:

`RU-001-Network-Baseline-001.md`

Структура отчёта:

1. Date / time
2. EMS-001 identification
3. Interface state
4. IPv4 configuration
5. IPv6 configuration
6. Default gateway
7. DNS
8. Routing table
9. Public IPv4
10. Public IPv6
11. HTTPS connectivity
12. GitHub HTTPS connectivity
13. DNS test
14. IPv4/IPv6 observations
15. Potential leaks
16. Anomalies
17. Engineer conclusion

---

### 7. Критерий завершения

Baseline считается завершённым, если:

* сетевое состояние EMS-001 полностью зафиксировано;
* IPv4 и IPv6 рассмотрены независимо;
* routing table сохранена;
* DNS configuration зафиксирована;
* external connectivity проверена;
* GitHub HTTPS проверен;
* потенциальные IPv4/IPv6/DNS leaks отмечены;
* конфигурация EMS-001 при выполнении работ не изменена.

После получения отчёта инженер **не выполняет настройку VPN/proxy/overlay самостоятельно**.

Следующий этап — архитектурный выбор сетевого слоя RU-001.
