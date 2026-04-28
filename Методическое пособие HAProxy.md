# Техническое руководство HAProxy

**HAProxy используется как L4/L7 балансировщик, reverse proxy, TLS terminator и точка наблюдаемости для HTTP/TCP-сервисов.**

**HAProxy Community Edition 2.8+/3.x. Перед эксплуатацией конфигурации проверяются командой `haproxy -c -f <file>` и сверяются с установленной версией через `haproxy -vv`.**

## 1. Область применения

**HAProxy принимает входящие соединения на `frontend`, выбирает правила маршрутизации и передает трафик в `backend` с пулом серверов.**

Основные сценарии:

- HTTP reverse proxy для web/API;
- TCP load balancing для PostgreSQL, MySQL, Redis, SSH, TLS passthrough;
- TLS termination на единой точке входа;
- routing по `Host`, `path`, headers, method, source IP;
- active health checks и автоматическое исключение неисправных серверов;
- sticky sessions через cookie или stick table;
- короткоживущее in-memory HTTP-кеширование;
- stats dashboard, runtime socket, Prometheus exporter и логи.

**Generic UDP не настраивается как обычный `mode udp` рядом с `mode tcp` и `mode http`. Для UDP-сервисов заранее проверяется поддержка конкретной версии/редакции HAProxy и требования протокола.**

## 2. Структура конфигурации

**Конфигурация HAProxy состоит из секций; область действия директив зависит от секции.**

| Секция | Назначение | Типовой параметр |
|---|---|---|
| `global` | Параметры процесса HAProxy | `maxconn`, `log`, `stats socket` |
| `defaults` | Значения по умолчанию для `frontend`, `backend`, `listen` | `mode`, `timeout`, `option httplog` |
| `frontend` | Точка приема клиентского трафика | `bind`, `acl`, `use_backend` |
| `backend` | Пул upstream-серверов | `balance`, `server`, `http-check` |
| `listen` | Совмещенный `frontend` + `backend` | удобно для простых TCP/stats-сценариев |
| `cache` | In-memory HTTP cache | `total-max-size`, `max-object-size`, `max-age` |
| `userlist` | Пользователи для basic auth | `user`, `password`, `insecure-password` |

**Минимальный HTTP reverse proxy содержит `global`, `defaults`, один `frontend` и один `backend`.**

Схема:

```text
+----------+
|  Client  |
+----------+
     |
     v
+----------------------+
| HAProxy fe_http:8080 |
+----------------------+
     |
     v
+----------------------+
| backend be_web       |
+----------------------+
     |--------------------> +----------------------+
     |                     | web1 127.0.0.1:9001 |
     |                     +----------------------+
     |
     `--------------------> +----------------------+
                           | web2 127.0.0.1:9002 |
                           +----------------------+
```

```bash
# Глобальные параметры процесса HAProxy.
global
    # Отправляет логи в stdout; удобно для контейнеров и локальной отладки.
    log stdout format raw local0
    # Ограничивает общее число одновременных соединений процесса.
    maxconn 2000

# Значения по умолчанию для последующих frontend/backend/listen.
defaults
    # Использует log target, объявленный в global.
    log global
    # Включает HTTP-анализ запросов и ответов.
    mode http
    # Пишет HTTP-логи с методом, URL, статусом и таймингами.
    option httplog
    # Не логирует пустые соединения без полезного трафика.
    option dontlognull
    # Максимальное время подключения HAProxy к backend-серверу.
    timeout connect 5s
    # Максимальный простой клиентского соединения.
    timeout client 30s
    # Максимальный простой соединения с backend-сервером.
    timeout server 30s

# Точка входа для клиентских HTTP-запросов.
frontend fe_http
    # Слушает TCP-порт 8080 на всех IPv4-интерфейсах.
    bind *:8080
    # Отправляет все запросы в backend be_web.
    default_backend be_web

# Пул backend-серверов приложения.
backend be_web
    # Распределяет запросы по серверам по очереди.
    balance roundrobin
    # Описывает первый backend-сервер и включает проверку доступности.
    server web1 127.0.0.1:9001 check
    # Описывает второй backend-сервер и включает проверку доступности.
    server web2 127.0.0.1:9002 check
```

Проверка:

```bash
haproxy -c -f haproxy.cfg
```

Запуск через Docker:

```bash
docker run --rm --name haproxy \
  -p 8080:8080 -p 8404:8404 -p 8405:8405 \
  -v "$PWD/haproxy.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro" \
  haproxy:3.2
```

**Базовые `timeout`-параметры задают границы ожидания на разных участках соединения; они не являются одним общим таймаутом запроса.**

| Параметр | Подробное пояснение |
|---|---|
| `timeout connect 5s` | Максимум времени на установку TCP-соединения от HAProxy до backend-сервера. Если backend не принимает соединение за 5 секунд, попытка считается неуспешной. |
| `timeout client 30s` | Максимальный простой клиентской стороны. Если клиент открыл соединение и не отправляет/не читает данные дольше 30 секунд, HAProxy может закрыть соединение. |
| `timeout server 30s` | Максимальный простой server-side соединения. Если backend не отвечает или не передает данные дольше 30 секунд, HAProxy может закрыть соединение. |
| `timeout http-request 10s` | Максимум времени на получение полного HTTP request от клиента. Защищает от медленной отправки headers/body. |
| `timeout queue 1m` | Максимум времени ожидания запроса в очереди backend, если все server заняты лимитами `maxconn`. |
| `timeout http-keep-alive 10s` | Сколько HAProxy держит idle keep-alive соединение в ожидании следующего HTTP request. |
| `timeout check 10s` | Максимальная длительность health check до признания проверки неуспешной. |

**Единицы времени лучше указывать явно: `ms`, `s`, `m`. Запись `inter 500` в старых конфигах обычно читается как миллисекунды, но `inter 500ms` заметно понятнее.**

## 3. Установка и диагностика

**Пакетная установка удобна для системного сервиса; Docker удобен для изолированных стендов и быстрой проверки конфигов.**

Debian/Ubuntu:

```bash
sudo apt update
sudo apt install -y haproxy
haproxy -vv
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
sudo journalctl -u haproxy -f
```

RHEL-like:

```bash
sudo dnf install -y haproxy
haproxy -vv
sudo haproxy -c -f /etc/haproxy/haproxy.cfg
sudo systemctl reload haproxy
```

Базовая диагностика:

```bash
ss -lntp
curl -v http://127.0.0.1:8080/
openssl s_client -connect 127.0.0.1:443 -servername localhost
```

## 4. Режимы TCP и HTTP

**`mode tcp` работает на L4 и передает байтовый поток без HTTP-разбора.**

**`mode http` работает на L7 и позволяет использовать ACL по path/header/method, HTTP redirects, header rewrites, cookies и cache.**

| Режим | Что анализирует HAProxy | Типовые сервисы |
|---|---|---|
| `tcp` | TCP-соединение, SNI при дополнительной настройке inspection | PostgreSQL, MySQL, Redis, SSH, TLS passthrough |
| `http` | HTTP request/response, headers, path, method, cookies | Web, REST API, routing по доменам и путям |

**HTTP ACL не применяются в чистом `mode tcp`. Для HTTP-логики требуется `mode http`.**

TCP passthrough:

Схема:

```text
+--------------+
|  TCP client  |
+--------------+
       |
       v
