id: 2026-08-31-iprsv1-win-nginx-websocket-443
product: IPRSV1
repo: THEDEVS-RU/iprsv1-win
branch: dev
depends_on: []
complexity: simple
impl_model: sonnet
impl_effort: high

# Контекст

Задача IPRSV1-15: веб-интерфейс CMSV6 по `https://scanvision.online/` теряет
websocket-соединения, по `http://` те же соединения работают. Из-за этого
главный интерфейс платформы под HTTPS не обновляет данные в реальном времени.

Замер 31.08.2026 13:00 UTC снаружи, один и тот же запрос с заголовками
`Connection: Upgrade`, `Upgrade: websocket`, `Sec-WebSocket-Version: 13`,
`Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==`:

| Запрос | Ответ |
|---|---|
| `https://scanvision.online/ws/webSocket/index/1` | **404** (`Server: nginx/1.30.4`) |
| `http://scanvision.online/ws/webSocket/index/1` | **101**, `Sec-WebSocket-Accept: s3pPLMBiTxaQ9kYGzzhZRbK+xOo=` |
| `https://scanvision.online/ws/webSocket/down/1` | **404** |
| `http://scanvision.online/ws/webSocket/down/1` | **101** |
| `https://test.thedevs.ru/ws/webSocket/index/1` | **404** |

То есть дефект не в приложении: тот же самый tomcat на 127.0.0.1:80 отдаёт
корректный handshake, когда до него доходит запрос с заголовком `Upgrade`.
Затронуты оба server-блока nginx, а не только `scanvision.online`.

Причина установлена прямым чтением конфига на машине 31.08.2026.
`C:\nginx\conf\nginx.conf` — 45 строк, два блока `server` с `listen 443 ssl`,
оба с `proxy_pass http://127.0.0.1:80` и четырьмя заголовками
(`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`). В обоих
`location /` нет `proxy_http_version 1.1` и нет проброса `Upgrade`/`Connection`,
поэтому nginx отправляет апгрейд-запрос на tomcat как обычный HTTP/1.0-вызов
без `Upgrade`; фильтр websocket в tomcat такой запрос не перехватывает, обычного
сервлета на этом пути нет — и ответом становится 404.

**Уточнение исходной постановки: это добавление, а не восстановление.** В
задаче и в `CLAUDE.md` сказано, что эти директивы были в версии конфига до
14.08.2026 и потерялись при перестройке. На машине это не подтверждается: поиск
`findstr /I` по `upgrade`, `http_version`, `proxy_buffering`,
`client_max_body_size` в `nginx.conf.bak`, `nginx.conf.bak2` и
`nginx.conf.bak-20260814` не даёт ни одного совпадения, при этом контрольный
поиск `proxy_pass` находит строки во всех трёх файлах — то есть файлы читаются,
директив в них действительно нет. Копировать конфиг из бэкапа нельзя, ни один из
них проблему не решает; строку `CLAUDE.md` про «более раннюю ревизию с
`map $http_upgrade`» надо исправить как ложную.

Второй дефект того же корня — размер тела запроса. В `http {}` нет
`client_max_body_size`, действует дефолт nginx 1 МБ, и через 443 загрузка файла
крупнее мегабайта отбивается самим nginx. Замер 31.08.2026 12:59 UTC: POST 2 000 000
байт на `https://scanvision.online/` → **413** при `size_upload=0` (nginx рвёт
запрос, не читая тело), тот же POST на `http://scanvision.online/` → **200**,
тело принято целиком. Это не теоретическая проблема: в `C:\nginx\logs\error.log`
уже есть записи `client intended to send too large body` от 18.08.2026 — два
POST по 10 МБ на `server: test.thedevs.ru`.

Потолок самого приложения — 500 МБ: в
`C:\Program Files\CMSServerV6\7.37.2_20260710\tomcat\webapps\gpsweb\WEB-INF\web.xml`
стоит `<max-file-size>524288000</max-file-size>`, в
`WEB-INF\classes\struts.properties` — `struts.multipart.maxFileSize=524288000`,
а у коннектора на порту 80 `maxPostSize="-1"`. Поэтому лимит nginx выбирается
равным `512m` — чуть выше потолка приложения, чтобы гейтом оставалось
приложение, а не прокси.

# Что сделать

