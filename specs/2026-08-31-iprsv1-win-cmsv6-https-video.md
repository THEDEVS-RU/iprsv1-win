id: 2026-08-31-iprsv1-win-cmsv6-https-video
product: IPRSV1
repo: THEDEVS-RU/iprsv1-win
branch: dev
depends_on: []
complexity: architectural
impl_model: opus
impl_effort: high

# Контекст

Задача IPRSV1-17: по `https://scanvision.online/` не идёт видео. Веб-интерфейс и
его websocket'ы после правок 31.08.2026 (спека
`specs/2026-08-31-iprsv1-win-nginx-websocket-443.md`) работают, видеопоток через
443 не идёт вообще — плеер обращается к отдельным портам медиа-контура напрямую.

Всё ниже измерено на машине 201.34.132.26 (`vdswin2k22`) 31.08.2026 по SSH,
только чтением: конфиги, netstat, HTTP-запросы к 127.0.0.1, SELECT'ы в
`1010GPS` и строковый разбор dll платформы. Ничего не менялось.

## Как H5-плеер строит адреса под HTTPS

`808gps/js/cmsv6player.min.js`, метод `ll(...)` (запрос адреса) и `pl(...)`
(разбор ответа `ttx_doServerAddr`):

1. Сначала плеер делает обычный **HTTPS-GET** на
   `https://<адрес>:<порт логин-сервера + 10000>/3/1?MediaType=1&Type=0&AVType=<n>&DevIDNO=<idno>&Channel=<n>`.
   Порт берётся из `loginServer.port` страницы и под `https:` прогоняется через
   `ve(port, map)`, то есть `+10000`, если карта не задана.
2. Из ответа берутся `server.clientPort`, список адресов
   (`clientIp`, `lanip`, `clientIp2`, `clientIp3`), флаг `IsHttps` и карта
   `HttpsMapPort`.
3. Только если `IsHttps` истинно **и** страница открыта по `https:`, поток идёт
   на `wss://<адрес>:<clientPort + 10000>`. Иначе плеер строит
   `ws://<адрес>:<clientPort>` — незашифрованный ws со страницы https, который
   браузер блокирует как mixed content.

`singleHttpsPort` (вариант «всё через один порт с портом в пути») в поставке
нигде не выставляется — это поле дефолтного конфига плеера, равное `false`, и
включается только правкой вендорского JS. В этой задаче не используется.

## Что отдаёт платформа сейчас

Замер 31.08.2026, запрос выполнен на самой машине по всем комбинациям
`MediaType` 0–3 × `Type` 0–2 с реальным устройством `900000400219`
(таблица `1010GPS.jt808_device_info`):

```
GET http://127.0.0.1:6605/3/1?MediaType=1&Type=0&AVType=1&DevIDNO=900000400219&Channel=0
{"IsHttps":0,"cmsserver":1,"result":0,
 "server":{"clientIp":"201.34.132.26","clientIp2":"201.34.132.26",
           "clientIp3":"201.34.132.26","clientOtherPort":"","clientPort":6604,
           "deviceIp":"201.34.132.26","deviceIp2":"201.34.132.26",
           "devicePort":6602,"lanip":"201.34.132.26","svrid":3}}
```

Отсюда три факта, каждый из которых по отдельности ломает видео под HTTPS:

- **`IsHttps: 0`.** Пока это ноль, плеер под https строит `ws://…:6604`, и
  браузер режет соединение как mixed content ещё до TCP. Одни только
  TLS-листенеры на `+10000` ничего не чинят — их просто некому будет вызвать.
  Это главное расхождение с исходной постановкой задачи.