+---------------------+
| HAProxy fe_tcp:9000 |
+---------------------+
       |
       v
+--------------------------------+
| backend be_tcp_services       |
| algorithm: leastconn          |
+--------------------------------+
       |---------------> +----------------------+
       |                | tcp1 127.0.0.1:9101 |
       |                +----------------------+
       |
       `---------------> +----------------------+
                        | tcp2 127.0.0.1:9102 |
                        +----------------------+
```

```bash
# Глобальные параметры процесса.
global
    # Пишет логи в stdout в простом формате.
    log stdout format raw local0
    # Ограничивает общее число соединений.
    maxconn 2000

# TCP-настройки по умолчанию.
defaults
    # Наследует global log target.
    log global
    # Включает L4-режим без HTTP-разбора.
    mode tcp
    # Использует TCP-формат логов.
    option tcplog
    # Таймаут подключения к backend.
    timeout connect 5s
    # Таймаут простоя клиентской стороны.
    timeout client 1m
    # Таймаут простоя server side.
    timeout server 1m

# Входящий TCP listener.
frontend fe_tcp
    # Принимает TCP-соединения на порту 9000.
    bind *:9000
    # Передает все соединения в TCP backend.
    default_backend be_tcp_services

# Backend для TCP-сервиса.
backend be_tcp_services
    # Выбирает сервер с наименьшим числом активных соединений.
    balance leastconn
    # Первый TCP-сервер с health check каждые 2 секунды.
    server tcp1 127.0.0.1:9101 check inter 2s fall 3 rise 2
    # Второй TCP-сервер с теми же порогами проверки.
    server tcp2 127.0.0.1:9102 check inter 2s fall 3 rise 2
```

## 5. Алгоритмы балансировки

**Алгоритм `balance` должен соответствовать характеру соединений, длительности сессий и требованиям к affinity.**

| Алгоритм | Применение | Особенности |
|---|---|---|
| `roundrobin` | Равные backend-серверы, короткие HTTP-запросы | Хороший базовый выбор |
| `static-rr` | Большие пулы, редко меняющиеся веса | Изменение weight на лету не влияет динамически |
| `leastconn` | Долгие соединения, WebSocket, DB, LDAP | Учитывает число активных соединений |
| `source` | Affinity по IP клиента | NAT может создавать перекос |
| `uri` | Cache locality по URL | Один URI стабильно попадает на один сервер |
| `url_param <name>` | Affinity по query parameter | Работает в HTTP backend |
| `hdr(<name>)` | Распределение по HTTP header | По `Host` или `User-Agent` |
| `random` | Большие фермы | Простое случайное распределение с учетом weight |

Несколько backend:

Схема:

```text
+----------------+
| HTTP traffic   |
+----------------+
        |
        v
+-------------------------+       +----------+
| backend be_roundrobin  | ----> | app1:80  |
| algorithm: roundrobin  |       +----------+
+-------------------------+ ----> +----------+
                                  | app2:80  |
                                  +----------+

+----------------+
| TCP traffic    |
+----------------+
        |
        v
+------------------------+        +-----------+
| backend be_leastconn  | -----> | db1:5432  |
| algorithm: leastconn  |        +-----------+
+------------------------+ -----> +-----------+
                                 | db2:5432  |
                                 +-----------+

+----------------+
| HTTP traffic   |
+----------------+
        |
        v
+------------------------+        +-------------------------+
| backend be_source     | -----> | app1:80, hash source IP |
| algorithm: source     |        +-------------------------+
+------------------------+ -----> +-------------------------+
                                 | app2:80, hash source IP |
                                 +-------------------------+
```

```bash
# HTTP backend с равномерным распределением.
backend be_roundrobin
    # Каждый сервер получает запросы по очереди с учетом weight.
    balance roundrobin
    # Основной сервер с весом 100.
    server app1 10.0.0.11:80 check weight 100
    # Второй сервер с весом 100.
    server app2 10.0.0.12:80 check weight 100

# TCP backend для долгих соединений.
backend be_leastconn
    # Включает L4-режим.
    mode tcp
    # Новое соединение получает сервер с меньшим числом активных соединений.
    balance leastconn
    # Первый backend-сервер базы данных.
    server db1 10.0.1.11:5432 check
    # Второй backend-сервер базы данных.
    server db2 10.0.1.12:5432 check

# HTTP backend с affinity по IP клиента.
backend be_source
    # Выбирает сервер на основе hash от source IP.
    balance source
    # Уменьшает перераспределение ключей при изменении состава backend.
    hash-type consistent
    # Первый сервер приложения.
    server app1 10.0.0.11:80 check
    # Второй сервер приложения.
    server app2 10.0.0.12:80 check
```

## 6. Health checks

**Health checks исключают неисправные backend-серверы из ротации и возвращают их после успешных проверок.**

Параметры `server`:

| Параметр | Подробное назначение |
|---|---|
| `check` | Включает активную проверку сервера. Без `check` HAProxy считает сервер доступным, пока не возникнет ошибка при реальном трафике. |
| `inter 2s` | Базовый интервал между проверками в стабильном состоянии. Если сервер стабильно `UP`, проверка идет раз в `2s`; если не задан `downinter`, для стабильного `DOWN` тоже используется `inter`. |
| `fastinter 500ms` | Ускоренный интервал в переходном состоянии. Применяется, когда сервер еще не признан окончательно `DOWN` или `UP`: был `UP`, получил первую ошибку и набирает счетчик `fall`; либо был `DOWN`, получил первый успех и набирает счетчик `rise`. Это ускоряет подтверждение аварии или восстановления. |
| `downinter 10s` | Интервал проверок для сервера, который уже признан `DOWN`. Обычно делают больше, чем `inter`, чтобы не создавать лишнюю нагрузку на заведомо неисправный узел. После первого успешного check сервер переходит в промежуточное восстановление и может проверяться через `fastinter`. |
| `fall 3` | Количество ошибок подряд, после которого сервер переводится в `DOWN`. При `fall 3` одиночная ошибка не выбивает сервер из ротации; нужно три последовательных неуспешных check. |
| `rise 2` | Количество успешных проверок подряд, после которого `DOWN`-сервер возвращается в `UP`. При `rise 2` один случайный успешный ответ не возвращает сервер в балансировку. |
| `port 8080` | Порт health check отличается от порта реального трафика. Полезно, когда приложение принимает трафик на `80`, а endpoint проверки слушает `8080`. |
| `addr 10.0.0.10` | Адрес health check отличается от адреса реального backend. Используется для sidecar/agent checks или отдельного management IP. |
| `maxconn 100` | Лимит одновременных соединений на конкретный server. При достижении лимита новые запросы ждут в очереди backend или идут на другой server, если это возможно. |
| `weight 50` | Вес сервера в алгоритмах, которые учитывают weight. При `roundrobin` доля запросов растет пропорционально весу сервера. |
| `backup` | Сервер резерва. Он не получает трафик, пока есть доступные основные серверы; включается при отказе основных. |
| `slowstart 30s` | Плавное возвращение сервера после `DOWN` или maintenance. Вес сервера растет постепенно в течение указанного времени, чтобы не отправить весь трафик на только что восстановившийся узел. |

