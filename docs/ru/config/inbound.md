# Входящие подключения

Входящие подключения используются для приема данных. Доступные протоколы см. в разделе [Входящие протоколы](./inbounds/).

## InboundObject

`InboundObject` соответствует дочернему элементу поля `inbounds` в конфигурационном файле.

```json
{
  "inbounds": [
    {
      "listen": "127.0.0.1",
      "port": 1080,
      "protocol": "название протокола",
      "settings": {},
      "streamSettings": {},
      "tag": "тег",
      "sniffing": {
        "enabled": true,
        "destOverride": ["http", "tls"]
      },
      "allocate": {
        "strategy": "always",
        "refresh": 5,
        "concurrency": 3
      }
    }
  ]
}
```

> `listen`: address

Адрес прослушивания, IP-адрес или Unix domain socket.  
Значение по умолчанию - `"0.0.0.0"`, что означает прием подключений на всех сетевых интерфейсах.

Можно указать любой доступный в системе IP-адрес.

`"::"` эквивалентно `"0.0.0.0"` — обе записи позволяют одновременно прослушивать IPv6 и IPv4. Однако, если вы хотите прослушивать только IPv6, установите параметр `v6only` в `sockopt` равным `true`. Если нужно прослушивать только IPv4, выполните команду `ip a`, чтобы узнать конкретный IP-адрес сетевого интерфейса (это может быть внешний IP-адрес машины или внутренний адрес, например 10.x.x.x), и настройте прослушивание на этот адрес. Аналогичный подход применяется и для IPv6.

