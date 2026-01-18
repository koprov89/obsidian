---
page-title: Решаем проблему блокировок (и YouTube) за 5 минут на роутере Mikrotik через контейнеры и без VPN / Хабр
url: https://habr.com/ru/articles/838452/
date: 2024-09-03 14:44:27
tags:
  - vpn
  - mikrotik
  - xray
  - container
---
В моем случае используется [[mikrotik]] hAP **ax3**. Стоит упомянуть, что подойдут только роутеры с архитектурой **ARM, ARM64** или **x86 (CHR)**, которые и поддерживают контейнеры.

А мне можно? Какая у меня архитектура?

![[7af8860251a32c701674b3e83e44ce15_MD5.png|"Место, куда нужно посмотреть и убедиться в подходящей архитектуре железа, прежде чем приступать к настройкам"]]

Место, куда нужно посмотреть и убедиться в подходящей архитектуре железа, прежде чем приступать к настройкам

![[dba47f6c7314c63a7dc95be10d8ee268_MD5.png]]

-   Прошивку желательно использовать 7.16, т.к. только в ней завезли DNS Forward запросы с использованием встроенного DoH.
    

Скоро версия выйдет в стабильном канале обновления, а пока в тестировании предрелизная версия на данный момент 7.16rc2.

Кто подходит под эти условия, велком под кат)

---

Для новичков добавил много скриншотов, т.к. не все ориентируются по командной строке.

-   Перед началом настройки рекомендую сделать бэкап конфигурации, чтобы быстро вернуться в рабочее состояние и начать заново при необходимости.
    

