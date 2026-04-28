# HAProxy TextMate Bundle для PyCharm

Локальный bundle добавляет подсветку синтаксиса для HAProxy-конфигов и Markdown-блоков вида:

````markdown
```haproxy
global
    maxconn 2000
```
````

В основном Markdown-файле HAProxy-блоки уже размечены как ` ```haproxy `.

## Подключение в PyCharm

1. Открой `Settings | Plugins` и проверь, что `TextMate Bundles` включен.
2. Открой `Settings | Editor | TextMate Bundles`.
3. Нажми `+` и выбери эту директорию:

   `/home/kirill/PycharmProjects/Haproxy_course/tools/pycharm/haproxy-textmate`

4. Примени настройки и переоткрой `examples/*.cfg`.

Bundle регистрирует `.cfg`, `.conf` и `haproxy.cfg` как HAProxy-файлы.