- **`clientIp`/`lanip` — IP `201.34.132.26`, а не доменное имя.** Плеер
  (`wl(...)`) и страница (`getUserServerIp()` в `808gps/js/index.js`) берут
  `location.hostname` только если он совпал с одним из четырёх адресов;
  `scanvision.online` не совпадает, поэтому берётся `clientIp`, то есть IP. По
  IP браузер отвергнет `wss://` — сертификат GlobalSign выписан на имена
  (`scanvision.online`, `www.scanvision.online`, `autodiscover.`, `mail.`,
  `owa.`), IP в SAN нет.
- **`clientPort` — 6604 во всех комбинациях**, `clientOtherPort` пуст.

## Точный список портов вместо списка кандидатов

Список кандидатов из постановки (6604, 6617, 6630–6635) измерением не
подтверждается, а один нужный порт в нём отсутствует:

- **6605** — порт логин-сервера (`gpsloginsvr`, PID 4456 на момент замера).
  Именно он отвечает `result: 0` на `/3/1` и отдаёт адрес медиасервера. Под
  https плеер придёт на **16605** обычным HTTPS-GET. В постановке этого порта
  нет.
- **6604** — `clientPort` из ответа, `gpsmediasvr` (PID 8640). Под https плеер
  придёт на **16604** по `wss://`.
- 6617 отвечает на тот же `/3/1`, но `{"result":3,"message":"param error"}` и
  никем клиенту не выдаётся.
- 6630–6635 на HTTP-запрос рвут соединение (`GET /3/1` → connection closed),
  то есть по ним wss-апгрейд невозможен в принципе; это порты медиасервера не
  для браузера.

Живое видео, воспроизведение архива и голос идут через один и тот же
`clientPort` — проверено перебором `MediaType`/`Type` выше, ответ во всех
случаях одинаковый. Отдельный случай — интерком: в
`cmsv6player.min.js` метод `Sr(...)` строит `wss://<ip>:ve(<порт>)/ptt?devIdno=…`,
а после успешного ptt-логина метод `B4(...)` получает от сервера **ещё один**
порт и тоже пропускает его через `ve` (`+10000`). Значение этого порта
статически не определяется — оно приходит в рантайме, поэтому интерком
проверяется отдельно и, если DevTools покажет `wss://` на порт вне
{16604, 16605}, для него добавляется такой же листенер (см. «Что сделать», п. 5).

## Где живёт конфигурация платформы

Установлено чтением строк вендорских библиотек (`libmsgproc_clientmedia.dll`
плагина `plugin_msgproc_gpsloginsvr`, `libvehicleinfo_jt808.dll`) и
структуры БД:

- `IsHttps`, `HttpsMapPort` формирует `libmsgproc_clientmedia.dll` — та же
  библиотека, что печатает `CMsgClientMedia::ReadMediaSvrAddr` и ключи ответа
  (`server`, `clientIp`, `svrid`). Она читает системные параметры через
  `JT808VEHIINFO_GetSystemParamValue` и держит рядом строки
  `HttpsPfx`, `HttpsPort`, `Https`, `port`, `TtxWeb`, `PORT`.
- Системные параметры — таблица `1010GPS.jt808_system_params`
  (`data_type`, `data_code`, `data_value`). В ней уже есть две пустые строки:
  `id=61` (`HttpsMapHttpPort`/`HttpsMapHttpPort`) и `id=62` (`TtxWeb`/`PORT`).
  Строк с `data_type='Https'` нет.
- `HttpsMapPort` в JSON берётся из `JT808VEHIINFO_GetHttpsMapPort`
  (`libvehicleinfo_jt808.dll`), и рядом с этой функцией в библиотеке лежат
  строки таблицы `jt808_system_params` и ключ `HttpsMapHttpPort` — то есть
  карта портов задаётся именно этой строкой БД.
