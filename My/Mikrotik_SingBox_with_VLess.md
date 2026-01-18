---
tags:
  - mikrotik
  - vpn
  - xray
  - my-config
  - sing-box
---

В данной статье будет содержаться исчерпывающая инструкция как дибилу, который ничего не знает о сетях (и который за каким-то хером купил себе Mikrotik), развернуть на этом самом микротике продвинутый обход блокировок.
Ладно, не исчерпывающая. 
Итак, к делу. Данная инструкция состоит из следующих пунктов:

1. Кратное описание того что НЕ включено в данную статью
2. В каком случае это вообще будет работать
3. Включение функционала контейнеров
4. Настойка бриджа и интерфейса для работы контейнера
5. Настройка хранилища для контейнеров
6. Запуск контейнера [[sing-box]]
7. Настройка маршрутизации трафика через контейнер 

## Что не включено в данную статью
#### Firewall
Я не буду включать сюда настройки firewall, предполагая что с этим пользователь справится самостоятельно. Ну или сделает универсальную и **абсолютно небезопасную и не рекомендуемую политику** следующего вида:

```
/ip/firewall/filter/add chain=input action=accept
/ip/firewall/filter/add chain=forward action=accept
/ip/firewall/filter/add chain=output action=accept
```
 В этом случае никаких дополнительных настроек firewall не потребуется, однако пользователь делает это на свой страх и риск, поскольку данная политика фактически отключает firewall и разрешает любые соединения с вашим роутером откуда угодно.

#### Настройка сервера XRAY
Также в данной статье не будет инструкции по настройке сервера xray и генерации рабочего конфига. Предполагается что у пользователя уже есть файл **config.json**, который протестирован и работает. В интернете достаточно статей на эту тему. Впрочем, я просто приложу сюда конфиг, который работает у меня, вырезав из него адрес своего сервера и ключ подключения. 

```json
{
    "dns": {
        "independent_cache": true,
        "servers": [{
                "address": "https://8.8.8.8/dns-query",
                "detour": "proxy",
                "strategy": "",
                "tag": "dns-remote"
            },
            {
                "tag": "local-dns",
                "address": "8.8.8.8",
                "detour": "direct-out"
            }]
        },
    "inbounds": [{
            "auto_route": true,
            "inet4_route_exclude_address": [
                "192.168.0.0/16"
            ],
            "domain_strategy": "",
            "endpoint_independent_nat": true,
            "inet4_address": "172.19.0.1/28",
            "interface_name": "singbox",
            "mtu": 9000,
            "sniff": true,
            "sniff_override_destination": false,
            "stack": "gvisor",
            "strict_route": true,
            "tag": "tun-in",
            "type": "tun"
        }
    ],
    "log": {
        "level": "info"
    },
    "outbounds": [{
            "domain_strategy": "",
            "flow": "xtls-rprx-vision",
            "packet_encoding": "",
            "server": "get this from xray server",
            "server_port": 443,
            "tag": "proxy",
            "tls": {
                "enabled": true,
                "reality": {
                    "enabled": true,
                    "public_key": "get this from xray server",
                    "short_id": "get this from xray server"
                },
                "server_name": "microsoft.com",
                "utls": {
                    "enabled": true,
                    "fingerprint": "random"
                }
            },
            "type": "vless",
            "uuid": "get this from xray server"
        }, 
        {
            "tag": "dns-out",
            "type": "dns"
        }
    ],
    "route": {
        "auto_detect_interface": true,
        "final": "proxy",
        "rules": [{
                "outbound": "dns-out",
                "protocol": "dns"
            }
        ]
    }
}
```

## В каком случае это вообще будет работать