**`inter 2s fastinter 500ms downinter 10s fall 3 rise 2`: стабильный `UP` проверяется каждые 2 секунды; после первой ошибки HAProxy ускоряет проверки до 500 ms и ждет еще ошибки до `fall 3`; после перевода в `DOWN` проверки идут раз в 10 секунд; после первого успешного ответа восстановление подтверждается ускоренными проверками до `rise 2`.**

**HTTP health check выполняется самим HAProxy к backend-серверам; клиентский запрос при этом приходит на `frontend`, а не на `backend` напрямую.**

Схема:

```text
+---------------+
|  HTTP client  |
+---------------+
        |
        v
+----------------------+
| frontend fe_api:8080 |
+----------------------+
        |
        | default_backend be_api
        v
+--------------------------------------+
| backend be_api                       |
+--------------------------------------+
        |----------------> +----------------------+
        |                 | api1 10.0.0.11:8080 |
        |                 +----------------------+
        |
        `----------------> +----------------------+
                          | api2 10.0.0.12:8080 |
                          +----------------------+
```

```bash
# Backend API с HTTP-проверками состояния.
backend be_api
    # Включает HTTP-режим для анализа ответов health endpoint.
    mode http
    # Использует round-robin распределение между доступными серверами.
    balance roundrobin
    # Включает HTTP health check вместо простого TCP connect.
    option httpchk
    # Отправляет GET /healthz с Host header api.local.
    http-check send meth GET uri /healthz ver HTTP/1.1 hdr Host api.local
    # Считает сервер здоровым только при HTTP 200.
    http-check expect status 200
    # Задает общие параметры health checks для всех server ниже.
    default-server inter 2s fastinter 500ms downinter 10s fall 3 rise 2
    # Первый API backend с наследованием default-server.
    server api1 10.0.0.11:8080 check
    # Второй API backend с наследованием default-server.
    server api2 10.0.0.12:8080 check
```

**TCP send/expect применяется для протоколов, где сервис может ответить на простой диагностический запрос.**

```bash
# TCP backend с протокольной проверкой PING/PONG.
backend be_tcp_custom
    # Включает TCP-режим.
    mode tcp
    # Разрешает последовательность tcp-check connect/send/expect.
    option tcp-check
    # Открывает TCP-соединение до сервера.
    tcp-check connect
    # Отправляет строку PING с переводом строки.
    tcp-check send PING\r\n
    # Ожидает строку PONG в ответе сервиса.
    tcp-check expect string PONG
    # Backend-сервер, на котором выполняется tcp-check.
    server srv1 10.0.0.11:1234 check
```

Проверка:

```bash
curl -i http://127.0.0.1:8080/healthz
curl -s http://127.0.0.1:8404/stats
```

## 7. ACL и HTTP access control

**ACL вычисляет логическое условие; поведение меняется только при использовании ACL в действии `use_backend`, `deny`, `auth`, `redirect`, `set-header` и т.д.**

Синтаксис:

```bash
# <name> — имя ACL; <sample_fetch> — источник данных; <value> — шаблон сравнения.
acl <name> <sample_fetch> [flags] [operator] <value>
```

Частые sample fetches:

| Fetch | Что проверяет | Фрагмент |
|---|---|---|
| `src` | IP клиента | `acl office src 10.0.0.0/8` |
| `path` | URL path целиком | `acl exact path /login` |
| `path_beg` | Начало path | `acl api path_beg /api/` |
| `path_reg` | Regex по path | `acl item path_reg ^/items/[0-9]+$` |
| `hdr(host)` | HTTP Host | `acl host_app hdr(host) -i app.local` |
| `hdr_beg(host)` | Начало header | `acl www hdr_beg(host) -i www.` |
| `method` | HTTP method | `acl is_post method POST` |
| `url_param(token)` | Query parameter | `acl has_token url_param(token) -m found` |
| `ssl_fc` | Факт TLS на frontend | `unless { ssl_fc }` |

Флаги:

| Флаг | Назначение |
|---|---|
| `-i` | Case-insensitive сравнение |
| `-m beg` | Значение начинается с шаблона |
| `-m end` | Значение заканчивается шаблоном |
| `-m sub` | Значение содержит подстроку |
| `-m reg` | Regex-сравнение |
| `-f file` | Загрузка значений из файла |

**Routing по `Host` и `path` строится через ACL и `use_backend`.**

Схема:

```text
+----------+      +----------------------+
| Client   | ---> | HAProxy fe_http:8080 |
+----------+      +----------------------+
                         |
       +-----------------+-----------------+
       |                 |                 |
       v                 v                 v
+----------------+ +----------------+ +----------------+
| backend        | | backend        | | backend        |
| be_admin       | | be_api         | | be_web         |
| Host+ /admin   | | Host only      | | default        |
+----------------+ +----------------+ +----------------+
```

```bash
# HTTP frontend с маршрутизацией по домену и пути.
frontend fe_http
    # Принимает HTTP-запросы на порту 8080.
    bind *:8080
    # Проверяет, что Host равен api.example.local без учета регистра.
    acl host_api hdr(host) -i api.example.local
    # Проверяет, что path начинается с /admin.
    acl path_admin path_beg /admin
    # Направляет /admin на отдельный backend только для Host api.example.local.
    use_backend be_admin if host_api path_admin
    # Направляет остальные запросы api.example.local в API backend.
    use_backend be_api if host_api
    # Все прочие запросы идут в backend по умолчанию.
    default_backend be_web
```

**Whitelist и basic auth обычно комбинируются: IP-фильтр отсекает внешний доступ, auth защищает разрешенный периметр.**

```bash
# Список пользователей для basic auth.
userlist admins
    # Тестовая запись; для production используются хешированные пароли.
    user admin insecure-password admin123

# HTTP frontend с доступом к /admin.
frontend fe_http
    # Принимает HTTP-запросы на порту 8080.
    bind *:8080
    # Выделяет административные URL.
    acl admin_path path_beg /admin
    # Задает доверенные IP-сети.
    acl trusted_src src 10.0.0.0/8 192.168.0.0/16 127.0.0.1
    # Возвращает 403 для /admin вне whitelist.
    http-request deny deny_status 403 if admin_path !trusted_src
    # Запрашивает basic auth для /admin, если пользователь не авторизован.
    http-request auth realm AdminArea if admin_path !{ http_auth(admins) }
    # Передает исходную схему запроса в backend.
    http-request set-header X-Forwarded-Proto http
    # Передает IP клиента в backend.
    http-request set-header X-Real-IP %[src]
    # Добавляет защитный response header.
    http-response set-header X-Frame-Options DENY
    # Передает разрешенные запросы в основной backend.
    default_backend be_web