По настройке контейнеров есть [официальная вики](https://help.mikrotik.com/docs/display/ROS/Container), но постараюсь всё кратко описать здесь.

Если команда `/system/device-mode print` показывает `container: yes` , то все ок, если нет, то для включения режима контейнеров на устройстве нужно выполнить следующую команду и следовать инструкциям в консоли:

`/system/device-mode/update container=yes`

Скорее всего придется [скачать](https://mikrotik.com/download) и доустановить пакеты **Extra packages - Container** для вашей платформы.

Для установки контейнеров не рекомендуется использовать внутреннюю память, поэтому на роутерах нужна внешняя память USB это флешка или жесткий диск и желательно USB 3.0 и выше отформатированный в ext3/ext4.

Для удобного доступа к файлам контейнеров я расшарил флешку на бридже локальной сети, куда устанавливаются контейнеры (здесь и далее обращайте внимание на имя **usb** накопителя в вашей системе):

```
/ip smb set enabled=yes interfaces=bridge
/ip smb shares add directory=usb1 name=flash
```

-   По умолчанию для samba присутствует пользователь guest без пароля. Для безопасности создайте своего пользователя и пароль.
    

UI SMB настройки

![[b5366d33301710547997fae43409873d_MD5.png|"Пример настройки SMB"]]

Пример настройки SMB

Далее мы создаем бридж для контейнеров, задаем ему адрес, создаем 2 интерфейса VETH с адресами и добавляем их в наш бридж:

```
/interface bridge add name=Bridge-Docker port-cost-mode=short
/ip address add address=192.168.254.1/24 interface=Bridge-Docker network=192.168.254.0
/interface veth
add address=192.168.254.5/24 gateway=192.168.254.1 name=BYEDPI-SOCKS
add address=192.168.254.2/24 gateway=192.168.254.1 name=TUN2SOCKS
/interface bridge port
add bridge=Bridge-Docker interface=BYEDPI-SOCKS
add bridge=Bridge-Docker interface=TUN2SOCKS
```

UI настройки Bridge, VETH

![[c1549e728e024f9eae781cb9404b5b07_MD5.png|"настройки бриджа и VETH"]]

настройки бриджа и VETH

![[c0d1d67dfbae6303b7d760e3d5856f39_MD5.png]]

Установим URL-адрес реестра для загрузки контейнеров из реестра Docker и установим каталог извлечения tmpdir для подключенного usb носителя:

```
/container config set registry-url=https://registry-1.docker.io tmpdir=/usb1/docker/pull
```

UI Container config

![[c6666593e7154e65482cd819d45b013c_MD5.png|"настройки Container"]]

настройки Container

Скачиваем образы контейнеров, привязываем их к созданным интерфейсам VETH и ставим на автоматический запуск при загрузке устройства:

-   В контейнере byedpi при запуске используется набор команд в аргументе cmd, которые вам возможно придется подбирать для своего провайдера (у меня дом.ру), чтобы обходить DPI, справку по настройке параметров можно посмотреть на [гитхабе проекта](https://github.com/hufrea/byedpi).
    

```
/container/add remote-image=tazihad/byedpi:latest interface=BYEDPI-SOCKS cmd="--disorder 1 --auto=torst --tlsrec 1+s --debug 1" root-dir=/usb1/docker/byedpi start-on-boot=yes
/container/add remote-image=xjasonlyu/tun2socks:latest interface=TUN2SOCKS root-dir=usb1/docker/tun2socks start-on-boot=yes
```

Далее открываем файл `\192.168.88.1\flash\docker\tun2socks\entrypoint.sh`

-   Если вы будете искать файлы через проводник в WinBox и не увидите содержимое каталогов контейнеров, всему виной файлы пустышки с именем .type в каталогах, они скрывают их содержимое, чтобы не нагружать проводник файлов.
    

![[ae1358039a009974010b61e9d8cea6f4_MD5.png|"WinBox - File list"]]

WinBox - File list

и до запуска контейнера заменяем его содержимое этим (рекомендую использовать редактор [Notepad++](https://notepad-plus-plus.org/downloads/)):

Проводник файлов редактирование entrypoint.sh

![[ba83a29931eda0d6dc818c1f5282a9e1_MD5.png|"Проводник открытие файла entrypoint.sh по SMB для редактирования"]]

Проводник открытие файла entrypoint.sh по SMB для редактирования

```
#!/bin/sh
ip tuntap add mode tun dev tun0
ip addr add 198.18.0.1/15 dev tun0
ip link set dev tun0 up
ip route del default
ip route add default via 198.18.0.1 dev tun0 metric 1
ip route add default via 192.168.254.1 dev eth0 metric 100
ip route add 192.168.0.0/16 via 192.168.254.1 dev eth0
tun2socks -device tun0 -proxy socks5://192.168.254.5:1080 -interface eth0
```

-   Предпоследней командой мы настраиваем локальный маршрут через роутер к BYEDPI-SOCKS контейнеру, т.к. без этого все запросы на шлюз TUN2SOCKS будут уходить на более приоритетный маршрут tun0 с метрикой 1 внутрь туннеля, при этом коннекта с самим туннелем не будет.
    

Теперь можно запустить контейнеры. Проще всего это сделать через WinBox в окне `Container` есть кнопка start.

UI Containers start

![[22308f046807ae27e62beb2b1809f71d_MD5.png]]

И как минимум кого-то может устроить вариант использования только контейнера BYEDPI-SOCKS с указанием его IP адреса и порта `(192.168.254.5:1080)` в настройках или плагинах браузера, такой вариант удобно использовать сразу для проверки обхода DPI провайдера. Я использую браузер Мозила и плагин [FoxyProxy](https://addons.mozilla.org/ru/firefox/addon/foxyproxy-standard/).

-   Плагин настраивается достаточно просто, например, для ютуба можно экспортировать [настройки](https://archive.org/download/foxy-proxy-settings-byedpi-youtube/FoxyProxy_MIkrotik_bye_DPI.json), так же есть очень удобная фича в окне плагина "`Set tab proxy`", отлично подходит для проверок работоспособности сайта, чтобы не выискивать все используемые им хосты, а включить прокси только в текущей вкладке.
    

UI плагина FoxyProxy

![[33e8f969fa9317ba9f914c41343f8e4d_MD5.png|"Включение прокси на вкладку"]]

Включение прокси на вкладку

![[e2feb5ac40dd32e185c41b1499d62c95_MD5.png|"Настройки FoxyProxy"]]

Настройки FoxyProxy

## Сначала рассмотрим настройку без получаемых маршрутов для роутинга на наши контейнеры по BGP

Будем использовать DNS Forward и списки хостов, которые будем пускать через нашу карусель из контейнеров анти dpi системы :)

Для начала установим срок хранения попавших в спискок адресов до одного дня, чтобы они не удалялись сразу же как только заканчивается TTL у полученной DNS записи:

`/ip dns set address-list-extra-time=1d`

Обязательно должен быть настроен [DoH DNS](https://help.mikrotik.com/docs/pages/viewpage.action?pageId=83099652#DNS-DNSoverHTTPS\(DoH\)), ибо можно получить "подарок" от провайдера с липовыми адресами используя не шифрованные запросы DNS.

Затем добавим хосты, которые нужно пускать через контейнер анти DPI:

```
/ip/dns/static/
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=googlevideo.com type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=youtube.com type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=youtubei.googleapis.com type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=ytimg.com type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=youtu.be type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=ggpht.com type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=rutracker.org type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=rutracker.cc type=FWD
add address-list=za_dpi_FWD forward-to=localhost match-subdomain=yes name=medium.com type=FWD
```

-   Указанные домены с их поддоменами будут добавляться в список **za\_dpi\_FWD**, называется он так, чтобы быть в конце всех списков при сортировке по имени списка)
    

UI DNS static

![[8bb966e7b2698f5e7fae6e53fb8ef68b_MD5.png]]

Добавляем новую таблицу маршрутизации:

`/routing table add disabled=no fib name=dpi_mark`

И добавляем маршрут в эту таблицу на наш шлюз контейнер tun2socks:

`/ip route add disabled=no distance=22 dst-address=0.0.0.0/0 gateway=192.168.254.2%Bridge-Docker pref-src="" routing-table=dpi_mark scope=30 suppress-hw-offload=no target-scope=10`

Теперь добавим mangle правило, чтобы заворачивать все полученные в список хосты в таблицу маршрутизации dpi\_mark где весь трафик пойдет на наш анти dpi туннель:

```
/ip firewall mangle add action=mark-routing chain=prerouting comment="заворачивание списка хостов по DNS FWD на тунель tun2socks => byedpi" connection-state=""  dst-address-list=za_dpi_FWD in-interface-list=LAN new-routing-mark=dpi_mark passthrough=yes
```

UI таблица маршрутов и mangle mark route

Так же в правиле forward fasttrack connection нужно включить на вкладке general фильтр routing mark = main, так сайты из списков по манглу будут открываться быстрее.

![[2321a68469e109a4fb0ddeffe2b651c3_MD5.png|"Настройки правила №14 fasttrack connection"]]

Настройки правила №14 fasttrack connection

Теперь чтобы это все работало корректно, нужно все запросы DNS отправлять на роутер, для этого нужно отключить в браузере запросы DoH ([например в мозилле](https://support.mozilla.org/ru/kb/nastrojka-urovnej-zashity-dns-cherez-https-v-firef)) и добавить перехват всех запросов DNS из локальной сети:

```
/ip firewall nat
add action=redirect chain=dstnat dst-port=53 in-interface-list=LAN protocol=udp
add action=redirect chain=dstnat dst-port=53 in-interface-list=LAN protocol=tcp
```

UI Firewall NAT

![[8cf7cfc90165d464ce67085b015a3842_MD5.png]]

Всё! Можно начинать тестировать ютуб и сайты из добавленного списка, но с одним замечанием, нужно либо перезагрузить все устройства, что иногда сделать проще всего, чтобы на них очистился кеш DNS запросов, либо очистить его вручную и на роутере в том числе. Иначе необходимые адреса не попадут в списки для роутинга.

Все это прекрасно совмещается с аналогичным выходом на vpn ресурсов по другому списку хостов в DNS FWD, которые не обходятся таким способом, но здесь не об этом.

## Теперь рассмотрим настройку с получаемыми маршрутами по BGP в т.ч. для Ютуба

Допустим мы уже получаем какие-то маршруты по BGP от сервисов [antifilter](https://antifilter.network/bgp) и дополнительно хотим получать адреса для Ютуба чтобы сразу направить их на наши контейнеры.

UI BGP подключение

![[99380b58f80f83b3064e93d291436cfa_MD5.png|"Основные настройки BGP"]]

Основные настройки BGP

![[cbac2d4d7b7bb54a3c13735f1f961df6_MD5.png|"Фильтр bgp_in, в который далее будем вносить маршруты."]]

Фильтр bgp\_in, в который далее будем вносить маршруты.

![[4c052e7ac52158060ad5ca150b47c963_MD5.png|"Включаем этот список на сайте antifilter для получения на роутер."]]

Включаем этот список на сайте [antifilter](https://antifilter.network/bgp) для получения на роутер.

В таком случае нужно ко всем вышеперечисленным настройкам создать новую таблицу маршрутизации clear\_out

`/routing table add disabled=no fib name=clear_out`

С маршрутом в интернет для этой таблицы, который у вас есть по дефолту, у меня это выше стоящий GPON роутер 192.168.1.100:

`/ip route add disabled=no distance=24 dst-address=0.0.0.0/0 gateway=192.168.1.100 pref-src="" routing-table=clear_out scope=30 suppress-hw-offload=no target-scope=10`

UI таблица маршрутов

![[afd20b6d2710ae84c1eda2df67c5a1f1_MD5.png]]

![[1703d4106e727198205c4878e6b6adf3_MD5.png|"По сути нужно скопировать основной маршрут в интернет без указания VRF и с указанием новой таблицы роутинга"]]

По сути нужно скопировать основной маршрут в интернет без указания VRF и с указанием новой таблицы роутинга

Далее в фильтрах роутинга **/routing/filter** нужно для Ютуба прописать наш шлюз TUN2SOCKS:

`if (bgp-communities includes 65444:770) {set gw 192.168.254.2%Bridge-Docker; accept;} else {set gw 10.10.0.1%wireguard-client-vpn; accept;}`

UI фильтры маршрутов

![[b345a2a482aa74d27d93dc46a7095f13_MD5.png|"Вносим в тот фильтр chain, который выбран в настройках BGP - Filter - Input filter"]]

Вносим в тот фильтр chain, который выбран в настройках BGP - Filter - Input filter

Скорее всего у большинства по bgp приходят роуты в `main routing table`, а теперь после выполнения последней команды есть роуты и на шлюз контейнера `tun2socks`

UI BGP и Route list

![[e5b76a68bbf1a5f2505e9d6c24a0ae7e_MD5.png|"По bgp приходят роуты в routing table main "]]

По bgp приходят роуты в `routing table` `main`

![[db95e3d3ebddc74961e500b5cece1323_MD5.png|"Полученные по bgp маршруты для Ютуба"]]

Полученные по bgp маршруты для Ютуба

в таком случае сам socks контейнер `byedpi` выводим в отдельную таблицу маршрутизации `clear_out` которая выходит прямо в интернет, чтобы не зациклить маршрутизацию адресов ютуба и всего что отправляется на шлюз контейнера `tun2socks` в таблице `main`, т.к. ничего не будет работать.

/`ip firewall mangle add action=mark-routing chain=prerouting comment="BYEDPI-SOCKS mark route to clear_out" connection-state="" new-routing-mark=clear_out passthrough=yes src-address=192.168.254.5`

UI mangle mark routing

![[abfdadbe04f14fce441766e2eefe3320_MD5.png|"По счетчикам видно срабатывание правила. Стоит приоритетнее, чем правило по спискам."]]

По счетчикам видно срабатывание правила. Стоит приоритетнее, чем правило по спискам.

Это правило отлично подходит для проверки работоспособности, т.к. есть счетчики на правилах. Либо же его можно заменить аналогичным другим правилом в другом разделе:

`/routing rule add action=lookup comment="BYEDPI-SOCKS mark route to clear_out" disabled=no src-address=192.168.254.5/32 table=clear_out`

UI routing rules

![[95d5b2ce831341cc195c87e99c40a476_MD5.png]]

Выбирайте на вкус и цвет, есть ли какая-то разница в производительности, не знаю. Иногда лучше видеть всё в настройках `mangle`, в одном месте так сказать.

-   После этой настройки наш контейнер `BYEDPI-SOCKS (192.168.254.5:1080)` перестает быть напрямую доступным из локальной сети, поэтому использовать его из браузера как socks5 прокси становится невозможным. Может кто-то знает как это решить, пишите в комментариях.
    

### Заключение

Большинство опытных пользователей получают маршруты по BGP и просто всё отправляют на VPN, простые пользователи борятся с проблемами доступа к заблокированным ресурсам разными бесплатными VPN'ами, вторых можно понять, да и первых, у кого старые роутеры без поддержки контейнеров.

После погружения в тему для меня стало удивлением, что много сайтов начали работать без VPN, поэтому меня привлекла логичная идея локально "лечить" трафик прямо на роутере.

После настройки этих двух контейнеров я даже отказался от BGP, потому что не так много сайтов я использую которые не доступны, все они прекрасно работают по спискам DNS FWD, тестирую уже неделю и полёт отличный.

В целом меня раззадорил спортивный интерес и, конечно же, проблемы с ютубом, которые на компе решались очень просто, но хотелось красивого решения на роутере, не зря же была куплена свежая железка с поддержкой контейнеров, после знакомства с которыми пару лет назад я понял, какой потенциал будет на будущее у моего роутера.

Спасибо всем за внимание!

P.S. т.к. пишу здесь первый раз, готов к вашим замечаниям и конструктивным комментариям и благодарю за понимание))


Если эта публикация вас вдохновила и вы хотите поддержать автора — не стесняйтесь нажать на кнопку

## Related Notes
- [[Моё/Mikrotik + SingBox с VLess]]
- [[openwrt_vpn]]