Поддерживается указание Unix domain socket в формате абсолютного пути, например `"/dev/shm/domain.socket"`.  
Можно добавить `@` в начало пути, чтобы использовать [абстрактный сокет](https://www.man7.org/linux/man-pages/man7/unix.7.html), или `@@`, чтобы использовать абстрактный сокет с заполнением.

При указании Unix domain socket параметры `port` и `allocate` игнорируются.  
В настоящее время поддерживаются протоколы VLESS, VMess, Trojan и типы транспорта TCP, WebSocket, HTTP/2, gRPC.

При указании Unix domain socket можно указать права доступа к сокету, добавив запятую и индикатор прав доступа, например `"/dev/shm/domain.socket,0666"`.  
Это может помочь решить проблемы с правами доступа к сокету, которые возникают по умолчанию.

> `port`: number | "env:variable" | string

Порт.  
Допустимые форматы:

- Целое число: фактический номер порта.
- Переменная окружения: начинается с `"env:"`, за которым следует имя переменной окружения, например `"env:PORT"`.  
   Xray будет анализировать эту переменную окружения как строку.
- Строка: может быть числом в виде строки, например `"1234"`, или диапазоном портов, например `"5-10"`, что означает порты с 5 по 10 (6 портов).  
   Можно использовать запятые для разделения диапазонов, например `11,13,15-17`, что означает порты 11, 13, 15, 16 и 17 (5 портов).

Если указан только один порт, Xray будет прослушивать входящие подключения на этом порту.  
Если указан диапазон портов, то фактическое поведение зависит от настройки `allocate`.

Обратите внимание, что прослушивание порта — это довольно ресурсоемкая операция. Прослушивание слишком большого диапазона портов может привести к значительному увеличению потребляемых ресурсов и даже нарушить нормальную работу Xray. Как правило, проблемы могут начаться, когда количество прослушиваемых портов приближается к четырехзначным числам. Если вам нужен большой диапазон, рассмотрите возможность использования iptables для перенаправления вместо того, чтобы настраивать его здесь.

> `protocol`: "dokodemo-door" | "http" | "shadowsocks" | "mixed" | "vless" | "vmess" | "trojan" | "wireguard"

Название протокола подключения.  
Список доступных протоколов см. в разделе "Входящие подключения" в левой части документации.

> `settings`: InboundConfigurationObject

Конкретные настройки зависят от протокола.  
См. описание `InboundConfigurationObject` для каждого протокола.

> `streamSettings`: [StreamSettingsObject](./transport.md#streamsettingsobject)

Тип транспорта (transport) - это способ взаимодействия текущего узла Xray с другими узлами.

> `tag`: string

Тег этого входящего подключения, используемый для идентификации этого подключения в других настройках.

::: danger
Если это поле не пустое, его значение должно быть **уникальным** среди всех тегов.
:::

> `sniffing`: [SniffingObject](#sniffingobject)

Обнаружение трафика в основном используется для прозрачного проксирования и других целей.  
Например, типичный сценарий выглядит следующим образом:

1. Устройство пытается получить доступ к abc.com.  
   Сначала устройство выполняет DNS-запрос и получает IP-адрес 1.2.3.4 для abc.com.  
   Затем устройство пытается установить соединение с 1.2.3.4.
2. Если обнаружение трафика не настроено, Xray получает запрос на подключение к 1.2.3.4 и не может использовать доменные правила для маршрутизации и разделения трафика.
3. Если в sniffing включен параметр `enabled`, Xray при обработке трафика этого соединения попытается извлечь доменное имя из данных трафика, т.е. abc.com.
4. Xray заменит 1.2.3.4 на abc.com.  
   Маршрутизация сможет использовать доменные правила для разделения трафика.

Так как запрос теперь направляется на abc.com, можно выполнять больше действий, например, повторное разрешение DNS, помимо разделения трафика по доменным правилам.

Если в sniffing включен параметр `enabled`, Xray также сможет обнаруживать трафик типа bittorrent, а затем можно настроить правила маршрутизации по протоколу, чтобы обрабатывать трафик BT, например, блокировать его на сервере или перенаправлять его на определенный VPS на клиенте.

> `allocate`: [AllocateObject](#allocateobject)

Настройки выделения портов при указании нескольких портов.

### SniffingObject

```json
{
  "enabled": true,
  "destOverride": ["http", "tls", "fakedns"],
  "metadataOnly": false,
  "domainsExcluded": [],
  "routeOnly": false
}
```

> `enabled`: true | false

Включить обнаружение трафика.

> `destOverride`: \["http" | "tls" | "quic" | "fakedns"\]

Заменить целевой адрес текущего подключения на указанные типы, если трафик соответствует им.

::: tip
Xray будет использовать доменные имена, обнаруженные с помощью sniffing, только для маршрутизации.  
Если вы хотите только обнаруживать доменные имена для маршрутизации, но не хотите изменять целевой адрес (например, при использовании Tor Browser изменение целевого адреса может привести к невозможности подключения), добавьте соответствующие протоколы в этот список и включите `routeOnly`.
:::

> `metadataOnly`: true | false

Если этот параметр включен, для обнаружения целевого адреса будут использоваться только метаданные подключения.  
В этом случае все снифферы, кроме `fakedns`, будут отключены.

Если этот параметр отключен, для определения целевого адреса будут использоваться не только метаданные, но и данные.  
В этом случае клиенту необходимо сначала отправить данные, чтобы прокси-сервер установил соединение.  
Это поведение несовместимо с протоколами, которые требуют, чтобы сервер первым отправил сообщение, например, SMTP.

> `domainsExcluded`: [string] <Badge text="WIP" type="warning"/>

Список доменных имен, для которых **не будет** выполняться замена целевого адреса, если они обнаружены с помощью sniffing.

Поддерживаются прямые доменные имена (точное совпадение) или регулярные выражения, начинающиеся с `regexp:`.

::: tip
Добавление некоторых доменных имен может решить проблемы с push-уведомлениями iOS, умными устройствами Mijia и голосовым чатом в некоторых играх (Rainbow Six Siege).

Если вам нужно выяснить причину каких-либо проблем, попробуйте отключить `"sniffing"` или включить `"routeOnly"`.
:::

```json
"domainsExcluded": [
    "courier.push.apple.com", // Push-уведомления iOS
    "Mijia Cloud", // Умные устройства Mijia
    "dlg.io.mi.com"
]

```

::: warning
В настоящее время `domainsExcluded` не поддерживает способы сопоставления доменов, аналогичные тем, что используются в маршрутизации.  
Этот параметр может быть изменен в будущем, совместимость между версиями не гарантируется.
:::

> `routeOnly`: true | false

Использовать обнаруженные доменные имена только для маршрутизации.  
Целевой адрес прокси-сервера остается IP-адресом.  
Значение по умолчанию - `false`.

Этот параметр требует, чтобы `destOverride` был включен.

::: tip
Если вы уверены, что **проксируемое соединение будет правильно разрешено DNS**, то при использовании `routeOnly` и включенном `destOverride` можно установить стратегию сопоставления маршрутов `domainStrategy` в `AsIs`, чтобы реализовать разделение трафика по доменам и IP-адресам без DNS-разрешения.  
В этом случае при сопоставлении правил на основе IP-адресов будет использоваться исходный IP-адрес домена.
:::

### AllocateObject

```json
{
  "strategy": "always",
  "refresh": 5,
  "concurrency": 3
}
```

> `strategy`: "always" | "random"

Стратегия выделения портов.

- `"always"` - всегда выделять все указанные порты.  
   Xray будет прослушивать все порты, указанные в `port`.
- `"random"` - случайным образом открывать порты.  
   Каждые `refresh` минут Xray будет случайным образом выбирать `concurrency` портов из диапазона, указанного в `port`, и прослушивать их.

> `refresh`: number

Интервал обновления случайных портов в минутах.  
Минимальное значение - `2`, рекомендуемое значение - `5`.  
Этот параметр действителен только при `strategy` = `"random"`.

> `concurrency`: number

Количество случайных портов.  
Минимальное значение - `1`, максимальное значение - треть от диапазона портов, указанного в `port`.  
Рекомендуемое значение - `3`.