```

## 8. Sticky sessions и persistence

**Sticky sessions сохраняют привязку клиента к одному backend-серверу, но увеличивают зависимость приложения от состояния конкретного узла.**

Cookie-based persistence:

Схема:

```text
+----------+      +----------------------+
| Client   | ---> | HAProxy fe_http      |
+----------+      +----------------------+
                         |
                         v
                  +----------------------+
                  | backend be_app       |
                  | sticky cookie: SRV   |
                  +----------------------+
                         |---------------> +----------------+
                         |                | app1 10.0.0.11 |
                         |                +----------------+
                         |
                         `---------------> +----------------+
                                          | app2 10.0.0.12 |
                                          +----------------+
```

```bash
# HTTP backend с sticky sessions через cookie.
backend be_app
    # Включает HTTP-режим, обязательный для cookie persistence.
    mode http
    # Первичный выбор сервера выполняется round-robin.
    balance roundrobin
    # HAProxy вставляет cookie SRV и не передает служебную cookie приложению.
    cookie SRV insert indirect nocache
    # Сервер app1 получает cookie-значение app1.
    server app1 10.0.0.11:80 check cookie app1
    # Сервер app2 получает cookie-значение app2.
    server app2 10.0.0.12:80 check cookie app2
```

Проверка:

```bash
curl -i http://127.0.0.1:8080/
curl -b "SRV=app1" http://127.0.0.1:8080/
```

IP-based persistence:

Схема:

```text
+----------+      +----------------------+
| Client   | ---> | HAProxy fe_tcp       |
+----------+      +----------------------+
                         |
                         v
                  +----------------------+
                  | backend be_tcp_sticky|
                  | stick on src         |
                  +----------------------+
                         |---------------> +----------------+
                         |                | app1 10.0.0.11 |
                         |                +----------------+
                         |
                         `---------------> +----------------+
                                          | app2 10.0.0.12 |
                                          +----------------+
```

```bash
# TCP backend с persistence по source IP.
backend be_tcp_sticky
    # Включает TCP-режим.
    mode tcp
    # Начальное распределение выполняется round-robin.
    balance roundrobin
    # Создает stick table с IP-ключом, размером 1m записей и TTL 30 минут.
    stick-table type ip size 1m expire 30m
    # Привязывает source IP клиента к выбранному серверу.
    stick on src
    # Первый backend-сервер.
    server app1 10.0.0.11:80 check
    # Второй backend-сервер.
    server app2 10.0.0.12:80 check
```

**Persistence по IP плохо различает клиентов за одним NAT: много пользователей могут попасть на один backend.**

## 9. HTTP cache

**HAProxy cache хранит ответы в памяти и подходит для короткоживущих публичных HTTP-объектов без персональных данных.**

Схема:

```text
+----------+      +----------------------+
| Client   | ---> | HAProxy fe_http:8080 |
+----------+      +----------------------+
                         |
        +----------------+----------------+
        |                                 |
        v                                 v
+-------------------+            +----------------+
| cache short_cache |            | backend be_app |
+-------------------+            +----------------+
                                          |
                                          |----> +-------------+
                                          |      | app1 server |
                                          |      +-------------+
                                          |
                                          `----> +-------------+
                                                 | app2 server |
                                                 +-------------+
```

```bash
# Область in-memory cache.
cache short_cache
    # Максимальный общий размер cache в мегабайтах.
    total-max-size 64
    # Максимальный размер одного объекта в байтах.
    max-object-size 1048576
    # Максимальный срок жизни объекта в секундах.
    max-age 60

# HTTP frontend с cache filter.
frontend fe_http
    # Принимает HTTP-запросы на порту 8080.
    bind *:8080
    # Подключает cache filter к frontend.
    filter cache short_cache
    # Разрешает кеширование только для публичных путей.
    acl cacheable path_beg /static /public /api/catalog
    # Пытается отдать объект из cache для cacheable-запросов.
    http-request cache-use short_cache if cacheable
    # Сохраняет подходящие HTTP-ответы в cache.
    http-response cache-store short_cache
    # Остальные запросы отправляются в приложение.
    default_backend be_app
```

Ограничения:

- кешировать только ответы без персональных данных;
- не кешировать ответы с авторизацией, пользовательскими cookie и приватными headers;
- контролировать `Cache-Control`, `Vary`, размер объекта и TTL;
- помнить, что cache не переживает restart/reload как постоянное хранилище.

## 10. SSL/TLS

**TLS termination завершает HTTPS на HAProxy; backend может получать HTTP или отдельное HTTPS-соединение.**

Self-signed PEM для проверки:

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout site.key -out site.crt -days 7 \
  -subj "/CN=localhost"
cat site.crt site.key > site.pem
```

TLS termination:

Схема:

```text
+-------------------+
| Client HTTPS :443 |
+-------------------+
        |
        v
+-------------------------------------------+
| frontend fe_https                         |
| TLS termination, certificate: site.pem    |
+-------------------------------------------+
        |
        | plain HTTP to backend
        v
+---------------------+
| backend be_app      |
+---------------------+
        |---------------> +--------------------+
        |                | app1 10.0.0.11:80 |
        |                +--------------------+
        |
        `---------------> +--------------------+
                         | app2 10.0.0.12:80 |
                         +--------------------+
```

```bash
# Глобальные параметры процесса и TLS.
global
    # Пишет логи в stdout.
    log stdout format raw local0
    # Ограничивает число одновременных соединений.
    maxconn 2000
    # Запрещает TLS ниже 1.2 и отключает TLS session tickets.
    ssl-default-bind-options ssl-min-ver TLSv1.2 no-tls-tickets
    # Задает набор cipher suites для TLS 1.3.
    ssl-default-bind-ciphersuites TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256:TLS_AES_128_GCM_SHA256

# HTTP-настройки по умолчанию.
defaults
    # Использует global log target.
    log global
    # Включает HTTP-анализ.
    mode http
    # Включает HTTP-формат логов.
    option httplog
    # Таймаут подключения к backend.
    timeout connect 5s
    # Таймаут клиентского соединения.
    timeout client 30s
    # Таймаут соединения с backend.
    timeout server 30s

# Frontend, принимающий HTTP и HTTPS.
frontend fe_https
    # Принимает plain HTTP на 80 порту.
    bind *:80
    # Принимает HTTPS на 443 порту и использует PEM-файл с cert+key.
    bind *:443 ssl crt /etc/haproxy/certs/site.pem alpn h2,http/1.1
    # Перенаправляет plain HTTP на HTTPS.
    http-request redirect scheme https code 301 unless { ssl_fc }
    # Добавляет HSTS только для TLS-запросов.
    http-response set-header Strict-Transport-Security "max-age=31536000; includeSubDomains" if { ssl_fc }
    # Передает запросы приложению.
    default_backend be_app

# Backend приложения по plain HTTP.
backend be_app
    # Распределяет запросы по очереди.
    balance roundrobin
    # Первый backend-сервер.
    server app1 10.0.0.11:80 check
    # Второй backend-сервер.
    server app2 10.0.0.12:80 check
```

TLS до backend:

Схема:

```text
+----------+
| Client   |
+----------+
     |
     v
+----------------+
| HAProxy        |
+----------------+
     |
     v
+----------------------+
| backend be_secure_app |
+----------------------+
     |-- TLS + certificate verification --> +---------------------+
     |                                      | app1 10.0.0.11:443 |
     |                                      +---------------------+
     |
     `-- TLS + certificate verification --> +---------------------+
                                            | app2 10.0.0.12:443 |
                                            +---------------------+
```

```bash
# Backend с шифрованием соединений HAProxy -> application.
backend be_secure_app
    # Включает HTTP-режим.
    mode http
    # Проверяет сертификат backend-сервера через указанный CA-файл.
    server app1 10.0.0.11:443 ssl verify required ca-file /etc/haproxy/ca.pem check
    # Второй backend-сервер с такой же проверкой сертификата.
    server app2 10.0.0.12:443 ssl verify required ca-file /etc/haproxy/ca.pem check
```

**`verify none` отключает проверку сертификата backend и не является production-настройкой.**

Проверка:

```bash
curl -k -I https://127.0.0.1/
openssl s_client -connect 127.0.0.1:443 -servername localhost
```

## 11. Мониторинг и логирование

**Stats dashboard показывает состояние frontend/backend/server в браузере; Prometheus exporter отдает метрики для внешней системы мониторинга.**

Stats dashboard:

Схема:

```text
+---------+
| Browser |
+---------+
     |
     v
+----------------------+
| frontend fe_stats    |
| bind :8404           |
+----------------------+
     |
     v
+-----------------------------+
| /stats                      |
| protected by stats auth     |
+-----------------------------+
```

```bash
# HTTP frontend для встроенной stats-страницы.
frontend fe_stats
    # Принимает HTTP-запросы stats dashboard на порту 8404.
    bind *:8404
    # Включает HTTP-режим для stats UI.
    mode http
    # Активирует встроенную stats-страницу.
    stats enable
    # Публикует dashboard по пути /stats.
    stats uri /stats
    # Обновляет страницу каждые 10 секунд.
    stats refresh 10s
    # Защищает dashboard basic auth.
    stats auth admin:change_me
```

Runtime socket:

```bash
# Глобальные параметры runtime API.
global
    # Создает Unix socket с admin-доступом к runtime API.
    stats socket /tmp/haproxy-admin.sock mode 660 level admin expose-fd listeners
    # Таймаут команд runtime API.
    stats timeout 30s
```

Команды runtime API:

```bash
echo "show info" | socat - /tmp/haproxy-admin.sock
echo "show stat" | socat - /tmp/haproxy-admin.sock
echo "disable server be_app/app1" | socat - /tmp/haproxy-admin.sock
echo "enable server be_app/app1" | socat - /tmp/haproxy-admin.sock
```

Prometheus exporter:

Схема:

```text
+------------+
| Prometheus |
+------------+
      |
      v
+-----------------------------+
| frontend fe_prometheus      |
| bind :8405                  |
+-----------------------------+
      |
      v
+-----------------------------+
| /metrics                    |
| prometheus-exporter service |
+-----------------------------+
```

```bash
# HTTP frontend для Prometheus scrape.
frontend fe_prometheus
    # Принимает scrape-запросы на порту 8405.
    bind *:8405
    # Включает HTTP-режим.
    mode http
    # Отдает встроенный prometheus-exporter только по path /metrics.
    http-request use-service prometheus-exporter if { path /metrics }
    # Отключает access logs для scrape-запросов.
    no log
```

`prometheus.yml`:

```yaml
# Список jobs для Prometheus.
scrape_configs:
  # Job для сбора метрик HAProxy.
  - job_name: haproxy
    # Статический список endpoint-адресов.
    static_configs:
      # HAProxy Prometheus frontend.
      - targets: ["127.0.0.1:8405"]
```

Logging:

```bash
# Глобальный log target для systemd/syslog.
global
    # Отправляет логи в локальный syslog socket.
    log /dev/log local0

# Общие настройки логирования HTTP.
defaults
    # Использует log target из global.
    log global
    # Включает HTTP-лог.
    option httplog
    # Отключает логирование пустых соединений.
    option dontlognull

# HTTP frontend с захватом headers в лог.
frontend fe_http
    # Принимает HTTP-запросы на 8080.
    bind *:8080
    # Захватывает Host header длиной до 64 символов.
    capture request header Host len 64
    # Захватывает User-Agent длиной до 128 символов.
    capture request header User-Agent len 128
    # Передает запросы в backend приложения.
    default_backend be_app
```

## 12. Полная конфигурация

**Полный шаблон объединяет HTTP routing, access control, health checks, sticky sessions, cache, stats и Prometheus endpoint.**

Схема:

```text
+---------------+      +----------------------+
| HTTP client   | ---> | frontend fe_http     |
|               |      | bind :8080           |
+---------------+      +----------------------+
                            |
              +-------------+-------------+
              |                           |
              v                           v
      +----------------+          +----------------------+
      | backend be_api |          | backend be_web       |
      +----------------+          | cache short_cache    |
              |                   +----------------------+
              |                           |
              v                           v
      +--------------+            +--------------------+
      | api1:9011    |            | web1:9001          |
      | api2:9012    |            | web2:9002          |
      +--------------+            +--------------------+

+---------+      +-------------------+      +--------+
| Browser | ---> | frontend fe_stats | ---> | /stats |
+---------+      +-------------------+      +--------+

+------------+      +------------------------+      +----------+
| Prometheus | ---> | frontend fe_prometheus | ---> | /metrics |
+------------+      +------------------------+      +----------+

```

```bash
# Глобальные параметры процесса.
global
    # Пишет логи в stdout.
    log stdout format raw local0 info
    # Ограничивает максимальное число соединений.
    maxconn 4000
    # Включает runtime socket для административных команд.
    stats socket /tmp/haproxy-admin.sock mode 660 level admin expose-fd listeners
    # Задает таймаут runtime API.
    stats timeout 30s

# Область in-memory cache.
cache short_cache
    # Максимальный размер cache в MB.
    total-max-size 64
    # Максимальный объект cache в bytes.
    max-object-size 1048576
    # TTL cache-объекта в секундах.
    max-age 60

# Общие HTTP-настройки.
defaults
    # Использует global log target.
    log global
    # Включает HTTP-режим.
    mode http
    # Включает HTTP-формат логов.
    option httplog
    # Скрывает пустые соединения из логов.
    option dontlognull
    # Таймаут подключения к backend.
    timeout connect 5s
    # Таймаут клиента.
    timeout client 30s
    # Таймаут backend-сервера.
    timeout server 30s

# Пользователи для basic auth.
userlist admins
    # Тестовая учетная запись администратора.
    user admin insecure-password admin123

