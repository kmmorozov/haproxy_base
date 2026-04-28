# Комментированные конфигурации HAProxy

**Файлы содержат технические сценарии HAProxy и комментарии к параметрам конфигурации.**

| Файл | Тема |
|---|---|
| `01-basic-http.cfg` | Первый HTTP reverse proxy |
| `02-balancing-tcp.cfg` | TCP passthrough и `leastconn` |
| `03-health-checks.cfg` | HTTP health checks |
| `04-acl-auth-whitelist.cfg` | ACL, whitelist, basic auth, headers |
| `05-cookies-cache.cfg` | Sticky cookie и cache |
| `06-ssl-termination.cfg` | TLS termination |
| `07-monitoring-logging-prometheus.cfg` | Stats, logs, Prometheus |
| `08-source-config-patterns.cfg` | Паттерны из исходных конфигов: global/defaults, auth, ACL, TLS, TCP, stats |

Проверка синтаксиса:

```bash
haproxy -c -f examples/01-basic-http.cfg
```

Запуск через Docker:

```bash
docker run --rm --name haproxy \
  -p 8080:8080 -p 8404:8404 -p 8405:8405 \
  -v "$PWD/examples/01-basic-http.cfg:/usr/local/etc/haproxy/haproxy.cfg:ro" \
  haproxy:3.2
```

Для `06-ssl-termination.cfg` нужен PEM-файл по пути `/etc/haproxy/certs/site.pem` внутри окружения HAProxy. Для локальной проверки путь можно заменить на доступный файл и собрать PEM так:

```bash
openssl req -x509 -newkey rsa:2048 -nodes \
  -keyout site.key -out site.crt -days 7 \
  -subj "/CN=localhost"
cat site.crt site.key > site.pem
```