Сразу следует сказать, что Mikrotik не поддерживает установку кастомных исполняемых файлов (бинарей), поэтому разворачивать Sing-box мы будем в докере (container, в терминологии MikroTik). Не всякий микрот поддерживает контейнеры, только те что имеют архитектуру arm, arm64 и x86. По [ссылке](https://mikrotik.com/products/matrix) ниже можно узнать архитектуру своего роутера. Также для корректной работы контейнера роутер должен либо иметь USB порт (в этом случае мы воткнем в него флешку и будем ее использовать в качестве хранилища данных контейнеров), либо иметь хотя бы 1Gb оперативной памяти (в этом случае мы сможем выделить часть ее под эти цели)

## Включение функционала контейнеров

Допустим, ты выяснил что твоя модель имеет подходящую архитектуру. Теперь нужно обновить систему до версии. Нужна версия 7.15.3 или выше. Так что смотрим текущую и если надо - обновляемся. Если текущая версия совсем старая, может понадобиться запусть процесс обновления несколько раз (нельзя просто взять и перейти с любой версии на любую другую)

```
/system/package/update/check-for-updates
/system/package/update/install
```

Теперь нам нужно включить функционал контейнеров. 

`/system/device-mode/update container=yes`

После этой команды будет необходимо в течение 5 минут перезагрузить роутер по питанию (не reboot  а именно отключить от электросети).
После того  как роутер перерезагрузился нужно установить пакет "containers". Для этого идём [сюда](https://mikrotik.com/download) и качаем пакет "Extra packages" для своей архитектуры. Извлекаем из архива пакет "**container-bla-bla.npk**".
Для более удобной передачи файлов между роутером и ПК можем включить на роутере SMB, подключаться можно из проводника windows \\\\<ip-of-mikrotik\>, юзер и пароль как указали ниже (рекомендую заменить пароль на более безопасный)

```
/ip/smb/set enabled=yes interfaces=bridge
/ip/smb/shares/add directory=usb1 name=flash
/ip/smb/users/add name=admin password=password
```

Или не включать и использовать SCP:

```
scp <path-to-local-file> admin@<mikrotik-ip>:/
```

Загружаем пакет на роутер тем или иным способом. Снова перезагружаем роутер. Если всё верно - после перезагрузки в интерфейсе winbox появится вкладка **Container**


## Настраиваем бридж и интерфейс для контейнеров

```
/interface bridge add name=Bridge-Docker port-cost-mode=short
/ip address add address=10.0.0.1/24 interface=Bridge-Docker network=10.0.0.0
/interface veth
add address=10.0.0.2/24 gateway=10.0.0.1 name=sing-box
/interface bridge port
add bridge=Bridge-Docker interface=sing-box
```
## Настройка хранилища для контейнеров

Проблемным местом в роутерах Mikrotik является малое количество ресурсов. В большнистве домашних моделей объем внутреннего хранилища составляет 128mb, чего недостаточно для хранения образов. Нам нужно дополнительное место чтобы все запустилось. 
##### Если на вашем роутере есть USB порт

В этом случае мы подключим флешку и будем использовать ее в качестве хранилища (оставим там ее навсегда). Вставляем флешку, смотрим ее название:

```
/disk/print
```

Форматируем флешку в ext4. Вместо usb1 подставляем наименование своей флешки, показанное в команде выше

```
/disk/format-drive file-system=ext4 usb1
```

##### Если USB порта нет

Чтобы данный способ был возможнен, на борту должно быть хотя бы 1Gb оперативной памяти. И да, как выяснилось, существует минимум один роутер, имеющий гиг оперативки и не имеющий USB порта - это свежий hap ax2. Я при покупке не обратил внимание на это и в итоге поимел проблем. Ну так вот. Mikrotik умеет создавать хранилище, выделяяя его прямо из оперативной памяти. Поскольку гигабайт ram обычно избыточен для домашнего роутера, мы вполне можем отрезать от него кусочек, в, скажем, 200Mb без каких-либо потерь. Стоит обратить внимание что 200Mb - это максимальное значение, которое может быть выделено при данной настройке, то есть фактичкски может быть занято от 0 и до 200 и не более

```
/disk/add type=tmpfs tmpfs-max-size=200M slot=tmpfs
```

После этого у нас появляется диск под названием tmpfs.  Во всей дальнейшей настройке следует, соответственно, вместо "usb1" использовать "tmpfs"

## Настройка самого контейнера

Создаем директорию для конфига
```
/file/add type=directory name=usb1/sing-box-config
```

Копируем конфиг в директорию. Можно сделать это через SMB, но я делаю через scp:
```
scp config.json admin@192.168.2.1:/usb1/sing-box-config/
```

Устанавливаем адрес registry, монтирование директории с конфигом, необходимые переменные среды и добавляем сам контейнер:

```
/container config set registry-url=https://ghcr.io tmpdir=/usb1/pull
/container/mounts/add src=/usb1/sing-box-config dst=/etc/sing-box name=sing-box

/container/envs/add name=sing-box key=devices value=/dev/net/tun
/container/envs/add name=sing-box key=network_mode value=host
/container/envs/add name=sing-box key=cap_add value=network_mode

/container/add remote-image=sagernet/sing-box:latest interface=sing-box cmd="run -D /etc/sing-box" root-dir=/usb1/sing-box start-on-boot=yes mounts=sing-box envlist=sing-box logging=yes
```

Запускаем контейнер. Number может отличаться если контейнер не единственный. Поэтому проверяем и потом запускаем:

```
/container/print
/container/start number=0
```

Если всё прошло успешно, у контейнера должен быть статус running. К нему можно подключиться и проверить всё ли ок. Если пинга нет - смотрим на правила firewall. Тут все может быть индивидуально в зависимости от установленной политики, разбирайтесь сами). Если пинг есть, ставим в контейнере curl и проверяем ip, с которым контейнер выходит в интернет. Если всё ок, это должен быть IP вашего сервера с xray
```
/container/shell 0
ping ya.ru
apk add curl
curl ident.me
```

## Маршрутизация трафика в тоннель

#### Руками через address-list
Теперь, когда контейнер поднят и работает, туда можно заворачивать трафик, который нуждается в обходе блокировок. Вариантов много, я использую простой - создаю address-list и заворачиваю в тоннель всё что идет в сторону адресов из списка. Также в список можно добавлять домены, а не IP адреса. Они будут динамически резолвиться.

```
/routing/table/add name=xray fib
/ip/route/add routing-table=xray gateway=10.0.0.2 check-gateway=ping dst-address=0.0.0.0/0
/ip/firewall/mangle/add chain=prerouting dst-address-list=xray_list action=mark-routing routing-mark=xray
```

В данном примере мы создали таблицу маршрутизации под названием xray, создали default route в этой таблице на наш контейнер и содали mangle правило, говорящее что всё что идет в сторону адресов из адрес-листа под названием xray_list направляется в эту таблицу.
Теперь чтобы завернуть в тоннель какой-либо ресурс, к примеру, rutracker.org, нам достаточно добавить домен в адрес лист. Это можно сделать через gui либо, как и всё в этой статье, через терминал:
```
/ip/firewall/address-list/add address=rutracker.org list=xray_list
```

#### Динамическое наполение address-list доменами

Поскольку в данный момент больной темой является блокировка ютуба, следует рассказать о том как его ускорить. К сожалению, способ "руками через address-list" не сработает, посколку ютуб использует динамические поддомены \*.googlevideo.com.
Очевидно, мы не можем просто добавить wildcard в address-list - это работать не будет, учите матчасть). Однако Mikrotik о нас позаботились и добавили замечательную функцию, позволяющую класть в определенный адрес-лист все зарезолвленные поддомены определенного домена. Либо делать это по регулярному выражению (это нам в данном случае не понадобится). Итак, магия:

```
/ip/dns/static/add type=FWD name=googlevideo.com match-subdomain=yes address-list=xray_list forward-to=8.8.8.8
```

После этого все поддомены, через которые работает ютуб видео, будут резолвиться и добавляться в наш лист xray_list, который, в свою очередь, заворачивается в тоннель. Более того, они будут динамически удаляться из этого листа в с TTL dns записи, то есть список не будет бесконечно разрастаться.

Однако этого мало для нормальной работы youtube, нужно завернуть в тоннель сам youtube.com и еще несколько доменов, которые нужно вручную добавить в наш лист xray_list. Не уверен что все они являются необходимыми, но в такой конфигурации точно работает:

```
/ip/firewall/address-list/add address=youtube.com list=xray_list
/ip/firewall/address-list/add address=www.youtube.com list=xray_list
/ip/firewall/address-list/add address=yt3.ggpht.com list=xray_list
/ip/firewall/address-list/add address=youtu.be list=xray_list
/ip/firewall/address-list/add address=i.ytimg.com list=xray_list
/ip/firewall/address-list/add address=accounts.youtube.com list=xray_list
```

## Related Notes
- [[гайды/Mikrotik BYEDPI tun2socks]]
- [[sing-box]]
- [[Configs/sing-box-config]]
- [[гайды/XRAY сервер установка]]