Единственный изменяемый файл на машине — `C:\nginx\conf\nginx.conf` (доступ по
SSH, `Administrator@201.34.132.26`, ключ из k8s-секрета `iprsv1-win-ssh`
namespace `ai-runners`). Плюс обновление `CLAUDE.md` в этом репозитории.

1. **Бэкап живого конфига** перед правкой: скопировать
   `C:\nginx\conf\nginx.conf` в `C:\nginx\conf\nginx.conf.bak-20260831`. Рядом
   уже лежат `.bak`, `.bak2`, `.bak-20260814` — это принятая на машине схема.

2. **На уровне `http {}`, до обоих `server`-блоков**, добавить map:

   ```
   map $http_upgrade $connection_upgrade {
       default upgrade;
       ''      close;
   }
   ```

3. **На уровне `http {}`** добавить `client_max_body_size 512m;` — значение
   именно такое, обоснование в «Контексте».

4. **В обоих `server`-блоках** (`test.thedevs.ru` и
   `scanvision.online www.scanvision.online`) в существующий `location /`
   добавить шесть директив рядом с уже имеющимися `proxy_set_header`:

   ```
   proxy_http_version 1.1;
   proxy_set_header   Upgrade $http_upgrade;
   proxy_set_header   Connection $connection_upgrade;
   proxy_buffering    off;
   proxy_read_timeout 3600s;
   proxy_send_timeout 3600s;
   ```

   Директивы ставятся именно в общий `location /`, а не в отдельный
   `location /ws/`: у CMSV6 два известных ws-пути (`/ws/webSocket/index/1`,
   `/ws/webSocket/down/1`), но список путей взят из логов и не гарантированно
   полон — ws-путь вне `/ws/` при отдельной локации молча получил бы дефолтный
   60-секундный таймаут и рвался бы по простою. Трафик машины мал (около 7 тысяч
   запросов за пять суток по анализу `access.log` в `CLAUDE.md`), поэтому
   глобальные `proxy_buffering off` и длинные таймауты здесь дешевле, чем вторая
   локация с продублированным набором заголовков.

   Существующие `proxy_pass http://127.0.0.1:80;` и четыре заголовка
   (`Host`, `X-Real-IP`, `X-Forwarded-For`, `X-Forwarded-Proto`) остаются как
   есть, ничего из них не переписывать.

5. **Проверка синтаксиса** — из каталога `C:\nginx` и с явными путями, иначе
   nginx не найдёт относительный `logs/error.log`:
   `nginx.exe -p C:\nginx -c conf\nginx.conf -t`. Ожидается «syntax is ok» и
   «test is successful». Тест не прошёл — конфиг не применять, вернуть бэкап.

6. **Применение** — `nginx.exe -s reload` из SSH не работает (мастер запущен
   планировщиком от SYSTEM, reload падает с `OpenEvent(...) failed
   (5: Access is denied)`). Порядок: `taskkill /F /IM nginx.exe`, затем
   `schtasks /run /tn nginx_run`.

7. **Проверка на осиротевший мастер** после рестарта: `Get-Process nginx`
   должен показать ровно два процесса, оба со `StartTime` момента рестарта, и
   `netstat -ano | findstr :443` — `LISTENING` только под этими PID. Базовое
   состояние до правки (31.08.2026 13:00 UTC): PID 5804 и 8064, оба стартовали
   14.08.2026 20:49, слушает 443 только 8064. Если после рестарта в списке
   остаётся процесс со старым `StartTime` — это тот самый отвязавшийся мастер из
   инцидента 04.08.2026, его надо снять отдельно и перезапустить задачу заново.

8. **Обновить `CLAUDE.md`** в корне репозитория:
   - секцию «🔴 Known issue: websockets do not survive the 443 proxy» заменить
     описанием исправленного состояния с датами замеров до и после;
   - убрать утверждение, что более ранняя ревизия конфига содержала
     `map $http_upgrade $connection_upgrade`, `proxy_buffering off` и таймауты
     3600 s — на машине оно не подтверждается (см. «Контекст»);
   - в разделе «### nginx» записать новый набор директив, значение
     `client_max_body_size 512m` и его обоснование (потолок приложения 500 МБ);
   - добавить `nginx.conf.bak-20260831` в перечень бэкапов рядом с живым файлом.

# Что уже есть