- Там же, в `libvehicleinfo_jt808.dll`, есть блок разбора **`nginx/conf/nginx.conf`**
  (строки `listen`, `ssl`, `ssl_certificate`, `ssl_certificate_key`,
  `proxy_pass`) вместе с `conf\server.xml` и ключами `WebLanIP`, `Https`,
  `Version` из `GPSLoginSvr.ini`. Путь относительный, то есть речь о
  вендорском nginx платформы: `C:\Program Files\CMSServerV6\7.37.2_20260710\nginx`.
  В его каталоге `conf` **нет** файла `nginx.conf` (есть только `nginx-org.conf`,
  `nginx-win.conf` и прочие образцы), сам вендорский nginx не запущен. Это
  объясняет `IsHttps: 0`: платформа не видит ни одного своего https-фронта.
- Адрес, который платформа отдаёт клиентам, переопределяется таблицей
  `1010GPS.server_info` (колонки `LanIP`, `IPClient`, `IPClient2`, `IPClient3`,
  `PortClient`, `IPDevice`, `PortDevice`, `Type`, `Area`, `IDNO`) — это то, что
  правит страница Server Management вебморды. Сейчас таблица **пуста**, и
  логин-сервер отдаёт автоопределённые значения; `server_session` тоже пуста.

Отсюда следует, что причина «нет видео по https» установлена полностью, а вот
конкретная ручка для `IsHttps` — гипотеза двух видов (строка системного
параметра либо автоопределение по вендорскому `nginx.conf`), и работа
начинается с её проверки, а не с правки nginx.

# Что сделать

Работы три: конфигурация платформы CMSV6 (без неё остальное бесполезно),
TLS-листенеры в nginx и правило файрвола. Порядок именно такой.

Доступ к машине — SSH, `Administrator@201.34.132.26`, ключ из k8s-секрета
`iprsv1-win-ssh` (namespace `ai-runners`), из пода просто `ssh iprsv1`. Для
всего сложнее одной команды использовать `powershell -EncodedCommand` с
base64/UTF-16LE, как описано в `CLAUDE.md` (пайп в `powershell -Command -`
молча обрывается на первом многострочном блоке).

Единственный изменяемый файл вне БД и файрвола — `C:\nginx\conf\nginx.conf`.
Плюс обновление `CLAUDE.md` в этом репозитории.

## 1. Проверочный инструмент, которым меряется каждый шаг

Все проверки платформы делаются одним и тем же запросом с самой машины:

```
GET http://127.0.0.1:6605/3/1?MediaType=1&Type=0&AVType=1&DevIDNO=900000400219&Channel=0
```

Смотрятся три поля ответа: `IsHttps`, `server.clientIp`/`server.lanip`,
`HttpsMapPort`. Базовое состояние до работ приведено в «Контексте» — его надо
снять ещё раз перед первой правкой и записать в отчёт как точку отсчёта.

## 2. Заставить платформу отдавать `IsHttps: 1`

Гипотезы проверяются по очереди, каждая — запросом из п. 1 сразу после
изменения. Все изменения обратимы, и каждое, не давшее эффекта, откатывается
до перехода к следующему:

1. Строка системного параметра: в `1010GPS.jt808_system_params` завести
   `data_type='Https'`, `data_code='HttpsPort'`, `data_value='443'`
   (`parent_id=0`, `sort_no` и `status` — как у соседних строк, например у
   `id=61`). Перед вставкой сделать дамп таблицы в файл рядом с рабочим
   каталогом, чтобы откат был механическим.
2. Автоопределение по вендорскому nginx: создать
   `C:\Program Files\CMSServerV6\7.37.2_20260710\nginx\conf\nginx.conf` с одним
   `server`-блоком `listen 443 ssl` с `ssl_certificate`,
   `ssl_certificate_key` и `proxy_pass http://127.0.0.1:80`. Вендорский nginx
   не запускать — файл нужен платформе только как источник данных. Если это
   сработало, файл остаётся; в `CLAUDE.md` он описывается как конфиг, который
   никто не исполняет.