# Основной HTTP frontend.
frontend fe_http
    # Принимает HTTP-запросы.
    bind *:8080
    # Подключает cache filter.
    filter cache short_cache
    # Определяет административные URL.
    acl admin_path path_beg /admin
    # Определяет доверенные IP-адреса.
    acl trusted_src src 127.0.0.1 10.0.0.0/8 192.168.0.0/16
    # Определяет API URL.
    acl api_path path_beg /api/
    # Определяет кешируемые публичные URL.
    acl cacheable path_beg /static /public
    # Блокирует /admin вне whitelist.
    http-request deny deny_status 403 if admin_path !trusted_src
    # Включает basic auth для /admin.
    http-request auth realm AdminArea if admin_path !{ http_auth(admins) }
    # Передает IP клиента в backend.
    http-request set-header X-Real-IP %[src]
    # Пробует отдать cacheable-запрос из cache.
    http-request cache-use short_cache if cacheable
    # Сохраняет подходящие ответы в cache.
    http-response cache-store short_cache
    # Направляет API-запросы в API backend.
    use_backend be_api if api_path
    # Остальные запросы идут в web backend.
    default_backend be_web

# Web backend со sticky cookie.
backend be_web
    # Распределяет новые клиенты по очереди.
    balance roundrobin
    # Вставляет cookie SRV для persistence.
    cookie SRV insert indirect nocache
    # Включает HTTP health check.
    option httpchk
    # Проверяет endpoint /healthz.
    http-check send meth GET uri /healthz ver HTTP/1.1 hdr Host web.local
    # Требует статус 200.
    http-check expect status 200
    # Первый web-сервер.
    server web1 127.0.0.1:9001 check cookie web1
    # Второй web-сервер.
    server web2 127.0.0.1:9002 check cookie web2

# API backend.
backend be_api
    # Выбирает сервер с меньшим числом активных соединений.
    balance leastconn
    # Включает HTTP health check.
    option httpchk
    # Проверяет endpoint /healthz.
    http-check send meth GET uri /healthz ver HTTP/1.1 hdr Host api.local
    # Требует статус 200.
    http-check expect status 200
    # Первый API-сервер.
    server api1 127.0.0.1:9011 check inter 2s fall 3 rise 2
    # Второй API-сервер.
    server api2 127.0.0.1:9012 check inter 2s fall 3 rise 2

# Stats dashboard.
frontend fe_stats
    # Принимает dashboard-запросы.
    bind *:8404
    # Включает HTTP-режим.
    mode http
    # Активирует stats UI.
    stats enable
    # Публикует stats UI по /stats.
    stats uri /stats
    # Автообновляет страницу каждые 10 секунд.
    stats refresh 10s
    # Защищает stats UI паролем.
    stats auth admin:change_me

# Prometheus endpoint.
frontend fe_prometheus
    # Принимает scrape-запросы.
    bind *:8405
    # Включает HTTP-режим.
    mode http
    # Отдает метрики по /metrics.
    http-request use-service prometheus-exporter if { path /metrics }
    # Отключает логи для scrape.
    no log
```

## 13. Покрытие исходных конфигов

**Директивы из `haproxy_20231124.cfg` и `haproxy.cfg` покрываются отдельными техническими блоками: системные параметры процесса, расширенные defaults, userlist, TLS redirect, HTTP/TCP backends, ACL routing, blocklist-файлы и stats admin.**

Сверенные источники:

- `https://raw.githubusercontent.com/kmmorozov/configs/refs/heads/main/haproxy_20231124.cfg`
- `https://raw.githubusercontent.com/kmmorozov/configs/refs/heads/main/haproxy.cfg`

### 13.1. Global и Defaults

**Параметры `chroot`, `pidfile`, `user`, `group` и `daemon` относятся к модели запуска процесса, а не к маршрутизации трафика.**

```bash
# Глобальные параметры процесса.
global
    # Отправляет логи на локальный syslog endpoint с facility local2.
    log 127.0.0.1 local2
    # Ограничивает файловую область процесса каталогом /var/lib/haproxy.
    chroot /var/lib/haproxy
    # Пишет PID HAProxy в указанный файл.
    pidfile /var/run/haproxy.pid
    # Ограничивает общее число одновременных соединений.
    maxconn 4000
    # Сбрасывает привилегии до пользователя haproxy после bind privileged ports.
    user haproxy
    # Сбрасывает группу процесса до haproxy.
    group haproxy
    # Переводит HAProxy в daemon mode при запуске без systemd foreground.
    daemon
    # Создает Unix socket для stats/runtime API.
    stats socket /var/lib/haproxy/stats
    # Использует системный crypto policy для входящих TLS bind.
    ssl-default-bind-ciphers PROFILE=SYSTEM
    # Использует системный crypto policy для TLS-соединений к backend.
    ssl-default-server-ciphers PROFILE=SYSTEM

# Общие HTTP defaults.
defaults
    # Включает HTTP-режим по умолчанию.
    mode http
    # Использует log target из global.
    log global
    # Альтернатива из исходного конфига: задать log target прямо в defaults.
    # log 127.0.0.1 local2
    # Включает HTTP-формат логов.
    option httplog
    # Не логирует пустые соединения.
    option dontlognull
    # Закрывает server-side HTTP-соединение после ответа.
    option http-server-close
    # Добавляет X-Forwarded-For, кроме клиентов из loopback-сети.
    option forwardfor except 127.0.0.0/8
    # При ошибке соединения разрешает выбрать другой server.
    option redispatch
    # Задает число повторных попыток подключения к backend.
    retries 3
    # Ограничивает время ожидания полного HTTP request.
    timeout http-request 10s
    # Ограничивает ожидание в backend queue.
    timeout queue 1m
    # Ограничивает подключение к backend.
    timeout connect 10s
    # Ограничивает простой клиента.
    timeout client 1m
    # Ограничивает простой backend-сервера.
    timeout server 1m
    # Ограничивает простой keep-alive соединения.
    timeout http-keep-alive 10s
    # Ограничивает длительность health check.
    timeout check 10s
    # Ограничивает соединения на proxy section.
    maxconn 3000
```

Технические замечания:

- **`PROFILE=SYSTEM` характерен для систем с crypto policies, в том числе RHEL-like окружений; на другой сборке HAProxy настройку нужно проверять через `haproxy -c`.**
- **`chroot` меняет видимость путей для runtime socket, сертификатов, ACL-файлов и лог-сокетов; пути должны существовать внутри chroot.**
- **`option forwardfor` добавляет `X-Forwarded-For`; backend-приложение должно доверять этому header только от HAProxy.**
- **`option redispatch` и `retries` помогают пережить отказ выбранного backend-сервера, но могут повторить запрос к другому серверу. Для неидемпотентных операций это учитывается отдельно.**

### 13.2. Userlist: открытый и хешированный пароль

**`insecure-password` хранит пароль в открытом виде; `password` хранит хеш и предпочтителен для постоянных конфигураций.**

```bash
# Userlist с открытыми паролями.
userlist mycredentials
    # Пользователь с открытым паролем.
    user kirill insecure-password 1985
    # Второй пользователь с открытым паролем.
    user nikolay insecure-password 2013

# Userlist с хешированным паролем.
userlist mycredentials2
    # Пользователь с SHA-crypt хешем.
    user anna password $5$NabWwyMwve.3Vmtx$BpdKLBmJo3K76/kXEPnffN/unaUeuoXLzdewvGycN45

# Frontend с basic auth.
frontend fe_auth
    # Принимает HTTP-запросы на порту 81.
    bind *:81
    # Запрашивает basic auth, если пользователь не прошел проверку.
    http-request auth unless { http_auth(mycredentials2) }
    # Передает авторизованный запрос в backend.
    default_backend be_apache3
```

