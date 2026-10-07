# network_logs

## Описание признаков

### 1. Базовые признаки соединения

<b>duration</b>
* float
* 0 – ~100
* Длительность соединения в секундах. 0 — короткие/мгновенные (SYN-flood, ICMP); большие значения — длительные сессии (SSH, FTP).

<b>protocol_type</b>
* категориальный
* tcp, udp, icmp
* Тип транспортного протокола. TCP — 80 % легитимного трафика, UDP — DNS/DHCP/SNMP, ICMP — ping/traceroute.

<b>service</b>
* категориальный
* http, ssh, ftp, dns, smtp, telnet, other, private, icmp
* Сетевой сервис (порт назначения, сгруппированный по типу).

<b>private</b>
* нестандартные порты, other — всё остальное.

<b>src_bytes</b>
* int
* 0 – ~50 000
* Количество байт, переданных от источника к получателю. Мало — сканирование/атаки; много — загрузки, эксплойты.

</b>dst_bytes</b>
* int
* 0 – ~50 000
* Количество байт, переданных от получателя к источнику. 0 при src_bytes > 0 — аномалия (односторонний трафик, характерен для smurf).


### 2. Признаки трафика (traffic features)

Описывают статистику соединений за окно 2 секунды к тому же хосту/сервису.

<b>count</b>
* int
* 0 – 512
* Число соединений к тому же хосту за 2 секунды. Высокое значение — DDoS, port scan, brute-force.

<b>srv_count</b>
* int
* 0 – 512
* Число соединений к тому же сервису за 2 секунды. Отличается от count тем, что группирует по сервису, а не по хосту.

<b>serror_rate</b>
* float
* 0.0 – 1.0
* Доля соединений с SYN-ошибками (SYN error). Высокое (>0.8) — признак SYN-flood (neptune).

<b>rerror_rate</b>
* float
* 0.0 – 1.0
* Доля соединений с REJ-ошибками (REJ error, отказ в соединении). Высокое — признак сканирования закрытых портов (portsweep, ipsweep).

<b>same_srv_rate</b>
* float
* 0.0 – 1.0
* Доля соединений к тому же сервису, что и текущее. Высокое — однотипный трафик (норма или smurf).

<b>diff_srv_rate</b>
* float
* 0.0 – 1.0
* Доля соединений к разным сервисам. Высокое — сканирование портов (portsweep, satan, ipsweep).

### 3. Признаки хоста назначения (host-based features)

Описывают статистику за 100 соединений к тому же хосту назначения.

<b>dst_host_count</b>
* int
* 0 – 255
* Число соединений к тому же хосту назначения за последние 100. Высокое — атака на конкретный хост (DDoS, brute-force).

<b>dst_host_srv_count</b>
* int
* 0 – 255
* Число соединений к тому же сервису на хосте назначения за 100. Низкое при высоком dst_host_count — сканирование разных портов.

### 4. Служебные столбцы

<b>label</b>
* int
* 0, 1
* Целевая переменная: 0 — нормальный трафик, 1 — атака. Используется для обучения с учителем.

<b>attack_type</b>
* категориальный
* normal, neptune, smurf, portsweep, ipsweep, guess_passwd, satan, back, teardrop, unknown_attack
* Метка типа трафика. Нужна только для анализа ошибок — <b>не должна попадать в признаки X</b>.

### 5. Признак -> атака

Какие признаки наиболее информативны для каждого типа атаки:

<b>neptune (SYN-flood)</b>

#### Ключевые признаки:
* count,
* srv_count,
* serror_rate,
* duration

#### Типичные значения:

* count 200–512,
* serror_rate 0.8–1.0,
* duration = 0

<b>smurf (ICMP-flood)</b>

#### Ключевые признаки:
* protocol_type,
* dst_bytes, count

#### Типичные значения:
* icmp,
* dst_bytes = 0,
* count 200–512

<b>portsweep (скан портов)</b>

#### Ключевые признаки:
* diff_srv_rate,
* rerror_rate,
* srv_count

#### Типичные значения:
* diff_srv_rate 0.8–1.0,
* rerror_rate 0.5–1.0

<b>ipsweep (скан хостов)</b>

#### Ключевые признаки:
* diff_srv_rate,
* count,
* service=private

#### Типичные значения:
* diff_srv_rate 0.5–1.0,
* service = private/other

<b>guess_passwd (brute-force)</b>

#### Ключевые признаки:
* service,
* count,
* duration

#### Типичные значения:
* ssh/ftp/telnet,
* count 1–10,
* низкий serror_rate

<b>satan (скан уязвимостей)</b>

#### Ключевые признаки:
* diff_srv_rate,
* same_srv_rate

#### Типичные значения:
* diff_srv_rate 0.7–1.0,
* same_srv_rate 0–0.3

<b>back (HTTP DoS)</b>

#### Ключевые признаки:
* service,
* src_bytes,
* dst_bytes

#### Типичные значения:
* http,
* src_bytes ≈ dst_bytes 1–10 КБ, count < 10

<b>teardrop (фрагментация)</b>

#### Ключевые признаки:
* protocol_type,
* dst_bytes, count

#### Типичные значения:
* udp,
* dst_bytes < 100, count < 10

<b>unknown_attack</b>

#### Ключевые признаки:
* смешанные

#### Типичные значения:
* ошибки разметки (label_flip)

### 6. Важность признаков (feature importance)

Оценка по вкладу в решение MLP (permutation importance на test):

1	serror_rate	(важность 0.21)	Главный сигнал для neptune, smurf

2	count	(важность 0.18)	Ловит flood-атаки

3	diff_srv_rate	(важность 0.14)	Ловит сканеры (portsweep, satan)

4	srv_count	(важность 0.11)	Дополняет count

5	dst_host_srv_count	(важность 0.09)	Хостовые атаки

6	same_srv_rate	(важность 0.07)	Отсекает ложные срабатывания

7	dst_host_count	(важность 0.06)	DDoS на конкретный хост

8	src_bytes	(важность 0.05)	Эксплойты, back

9	rerror_rate	(важность 0.04)	Сканирование

10	protocol_type	(важность 0.03)	icmp → smurf

11	service	(важность 0.02)	ssh → guess_passwd

12	duration	(важность <0.01)	Слабый сигнал на этом датасете

13	dst_bytes	(важность <0.01)	Слабый сигнал

<b>Практический вывод:</b> 80 % сигнала дают всего 5 признаков (serror_rate, count, diff_srv_rate, srv_count, dst_host_srv_count). Именно на них стоит смотреть в первую очередь при разборе FP/FN.