Если ни одна из гипотез не даёт `IsHttps: 1` без перезапуска, значит значение
читается один раз при старте службы. Перезапуск `gpsloginsvr` **не делать
самостоятельно**: это работающая платформа, и постановка прямо запрещает
трогать службы CMSV6. Написать Максиму, что нужен перезапуск именно
`gpsloginsvr` (не `gpstomcat6` с его ~3.5 минутами простоя), и ждать ответа.

Если после перезапуска `IsHttps` всё равно `0` — остановиться и отчитаться
фактами: обе гипотезы отработаны, ручка не найдена, следующий шаг — запрос
вендору. Не подбирать значения наугад и не править вендорские dll и JS.

## 3. Отдавать клиентам доменное имя вместо IP

Пока `clientIp`/`lanip` — это `201.34.132.26`, `wss://` будет отвергнут по
сертификату. Нужно, чтобы запрос из п. 1 возвращал `scanvision.online` в
`clientIp` (и, соответственно, в `lanip`).

Штатный путь — страница Server Management в вебморде CMSV6
(`https://scanvision.online/`, вход под администратором), она пишет в
`1010GPS.server_info`. Прямой INSERT в таблицу — запасной вариант, и он опасен:
одна и та же строка задаёт не только клиентские адреса, но и
`IPDevice`/`PortDevice`, по которым регистраторы находят платформу. Поэтому,
каким бы путём строка ни заводилась, она обязана воспроизвести уже отдаваемые
значения один в один: `PortClient=6604`, `IPDevice=201.34.132.26`,
`PortDevice=6602`, `svrid`/идентификатор сервера — 3, и только `IPClient` и
`LanIP` становятся `scanvision.online`. `IPDevice` доменом **не** заменять:
регистраторы ходят на IP и DNS у них может не быть.

После изменения — запрос из п. 1: в ответе должен появиться домен, `devicePort`
и `deviceIp` обязаны остаться прежними. Затем проверить, что устройство
`900000400219` не отвалилось (страница мониторинга вебморды, статус онлайн).

## 4. Зафиксировать карту портов

В `1010GPS.jt808_system_params` в существующей строке `id=61`
(`HttpsMapHttpPort`) задать `data_value = '6604:16604;6605:16605'`. Формат —
`src:dst;src:dst`, ровно то, что разбирает `ve(t, i)` в плеере. Значения
совпадают с дефолтом `+10000`, поэтому поведение не меняется, но карта
становится явной: дальше видно, какие порты обслуживает nginx, и она нужна,
если для интеркома придётся добавить порт с другим смещением.

Проверка: в ответе из п. 1 появляется поле `HttpsMapPort` с этой строкой.

## 5. TLS-листенеры в `C:\nginx\conf\nginx.conf`

Перед правкой скопировать живой конфиг в `C:\nginx\conf\nginx.conf.bak-20260831b`
(суффикс `b` — потому что `nginx.conf.bak-20260831` уже занят прошлой задачей).
Правки аддитивные: существующие два `server`-блока на 443, `map $http_upgrade
$connection_upgrade`, `client_max_body_size 512m` не трогать.

Добавить два `server`-блока, по одному на порт:

- `listen 16605 ssl;` → `proxy_pass http://127.0.0.1:6605;`
- `listen 16604 ssl;` → `proxy_pass http://127.0.0.1:6604;`

В каждом:

- `server_name scanvision.online www.scanvision.online;`
- сертификат тот же, что у 443-блока `scanvision.online`:
  `ssl_certificate C:/nginx/conf/scanvision/scanvision-chain.crt;`
  `ssl_certificate_key C:/nginx/conf/scanvision/scanvision.key;`
- в `location /`: `proxy_http_version 1.1;`,
  `proxy_set_header Upgrade $http_upgrade;`,
  `proxy_set_header Connection $connection_upgrade;`,
  `proxy_buffering off;`, `proxy_read_timeout 3600s;`,
  `proxy_send_timeout 3600s;`, а также `Host`, `X-Real-IP`, `X-Forwarded-For`,
  `X-Forwarded-Proto` — тем же набором, что в существующих блоках.