### 13.3. Static routing через `path_beg` и `path_end`

**Статический контент удобно отделять ACL-правилами по началу пути и расширению файла.**

Схема:

```text
+----------+      +--------------------+
| Client   | ---> | frontend main:5000 |
+----------+      +--------------------+
                         |
       +-----------------+------------------+
       | static path or static extension    | other
       v                                    v
+----------------+                  +----------------+
| backend static |                  | backend app    |
+----------------+                  +----------------+
       |                                  |
       v                                  v
+------------------------+       +-------------------------+
| static 127.0.0.1:4331 |       | app1..app4 127.0.0.1   |
+------------------------+       +-------------------------+
```

```bash
# Основной HTTP frontend.
frontend main
    # Принимает HTTP-запросы на порту 5000.
    bind *:5000
    # Определяет static URL по началу path.
    acl url_static path_beg -i /static /images /javascript /stylesheets
    # Определяет static URL по расширению path.
    acl url_static path_end -i .jpg .gif .png .css .js
    # Отправляет static-запросы в backend static.
    use_backend static if url_static
    # Остальные запросы идут в backend app.
    default_backend app

# Backend статического контента.
backend static
    # Распределяет запросы по очереди.
    balance roundrobin
    # Единственный static-сервер.
    server static 127.0.0.1:4331 check

# Backend приложения.
backend app
    # Распределяет запросы по очереди.
    balance roundrobin
    # Первый app server.
    server app1 127.0.0.1:5001 check
    # Второй app server.
    server app2 127.0.0.1:5002 check
    # Третий app server.
    server app3 127.0.0.1:5003 check
    # Четвертый app server.
    server app4 127.0.0.1:5004 check
```

### 13.4. TLS bind и redirect по `ssl_fc`

**Fetch `ssl_fc` показывает, что клиентское соединение на frontend установлено через TLS.**

Схема:

```text
+----------------+
| Client :80     |
+----------------+
        |
        v
+-------------------+
| frontend fe_https|
+-------------------+

+----------------+
| Client :443 TLS|
+----------------+
        |
        v
+-------------------+
| frontend fe_https|
+-------------------+
        |
        v
+----------------------+
| backend be_apache3   |
+----------------------+
```

```bash
# HTTP/HTTPS frontend.
frontend fe_https
    # Включает HTTP-режим.
    mode http
    # Принимает plain HTTP.
    bind *:80
    # Принимает HTTPS и использует PEM-файл с сертификатом и private key.
    bind *:443 ssl crt /etc/ssl/certs/xzxzxz.pem
    # Перенаправляет plain HTTP на HTTPS.
    http-request redirect scheme https unless { ssl_fc }
    # Передает TLS-запросы в backend.
    default_backend be_apache3
```

### 13.5. Алгоритмы из исходных конфигов

**В исходных конфигах перечислены несколько алгоритмов; активным должен быть только один `balance` в конкретном backend.**

| Алгоритм | Смысл |
|---|---|
| `roundrobin` | Очередность с учетом `weight` |
| `leastconn` | Сервер с меньшим числом активных соединений |
| `first` | Первый доступный сервер из списка |
| `uri` | Хеширование URI |
| `source` | Хеширование IP источника |
| `url_param <name>` | Хеширование параметра запроса |
| `hdr(User-Agent)` | Хеширование значения HTTP header |

Схема:

```text
+-------------+      +--------------------+
| HTTP client | ---> | backend be_apache  |
+-------------+      +--------------------+
                           |
        +------------------+------------------+
        |                  |                  |
        v                  v                  v
+-------------------+ +-------------------+ +-------------------+
| ap1 192.168.20.49| | ap3 192.168.20.48| | ap2 192.168.20.51|
+-------------------+ +-------------------+ +-------------------+
```

```bash
# Backend с hash по User-Agent.
backend be_apache
    # Альтернатива: распределение по очереди.
    # balance roundrobin
    # Альтернатива: сервер с меньшим числом активных соединений.
    # balance leastconn
    # Альтернатива: первый доступный сервер из списка.
    # balance first
    # Альтернатива: хеширование URI.
    # balance uri
    # Альтернатива: хеширование IP источника.
    # balance source
    # Альтернатива: хеширование query parameter.
    # balance url_param user_id
    # Хеширует значение HTTP header User-Agent.
    balance hdr(User-Agent)
    # Первый сервер с weight 10 и check interval 500 ms.
    server ap1 192.168.20.49:80 check inter 500 fall 10 rise 10 weight 10
    # Второй сервер с weight 10 и check interval 3000 ms.
    server ap3 192.168.20.48:80 check inter 3000 fall 10 rise 10 weight 10
    # Третий сервер с weight 10 и check interval 4000 ms.
    server ap2 192.168.20.51:80 check inter 4000 fall 10 rise 10 weight 10
```

**Если время указано числом без суффикса, в типовой HAProxy-конфигурации оно интерпретируется как миллисекунды; для читаемости предпочтительны явные суффиксы `ms`, `s`, `m`.**

### 13.6. Старый и новый синтаксис HTTP health checks

**`option httpchk GET /path` встречается в старых и коротких конфигах; современный вариант через `http-check send` и `http-check expect` более явный.**

```bash
# Backend со старым кратким синтаксисом httpchk.
backend be_apache2
    # Проверяет HTTP endpoint GET /health/index.html.
    option httpchk GET /health/index.html
    # Первый backend-сервер.
    server apache1 172.16.30.125:8080 weight 90 check
    # Второй backend-сервер.
    server apache2 172.16.30.136:8080 weight 10 check

# Backend с современным синтаксисом http-check.
backend be_apache3
    # Включает HTTP health check.
    option httpchk
    # Отправляет GET /health.
    http-check send meth GET uri /health
    # Требует статус 200.
    http-check expect status 200
    # Первый backend-сервер.
    server ap1 192.168.20.49:80 check inter 500 fall 10 rise 10 weight 10
```

### 13.7. ACL по Host, path и внешнему файлу

**ACL с `-f` загружает список значений из файла; файл должен существовать и быть доступен HAProxy с учетом `chroot` и прав пользователя.**

Схема:

```text
+----------+      +----------------------+
| Client   | ---> | fe_host_routing:82   |
+----------+      +----------------------+
                         |
       +-----------------+------------------+
       |                 |                  |
       v                 v                  v
+----------------+ +----------------+ +----------------+
| backend        | | backend        | | backend        |
| be_apache      | | be_apache2     | | be_apache3     |
+----------------+ +----------------+ +----------------+

+----------+      +----------------------+
| Client   | ---> | fe_blocklist:83      |
+----------+      +----------------------+
                         |
                         v
                  +----------------+
                  | backend        |
                  | be_apache2     |
                  +----------------+
```