- `C:\nginx\conf\nginx.conf` — оба `server`-блока уже настроены и рабочие:
  сертификаты (`le/test.thedevs.ru-*.pem` и `scanvision/scanvision-*`),
  `ssl_stapling` с `resolver 8.8.8.8 8.8.4.4` в блоке scanvision,
  `proxy_pass http://127.0.0.1:80` и четыре проксирующих заголовка. Задача —
  дополнить этот конфиг, а не переписать.
- `nginx.conf.bak`, `nginx.conf.bak2`, `nginx.conf.bak-20260814` — бэкапы рядом
  с живым файлом. Ни в одном нет ws-директив (проверено), источником для
  копирования они не являются.
- `C:\nginx\run_nginx.bat` — идемпотентный стартер: проверяет `tasklist` на
  живой `nginx.exe` и только тогда `cd /d C:\nginx` + `start "" nginx.exe`.
  Второго мастера он поднять не может, свой стартер писать не нужно.
- Задача планировщика `nginx_run` (от SYSTEM, при старте машины) — единственная
  задача nginx на машине. Дубликат от 14.08.2026 был удалён, воссоздавать его
  нельзя.
- `CLAUDE.md` в корне репозитория, разделы «### nginx» (три гоча: reload от
  SYSTEM, рабочий каталог при `-t`, осиротевший мастер), «🔴 Known issue:
  websockets…» и «History: 14.08.2026 port rework» — это и есть цель правки
  документации.
- nginx 1.30.4, стандартная windows-сборка: `nginx.exe -V` перечисляет только
  недефолтные модули, `ngx_http_map_module` входит в сборку по умолчанию —
  `map` доступен, пересборка или замена бинаря не нужны.

# Критерии готовности

Все проверки — снаружи, тем же запросом, что в «Контексте» (заголовки
`Connection: Upgrade`, `Upgrade: websocket`, `Sec-WebSocket-Version: 13`,
`Sec-WebSocket-Key: dGhlIHNhbXBsZSBub25jZQ==`):

- `https://scanvision.online/ws/webSocket/index/1` → **101** с заголовком
  `Sec-WebSocket-Accept`; то же для `https://scanvision.online/ws/webSocket/down/1`
  и `https://test.thedevs.ru/ws/webSocket/index/1`.
- Те же пути по `http://` по-прежнему отвечают **101** — поведение порта 80 не
  изменилось.
- `https://scanvision.online/` и `https://test.thedevs.ru/` отвечают **200**,
  цепочка сертификата валидна — обычный HTTPS не сломан.
- POST 2 000 000 байт на `https://scanvision.online/` больше не даёт 413:
  ответ приходит от приложения (**200**, как сейчас на порту 80), а не от nginx.
- `nginx.exe -p C:\nginx -c conf\nginx.conf -t` — «syntax is ok» и «test is
  successful».
- После рестарта живых процессов nginx ровно два, оба со `StartTime` момента
  рестарта, 443 слушают только они; параллельного старого мастера нет.
- Главный интерфейс CMSV6 по `https://scanvision.online/` обновляет данные в
  реальном времени (проверка глазами в браузере).
- `CLAUDE.md` обновлён по пункту 8, даты замеров проставлены.

# Не трогать

- CMSV6 и его tomcat: `server.xml`, порт 80, служба `gpstomcat6` и все службы
  `GPS*`. В коннектор на порту 80 ни при каких условиях не добавлять
  `address="127.0.0.1"` — десктопный клиент ходит на этот порт напрямую и
  зависает в `SynSent`. Рестарт `gpstomcat6` в этой задаче не нужен вообще.
- Существующие `proxy_pass`, четыре проксирующих заголовка, пути к
  сертификатам, `ssl_stapling`/`ssl_trusted_certificate`/`resolver` в блоке
  scanvision, `worker_processes`, `worker_connections`, `keepalive_timeout`.
- Правила файрвола, DNS, `frps` и его нерабочее состояние (это отдельная
  задача), задача win-acme и сертификаты.
- Задача планировщика `nginx_run` и `run_nginx.bat`; второй задачи nginx не
  создавать.
- Не добавлять `listen 80`, редирект на 443 и HSTS — порт 80 оставлен
  полноценным входом сознательно.
- Версию nginx не поднимать, бинарь не заменять, другие файлы в `specs/` не
  править.
- Значения `LOGIN`, `PASSWORD`, `SERVER`, токена frps и пароля дашборда frps
  не писать ни в репозиторий, ни в описания коммитов, ни в логи команд — только
  их имена.