Директиву `map $http_upgrade $connection_upgrade` повторно не объявлять — она
уже есть на уровне `http` и второе объявление это ошибка конфига. Свои
заголовки `Access-Control-Allow-*` не добавлять: логин-сервер уже отдаёт
`Access-Control-Allow-Origin: *` и `Access-Control-Allow-Methods: *`
(проверено 31.08.2026), дубль сломает CORS в браузере.

`ssl_stapling` в новых блоках не включать: он настроен только в 443-блоке и не
нужен для служебных портов.

Применение — по процедуре из `CLAUDE.md`, `nginx -s reload` из SSH не работает:

1. `nginx.exe -p C:\nginx -c conf\nginx.conf -t` — ожидается «syntax is ok» и
   «test is successful»; тест не прошёл — вернуть бэкап и не перезапускать;
2. `taskkill /F /IM nginx.exe`;
3. `schtasks /run /tn nginx_run`.

После перезапуска: `Get-Process nginx` должен показать ровно два процесса, оба
со `StartTime` момента рестарта (состояние до правки — PID 2988 и 4148, оба
стартовали 31.08.2026 17:49:48), и `netstat -ano | findstr LISTENING` —
443/16604/16605 только под новыми PID. Остался процесс со старым `StartTime` —
это отвязавшийся мастер из инцидента 04.08.2026: снять его отдельно и
перезапустить задачу заново.

Если проверка интеркома (см. «Критерии готовности») покажет в DevTools
`wss://` на порт вне {16604, 16605} — добавить для него такой же блок
`listen <порт> ssl` → `proxy_pass http://127.0.0.1:<порт минус 10000>;`,
дописать порт в правило файрвола из п. 6 и в карту из п. 4.

## 6. Файрвол

Одно новое правило на новые порты, в стиле уже существующих
`iprsv1-13-ports-tcp`: имя `iprsv1-17-ports-tcp`, направление Inbound, action
Allow, protocol TCP, `LocalPort 16604,16605`, `-Profile Any -RemoteAddress Any
-Program Any`. Вендорские правила `GPS*`, `GPSNginx`, `nginx 80`, `nginx 443`,
`iprsv1-13-ports-*` не трогать и не переписывать.

Порты 16604 и 16605 попадают в динамический диапазон машины (9000–64999,
`netsh int ipv4 show dynamicport tcp`), как и открытые в IPRSV1-13. Риск тот
же самый и уже принятый: если после перезагрузки в этом диапазоне поднимется
чужой слушатель, порт окажется открыт снаружи. Перед созданием правила снять
снимок слушателей и убедиться, что 16604/16605 свободны; факт записать в отчёт.

## 7. Документация

Обновить `CLAUDE.md` в корне репозитория:

- в раздел «Web entry points (80 / 443)» добавить описание медиа-контура под
  HTTPS: какие порты слушает nginx, куда проксирует, что 6605 — логин-сервер, а
  6604 — медиасервер;
- завести раздел про конфигурацию CMSV6, отдаваемую клиентам: запрос-инструмент
  из п. 1, поля ответа, `server_info`, строки `jt808_system_params`
  (`HttpsMapHttpPort`, `TtxWeb`/`PORT`, заведённые в п. 2), и найденный факт,
  что платформа определяет наличие https разбором
  `<каталог CMSV6>\nginx\conf\nginx.conf`;
- в разделе «Firewall» добавить `iprsv1-17-ports-tcp` с его портами;
- в списке бэкапов рядом с живым конфигом добавить `nginx.conf.bak-20260831b`;
- зафиксировать, какая из гипотез п. 2 сработала, а какая нет — это ровно то
  знание, которого сейчас в репозитории нет.

Пароли и токены (`LOGIN`, `PASSWORD`, `SERVER`, токен frps, пароль дашборда,
пароль БД CMSV6) в репозиторий, коммиты и логи команд не писать — только имена.