```bash
# Frontend с маршрутизацией по Host и path.
frontend fe_host_routing
    # Принимает HTTP-запросы на порту 82.
    bind *:82
    # Находит подстроку firefox в User-Agent без учета регистра.
    acl blocked_ua hdr_sub(user-agent) -i firefox
    # Проверяет Host www.test1.local.
    acl site1 hdr(host) -i www.test1.local
    # Проверяет Host www.test2.local.
    acl site2 hdr(host) -i www.test2.local
    # Проверяет Host www.test3.local.
    acl site3 hdr(host) -i www.test3.local
    # Проверяет path, начинающийся с /static/.
    acl is_static path -i -m beg /static/
    # Блокирует запросы с запрещенным User-Agent.
    http-request deny if blocked_ua
    # Маршрутизирует site1.
    use_backend be_apache if site1
    # Маршрутизирует site2.
    use_backend be_apache2 if site2
    # Маршрутизирует site3.
    use_backend be_apache3 if site3
    # Маршрутизирует static path.
    use_backend be_apache3 if is_static

# Frontend с blocklist из файла.
frontend fe_blocklist
    # Принимает HTTP-запросы на порту 83.
    bind *:83
    # Включает HTTP-режим.
    mode http
    # Загружает заблокированные IP/CIDR из файла.
    acl is_blocked_ip src -f /etc/haproxy/blocklisted.ips
    # Отказывает клиентам из blocklist.
    http-request deny if is_blocked_ip
    # Остальные запросы отправляет в backend.
    default_backend be_apache2
```

### 13.8. TCP MySQL backend

**Для MySQL используется `mode tcp`: HAProxy не анализирует SQL-протокол, а балансирует TCP-соединения.**

Схема:

```text
+-------------------+
| MySQL client      |
+-------------------+
        |
        v
+-----------------------------+
| frontend fe_mysql:3306      |
| mode tcp                    |
+-----------------------------+
        |
        v
+-----------------------------+
| backend be_mysql_servers    |
| balance roundrobin          |
+-----------------------------+
        |---------------> +-----------------------------+
        |                | s1 192.168.20.49:3306      |
        |                | weight 1                    |
        |                +-----------------------------+
        |---------------> +-----------------------------+
        |                | s2 192.168.20.48:3306      |
        |                | weight 5                    |
        |                +-----------------------------+
        `---------------> +-----------------------------+
                         | s3 192.168.20.51:3306      |
                         | weight 10                   |
                         +-----------------------------+
```

```bash
# TCP frontend для MySQL.
frontend fe_mysql
    # Включает TCP-режим.
    mode tcp
    # Принимает MySQL-соединения на 3306.
    bind :3306
    # Передает соединения в TCP backend.
    default_backend be_mysql_servers

# TCP backend для MySQL.
backend be_mysql_servers
    # Включает TCP-режим.
    mode tcp
    # Распределяет соединения по очереди с учетом weight.
    balance roundrobin
    # Первый MySQL-сервер.
    server s1 192.168.20.49:3306 check inter 500 fall 10 rise 10 weight 1
    # Второй MySQL-сервер.
    server s2 192.168.20.48:3306 check inter 500 fall 10 rise 10 weight 5
    # Третий MySQL-сервер.
    server s3 192.168.20.51:3306 check inter 500 fall 10 rise 10 weight 10
```

### 13.9. Stats admin и валидация ошибок

**`stats admin if LOCALHOST` разрешает административные действия в stats UI только локальному клиенту.**

Схема:

```text
+---------+      +---------------+      +-----------------+
| Browser | ---> | fe_stats:8404 | ---> | /stats dashboard|
+---------+      +---------------+      +-----------------+
```

```bash
# Stats frontend.
frontend fe_stats
    # Включает HTTP-режим.
    mode http
    # Принимает stats-запросы на порту 8404.
    bind *:8404
    # Активирует stats UI.
    stats enable
    # Публикует stats UI по /stats.
    stats uri /stats
    # Обновляет stats UI каждые 10 секунд.
    stats refresh 10s
    # Разрешает admin mode только для predefined ACL LOCALHOST.
    stats admin if LOCALHOST
```

**В `haproxy_20231124.cfg` встречается строка `aacl block-ua ...`; это не директива HAProxy. Корректная директива — `acl`. Такой дефект должен обнаруживаться через `haproxy -c -f <file>` до reload.**

## 14. Troubleshooting

**Диагностика начинается с синтаксиса, портов, состояния backend-серверов и логов.**

| Симптом | Проверка | Вероятная причина |
|---|---|---|
| HAProxy не стартует | `haproxy -c -f haproxy.cfg` | Синтаксис, отсутствующий cert file, неверная секция, опечатка вроде `aacl` вместо `acl` |
| Port already in use | `ss -lntp` | Порт занят другим процессом |
| `503 Service Unavailable` | stats dashboard, logs | Все backend-серверы DOWN |
| ACL не срабатывает | проверить `mode http` | HTTP ACL используется в TCP mode |
| Sticky не работает | `curl -i`, cookie jar | Нет `cookie` на backend/server или клиент не возвращает cookie |
| Health check DOWN | backend logs, `curl /healthz` | Неверный path/status/Host header |
| Нет логов | `log global`, syslog/journal | Не настроен log target или syslog |
| TLS ошибка | `openssl s_client` | Нет private key в PEM, неверный SNI/cert path |

## 15. Готовые конфигурации

**Каталог `examples/` содержит комментированные конфиги для отдельных технических сценариев.**

- `01-basic-http.cfg` — первый HTTP reverse proxy.
- `02-balancing-tcp.cfg` — TCP passthrough и `leastconn`.
- `03-health-checks.cfg` — HTTP health checks с `fall`/`rise`.
- `04-acl-auth-whitelist.cfg` — ACL, whitelist, basic auth, headers.
- `05-cookies-cache.cfg` — sticky cookie и in-memory cache.
- `06-ssl-termination.cfg` — TLS termination, redirect, HSTS.
- `07-monitoring-logging-prometheus.cfg` — stats dashboard, socket, Prometheus.
- `08-source-config-patterns.cfg` — паттерны из исходных конфигов: global/defaults, auth, ACL, TLS, TCP, stats.

## 16. Источники

- HAProxy Configuration Manual 3.2: https://docs.haproxy.org/3.2/configuration.html
- HAProxy configuration tutorials: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/
- Configuration basics: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/
- Backends and balancing: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/configuration-basics/backends/
- ACLs: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/proxying-essentials/custom-rules/acls/
- Health checks: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/reliability/health-checks/
- Session persistence: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/session-persistence/
- Caching: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/performance/caching/
- SSL/TLS: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/ssl-tls/
- Prometheus metrics: https://www.haproxy.com/documentation/haproxy-configuration-tutorials/alerts-and-monitoring/prometheus/
- Исходный конфиг `haproxy_20231124.cfg`: https://raw.githubusercontent.com/kmmorozov/configs/refs/heads/main/haproxy_20231124.cfg
- Исходный конфиг `haproxy.cfg`: https://raw.githubusercontent.com/kmmorozov/configs/refs/heads/main/haproxy.cfg