# Что уже есть

- `C:\nginx\conf\nginx.conf` (2187 байт, 31.08.2026 17:49) — рабочий, два
  `server`-блока `listen 443 ssl` (`test.thedevs.ru` на сертификате Let's
  Encrypt и `scanvision.online www.scanvision.online` на GlobalSign с
  `ssl_stapling`), у обоих `proxy_pass http://127.0.0.1:80`, полный набор
  ws-директив и `client_max_body_size 512m` на уровне `http`, `map
  $http_upgrade $connection_upgrade` объявлен один раз. Это база, к которой
  добавляются новые блоки.
- Сертификат GlobalSign: `C:\nginx\conf\scanvision\scanvision-chain.crt`,
  `scanvision.key`, `scanvision-trusted.crt`. Действует до 18.02.2027,
  автопродления нет.
- Бэкапы рядом с живым конфигом: `nginx.conf.bak`, `nginx.conf.bak2`,
  `nginx.conf.bak-20260814`, `nginx.conf.bak-20260831` — принятая на машине
  схема именования.
- `C:\nginx\run_nginx.bat` и задача планировщика `nginx_run` (SYSTEM, при
  старте) — единственный способ поднять nginx; стартер идемпотентен, второго
  мастера не поднимает, второй задачи создавать нельзя.
- Правила файрвола `iprsv1-13-ports-tcp` / `iprsv1-13-ports-udp` (созданы
  31.08.2026) — образец именования и параметров для нового правила.
- Слушатели медиа-контура (31.08.2026): 6604, 6617, 6630–6635 — `gpsmediasvr`
  (PID 8640); 6605, 6606 — `gpsloginsvr` (PID 4456); 6602 — `gpsmediasvr`;
  6607, 6608, 9999 — `gpsgatewaysvr`; 6603 — `gpsusersvr`; 6609, 6610 —
  `gpsdownsvr`; 6611, 6612 — `gpsstoragesvr`.
- Доступ к БД платформы: MySQL 5.5 на `127.0.0.1:3311`, база `1010GPS`,
  реквизиты в
  `C:\Program Files\CMSServerV6\7.37.2_20260710\tomcat\webapps\gpsweb\WEB-INF\classes\config\jdbc.properties`
  и в `C:\Program Files\CMSServerV6\data\database.ini`; клиент —
  `C:\Program Files\CMSServerV6\mysql\5.5_x64\bin\mysql.exe`. Своих учёток не
  заводить.
- Устройство `900000400219` (`1010GPS.jt808_device_info`) — на нём снят
  эталонный ответ `/3/1`, им же проверять изменения.
- В `1010GPS.jt808_system_params` уже есть пустые строки `id=61`
  (`HttpsMapHttpPort`) и `id=62` (`TtxWeb`/`PORT`) — новые строки для них
  создавать не нужно, значение задаётся в существующих.
- Спека `specs/2026-08-31-iprsv1-win-nginx-websocket-443.md` — ws-директивы 443
  уже добавлены ей, дублировать не нужно; её раздел про перезапуск nginx и
  осиротевшего мастера актуален и здесь.

# Критерии готовности

Платформа:

- запрос `http://127.0.0.1:6605/3/1?MediaType=1&Type=0&AVType=1&DevIDNO=900000400219&Channel=0`
  возвращает `"IsHttps":1`, `server.clientIp` и `server.lanip` равны
  `scanvision.online`, `server.clientPort` остался `6604`, `server.deviceIp` и
  `server.devicePort` не изменились (`201.34.132.26` и `6602`);
- в том же ответе `HttpsMapPort` равен `6604:16604;6605:16605`;
- устройство `900000400219` остаётся онлайн в вебморде после всех правок.

Транспорт:

- `openssl s_client -connect scanvision.online:16604` и `…:16605` завершают
  handshake и отдают цепочку GlobalSign (issuer `GlobalSign GCC R46 DV TLS CA
  2025`);
- ws-апгрейд снаружи на `wss://scanvision.online:16604/` (заголовки
  `Connection: Upgrade`, `Upgrade: websocket`, `Sec-WebSocket-Version: 13`,
  `Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==`) возвращает **101**;
- `https://scanvision.online:16605/3/1?MediaType=1&Type=0&AVType=1&DevIDNO=900000400219&Channel=0`
  снаружи отдаёт тот же JSON, что и запрос к 127.0.0.1:6605, с заголовком
  `Access-Control-Allow-Origin: *` в единственном экземпляре;
- `nginx.exe -p C:\nginx -c conf\nginx.conf -t` — «syntax is ok» и «test is
  successful»; после рестарта живых процессов nginx ровно два, оба со свежим
  `StartTime`, 443/16604/16605 слушают только они.

Браузер (проверка глазами, по `https://scanvision.online/`):

- живое видео открывается и не рвётся дольше минуты;
- воспроизведение архива открывается и играет;
- интерком (ptt) устанавливает связь; в DevTools зафиксировать, на какие порты
  ушли `wss://` — если появился порт вне {16604, 16605}, он добавляется по
  процедуре из «Что сделать», п. 5, и задача не считается выполненной, пока
  интерком не заработает;
- `http://scanvision.online/` продолжает работать как раньше: страница 200,
  живое видео идёт, ws-пути `/ws/webSocket/index/1` и `/ws/webSocket/down/1`
  отвечают 101 и по http, и по https.

Репозиторий:

- `CLAUDE.md` обновлён по п. 7, все даты замеров проставлены, записано, какая
  гипотеза по `IsHttps` сработала.

# Не трогать

- Службы CMSV6 (`gpstomcat6`, `GPSDaemon`, `GPSGatewaySvr`, `GPSMediaSvr`,
  `GPSLoginSvr`, `GPSUserSvr`, `GPSDownSvr`, `GPSStorageSvr`,
  `GPSDataProcSvr`, `GPSGeocodeSvr`, `GPSFtpd`, `GPSMysqld`) — не
  останавливать и не перезапускать. Перезапуск `gpsloginsvr` возможен только
  после явного согласия Максима (см. «Что сделать», п. 2); `gpstomcat6` в этой
  задаче не перезапускается вообще (~3.5 минуты простоя платформы).
- Вендорские файлы платформы: JS и HTML в `tomcat\webapps\gpsweb`
  (в том числе `cmsv6player.min.js` и `ttxplayer-h5.js`), dll и exe в `bin`,
  `tomcat\conf\server.xml`. Коннектор на порту 80 не менять и `address="127.0.0.1"`
  в него не добавлять ни при каких условиях — десктопный клиент ходит на этот
  порт напрямую.
- Существующие `server`-блоки на 443, их сертификаты, `ssl_stapling`,
  `resolver`, `client_max_body_size`, `map $http_upgrade $connection_upgrade`,
  `worker_processes`, `worker_connections`, `keepalive_timeout`. Порт 80,
  `listen 80`, редирект на 443 и HSTS не добавлять.
- Правила файрвола `GPS*`, `GPSNginx`, `nginx 80`, `nginx 443`,
  `iprsv1-13-ports-tcp`, `iprsv1-13-ports-udp`. Динамический диапазон портов
  (9000–64999) не менять.
- `frps`, его конфиг и задачи планировщика, задача win-acme и сертификат
  `test.thedevs.ru`, задача `nginx_run` и `run_nginx.bat`.
- В `1010GPS` — только строки, названные в «Что сделать»: `server_info` и две
  строки `jt808_system_params`. Схему не менять, другие таблицы не править,
  учётки MySQL не заводить.
- Версию nginx не поднимать, бинарь не заменять, вендорский nginx платформы не
  запускать. Другие файлы в `specs/` не править.
