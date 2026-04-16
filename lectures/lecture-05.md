---

# lecture-05: Работа с сетью и основы backend-разработки

---

<style>
.two-cols-header {
  column-gap: 20px; /* Adjust the gap size as needed */
}
</style>


<img src="https://i.redd.it/ivzwwzt2p0m61.png" width="650" />

---

# Зачем приложениям сеть?

---
layout: two-cols-header
---

## Почти любое приложение — сетевое

::left::

Сеть нужна, когда приложение хочет общаться с **внешним миром**:

- загрузить данные с сервера
- отправить форму регистрации
- получить список товаров
- синхронизировать сообщения
- обновить статус заказа
- загрузить файл
- авторизовать пользователя

::right::

```text
┌─────────────┐      internet       ┌─────────────┐
│ Mobile App  │ ──────────────────► │   Server    │
└─────────────┘                     └─────────────┘
       ▲                                   │
       │                                   ▼
       │                            ┌─────────────┐
       └─────────────────────────── │  Database   │
                                    └─────────────┘
```

> Если данные живут не только на вашем устройстве — вам нужна сеть.

---

## Примеры сетевых сценариев

| Приложение | Что делает по сети |
|---|---|
| Telegram | отправляет сообщения, получает новые события |
| YouTube | загружает видео и рекомендации |
| Wildberries / Ozon | получает каталог, создаёт заказ, отслеживает доставку |
| Banking app | отправляет платежи, получает баланс |
| Online game | синхронизирует состояние игроков |
| IDE | скачивает зависимости, плагины, обновления |

---
layout: two-cols-header
---

## Сеть — это медленно и ненадёжно

::left::

Вызов функции внутри процесса почти мгновенный.

Сетевой вызов зависит от:

- качества соединения
- расстояния до сервера
- нагрузки на сервер
- DNS
- TLS
- прокси и балансировщиков
- потерь пакетов

::right::

```text
Обычный вызов функции:

main() ──► calculatePrice()
       <── result

Сетевой вызов:

client ──► DNS ──► TCP ──► TLS ──► server ──► DB
       <──────────────────────────────────── result
```

> Главное правило: сеть может быть медленной, недоступной или вернуть ошибку.

---

## Fallacies of Distributed Computing

Классические заблуждения при работе с сетью:

1. Сеть надёжна
2. Задержка равна нулю
3. Пропускная способность бесконечна
4. Сеть безопасна
5. Топология сети не меняется
6. Есть один администратор
7. Транспортные расходы равны нулю
8. Сеть однородна

> Хороший сетевой код всегда предполагает: **что-то может пойти не так**.

---

# Базовые понятия

---
layout: two-cols-header
---

## IP-адрес

::left::

**IP-адрес** — адрес устройства в сети.

Примеры:

```text
127.0.0.1        localhost
192.168.1.42     устройство в локальной сети
8.8.8.8          публичный DNS Google
142.250.185.14   один из IP Google
```

- IPv4: `192.168.1.42`
- IPv6: `2a00:1450:4001:82a::200e`
- `localhost` / `127.0.0.1` — текущий компьютер

::right::

```text
┌──────────────┐
│ Your laptop  │
│ 192.168.1.10 │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Wi-Fi router │
│ 192.168.1.1  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│ Remote server│
│ 93.184.216.34│
└──────────────┘
```

---

## Порт

Если IP — это адрес здания, то **порт** — номер квартиры.

```text
server.com:80      HTTP
server.com:443     HTTPS
localhost:8080     типичный порт dev-сервера
localhost:5432     PostgreSQL
localhost:6379     Redis
```

```text
┌─────────────────────────────┐
│ Server: 93.184.216.34       │
│                             │
│  :80    ──► HTTP server     │
│  :443   ──► HTTPS server    │
│  :5432  ──► PostgreSQL      │
│  :6379  ──► Redis           │
└─────────────────────────────┘
```

> Один сервер может запускать много сетевых приложений на разных портах.

---

## Socket

**Socket** — точка сетевого соединения: IP + порт + протокол.

```text
Client socket                       Server socket
192.168.1.10:53124  ─────────────►  93.184.216.34:443
```

Для TCP-соединения важны обе стороны:

```text
┌─────────────────────────────────────────────────────┐
│ TCP connection                                      │
├──────────────────────┬──────────────────────────────┤
│ client IP            │ 192.168.1.10                 │
│ client port          │ 53124                        │
│ server IP            │ 93.184.216.34                │
│ server port          │ 443                          │
│ protocol             │ TCP                          │
└──────────────────────┴──────────────────────────────┘
```

> Клиентский порт обычно временный: ОС выдаёт его автоматически.

---

## DNS

Людям удобно помнить домены:

```text
google.com
github.com
kotlinlang.org
```

Компьютерам нужны IP-адреса:

```text
142.250.185.14
140.82.121.4
```

**DNS** переводит доменное имя в IP-адрес.

---

<img src="https://assets.bytebytego.com/diagrams/0176-dns-look-up.png" width="1040" />

---

## Что происходит при открытии сайта?

```text
Пользователь вводит: https://example.com/products

┌─────────┐
│Browser  │
└────┬────┘
     │ 1. DNS: example.com -> IP
     ▼
┌─────────┐
│ DNS     │
└────┬────┘
     │ 2. TCP connection to IP:443
     ▼
┌─────────┐
│ Server  │
└────┬────┘
     │ 3. TLS handshake
     │ 4. HTTP request: GET /products
     │ 5. HTTP response: HTML/JSON
     ▼
┌─────────┐
│Browser  │ renders result
└─────────┘
```

---

# Модель TCP/IP

---
layout: two-cols-header
---

## Зачем нужна модель уровней?

::left::

Сеть сложная, поэтому её делят на **уровни**.

Каждый уровень решает свою задачу:

- приложение формирует смысл сообщения
- транспорт доставляет данные между процессами
- IP доставляет пакеты между устройствами
- Wi‑Fi/Ethernet передают биты в конкретной сети

::right::

```text
┌─────────────────────────────────────┐
│ Application                         │
│ HTTP / DNS / WebSocket / gRPC       │
├─────────────────────────────────────┤
│ Transport                           │
│ TCP / UDP                           │
├─────────────────────────────────────┤
│ Internet                            │
│ IP                                  │
├─────────────────────────────────────┤
│ Link                                │
│ Ethernet / Wi‑Fi                    │
└─────────────────────────────────────┘
```

> Мы пишем HTTP-код, но под ним работают ещё несколько уровней.

---

# Модель OSI

---
layout: two-cols-header
---

## OSI: 7 уровней сети

::left::

**OSI** — учебная модель, которая помогает разложить сетевое взаимодействие по слоям.

```text
┌─────────────────────────────────────┐
│ 7. Application                      │
├─────────────────────────────────────┤
│ 6. Presentation                     │
├─────────────────────────────────────┤
│ 5. Session                          │
├─────────────────────────────────────┤
│ 4. Transport                        │
├─────────────────────────────────────┤
│ 3. Network                          │
├─────────────────────────────────────┤
│ 2. Data Link                        │
├─────────────────────────────────────┤
│ 1. Physical                         │
└─────────────────────────────────────┘
```

::right::

| Уровень | За что отвечает |
|---|---|
| Application | протоколы приложений |
| Presentation | формат, кодирование, шифрование |
| Session | управление сессией |
| Transport | доставка между процессами |
| Network | маршрутизация между сетями |
| Data Link | передача внутри локальной сети |
| Physical | сигнал: кабель, радио |

---

## OSI на примерах

| OSI layer | Примеры |
|---|---|
| 7. Application | HTTP, DNS, SMTP, WebSocket, gRPC |
| 6. Presentation | TLS, UTF-8, JSON, Protobuf |
| 5. Session | логическая сессия, reconnect, keep-alive |
| 4. Transport | TCP, UDP, ports |
| 3. Network | IP, routing |
| 2. Data Link | Ethernet, Wi‑Fi, MAC address |
| 1. Physical | витая пара, оптика, радиосигнал |

> В реальной разработке границы уровней иногда размыты, но модель полезна для мышления.

---
layout: two-cols-header
---

## OSI vs TCP/IP

::left::

OSI подробнее и академичнее:

```text
7. Application   ┐
6. Presentation  ├──► Application
5. Session       ┘

4. Transport     ───► Transport

3. Network       ───► Internet

2. Data Link     ┐
1. Physical      ┘──► Link
```

::right::

TCP/IP практичнее:

```text
┌─────────────────────────┐
│ Application             │
│ HTTP / DNS / gRPC       │
├─────────────────────────┤
│ Transport               │
│ TCP / UDP               │
├─────────────────────────┤
│ Internet                │
│ IP                      │
├─────────────────────────┤
│ Link                    │
│ Ethernet / Wi‑Fi        │
└─────────────────────────┘
```

> OSI — хорошая карта. TCP/IP — то, с чем чаще работают на практике.

---

## Диагностика проблем по уровням

| Симптом | Где искать проблему |
|---|---|
| Нет Wi‑Fi | Physical / Data Link |
| Не получается получить IP | Data Link / Network |
| Не пингуется сервер | Network |
| `Connection refused` | Transport / Application |
| TLS certificate expired | Presentation / Application |
| `401 Unauthorized` | Application |
| `404 Not Found` | Application |
| Медленно грузится файл | Network / Transport / Application |

```text
Чем ниже уровень проблемы, тем меньше смысла дебажить код приложения.
```

---

## Модель TCP/IP на примере HTTP

```text
GET /products HTTP/1.1
Host: example.com
```

Проходит вниз по стеку:

```text
┌─────────────────────────────────────────────┐
│ Application: HTTP request                   │
│ "GET /products"                             │
└───────────────────┬─────────────────────────┘
                    ▼
┌─────────────────────────────────────────────┐
│ Transport: TCP segment                      │
│ source port -> destination port             │
└───────────────────┬─────────────────────────┘
                    ▼
┌─────────────────────────────────────────────┐
│ Internet: IP packet                         │
│ source IP -> destination IP                 │
└───────────────────┬─────────────────────────┘
                    ▼
┌─────────────────────────────────────────────┐
│ Link: Ethernet/Wi‑Fi frame                  │
│ передача внутри конкретной сети             │
└─────────────────────────────────────────────┘
```

---
layout: two-cols-header
---

## Инкапсуляция

::left::

Каждый уровень добавляет свои служебные данные — **header**.

```text
HTTP data
  ↓ добавить TCP header
TCP segment
  ↓ добавить IP header
IP packet
  ↓ добавить Ethernet/Wi‑Fi header
Frame
```

На другой стороне процесс идёт обратно: headers снимаются слой за слоем.

::right::

```text
Отправка:

[ HTTP data ]
[ TCP header | HTTP data ]
[ IP header  | TCP header | HTTP data ]
[ Link header| IP header  | TCP header | HTTP data ]

Получение:

Link снимает Link header
IP снимает IP header
TCP снимает TCP header
HTTP получает исходные данные
```

---

## Кто за что отвечает?

| Уровень | Вопрос | Примеры |
|---|---|---|
| Application | Что означает сообщение? | HTTP, DNS, gRPC |
| Transport | Какому процессу доставить? | TCP, UDP, ports |
| Internet | На какое устройство доставить? | IP |
| Link | Как передать в этой сети? | Ethernet, Wi‑Fi |

```text
URL: https://example.com:443/products
              │          │
              │          └─ port -> Transport layer
              └──────────── domain -> DNS -> IP -> Internet layer
```

> Endpoint в приложении — это верхушка айсберга. Ниже работают транспорт, IP и физическая сеть.

---
layout: two-cols-header
---

## TCP vs UDP

::left::

### TCP

- соединение перед передачей данных
- гарантирует доставку
- сохраняет порядок сообщений
- контролирует перегрузку
- медленнее, но надёжнее

Используется в:

- HTTP/HTTPS
- SSH
- PostgreSQL
- SMTP

::right::

### UDP

- без установки соединения
- не гарантирует доставку
- не гарантирует порядок
- меньше накладных расходов
- быстрее, но проще

Используется в:

- DNS
- VoIP
- online games
- video streaming

---

## TCP как телефонный звонок, UDP как открытка

```text
TCP:

A: Привет, слышишь меня?
B: Да, слышу.
A: Отправляю данные #1.
B: Получил #1.
A: Отправляю данные #2.
B: Получил #2.
```

```text
UDP:

A: Данные #1 ─────────────► B
A: Данные #2 ─────X         B  // потерялось
A: Данные #3 ─────────────► B
```

> TCP выбирают, когда важна корректность. UDP — когда важнее скорость и можно пережить потерю части данных.

---
layout: two-cols-header
---

## Latency и bandwidth

::left::

**Latency** — задержка до первого ответа.

```text
client ───── request ─────► server
client ◄──── response ───── server

latency = сколько времени прошло
```

Примеры:

- Москва → Москва: условно быстро
- Москва → Европа: дольше
- Москва → США: ещё дольше

::right::

**Bandwidth** — пропускная способность канала.

```text
Низкий bandwidth:
[█░░░░░░░░░]  медленная загрузка файла

Высокий bandwidth:
[██████████]  быстрая загрузка файла
```

> Latency особенно важна для большого количества маленьких запросов.

---

## Timeout

**Timeout** — максимальное время ожидания операции.

```kotlin
val client = HttpClient(CIO) {
    install(HttpTimeout) {
        requestTimeoutMillis = 5_000
        connectTimeoutMillis = 2_000
        socketTimeoutMillis = 5_000
    }
}
```

Без timeout приложение может зависнуть надолго:

```text
client ───── request ─────► server
       ◄──── нет ответа ───
       waiting...
       waiting...
       waiting forever?
```

> Для сетевого кода timeout — не опция, а обязательная часть дизайна.

---

# Клиент-серверная архитектура

---
layout: two-cols-header
---

## Клиент и сервер

::left::

**Клиент** — тот, кто инициирует запрос.

Примеры клиентов:

- браузер
- мобильное приложение
- desktop-приложение
- другой backend-сервис
- CLI-утилита
- тестовый HTTP-клиент

::right::

**Сервер** — тот, кто принимает запрос и возвращает ответ.

```text
┌─────────────┐     request      ┌─────────────┐
│   Client    │ ───────────────► │   Server    │
└─────────────┘                  └──────┬──────┘
       ▲                                │
       │            response            ▼
       └──────────────────────── ┌─────────────┐
                                 │  Database   │
                                 └─────────────┘
```

> Роль зависит от ситуации: backend может быть сервером для frontend и клиентом для базы или другого API.

---

## Request / Response

Классическая HTTP-коммуникация устроена как пара:

```text
1. Client отправляет request
2. Server обрабатывает request
3. Server возвращает response
4. Client обрабатывает response
```

```text
┌──────────────┐                         ┌──────────────┐
│   Browser    │                         │  Web Server  │
└──────┬───────┘                         └──────┬───────┘
       │                                        │
       │ GET /products                          │
       │───────────────────────────────────────►│
       │                                        │ ищет товары
       │ 200 OK + JSON                          │
       │◄───────────────────────────────────────│
       │                                        │
       ▼                                        ▼
 показывает пользователю                 ждёт следующий запрос
```

---
layout: two-cols-header
---

## Что находится в request?

::left::

Request описывает, **что клиент хочет сделать**.

Обычно внутри есть:

- HTTP method
- URL / endpoint
- headers
- query parameters
- path parameters
- body

::right::

```http
POST /api/v1/users?sendEmail=true HTTP/1.1
Host: example.com
Content-Type: application/json
Authorization: Bearer <token>

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

> Request — это не просто URL. Важны method, headers и body.

---
layout: two-cols-header
---

## Что находится в response?

::left::

Response описывает, **чем закончилась обработка запроса**.

Обычно внутри есть:

- status code
- headers
- body

Статус говорит клиенту, что произошло:

- успех
- ошибка клиента
- ошибка сервера
- redirect

::right::

```http
HTTP/1.1 201 Created
Content-Type: application/json
Location: /api/v1/users/42

{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com"
}
```

> Клиент должен смотреть не только на body, но и на status code.

---

## Backend, frontend, mobile client

```text
┌─────────────────┐
│ Browser UI      │  frontend
│ React/Vue/etc.  │
└────────┬────────┘
         │ HTTP
         ▼
┌─────────────────┐
│ Backend API     │  Kotlin / Ktor / Spring
│ business logic  │
└────────┬────────┘
         │ SQL / protocol
         ▼
┌─────────────────┐
│ Database        │  PostgreSQL / MongoDB
│ persistent data │
└─────────────────┘
```

```text
Mobile App ───────► тот же Backend API
Another service ──► тот же Backend API
CLI tool ─────────► тот же Backend API
```

> API позволяет разным клиентам использовать одну и ту же бизнес-логику.

---
layout: two-cols-header
---

## Stateless-сервер

::left::

**Stateless** означает: сервер не хранит состояние конкретного клиента между запросами.

Каждый request должен содержать всё необходимое:

- кто пользователь
- что он хочет сделать
- какие параметры нужны
- токен авторизации

::right::

```text
Request #1:
GET /profile
Authorization: Bearer token-123

Request #2:
GET /orders
Authorization: Bearer token-123

Request #3:
POST /orders
Authorization: Bearer token-123
```

Сервер не должен думать:

```text
"А, это тот самый клиент из прошлого запроса"
```

---

## Почему stateless — это удобно?

```text
             ┌──────────────┐
Request #1 ─►│ Backend #1   │
             └──────────────┘

             ┌──────────────┐
Request #2 ─►│ Backend #2   │
             └──────────────┘

             ┌──────────────┐
Request #3 ─►│ Backend #3   │
             └──────────────┘
```

Если серверы stateless, запросы можно отправлять на любой экземпляр backend-а.

Это помогает:

- масштабировать приложение
- перезапускать серверы
- балансировать нагрузку
- проще восстанавливаться после падений

> Состояние лучше хранить в базе, кэше или токене, а не в памяти одного backend-сервера.

---
layout: two-cols-header
---

## Reverse proxy

::left::

**Reverse proxy** стоит перед backend-ом и принимает внешние запросы.

Типичные задачи:

- принимать HTTPS
- проксировать запросы во внутреннюю сеть
- отдавать static files
- сжимать ответы
- ограничивать размер request body
- скрывать реальные backend-серверы

Примеры: **nginx**, **Caddy**, **Traefik**.

::right::

```text
Internet
   │
   ▼
┌───────────────┐
│ Reverse proxy │  :443 HTTPS
│ nginx         │
└───────┬───────┘
        │ internal HTTP
        ▼
┌───────────────┐
│ Kotlin app    │  :8080
│ Ktor/Spring   │
└───────────────┘
```

---

## Forward proxy vs Reverse proxy

<img src="https://www.indusface.com/wp-content/uploads/2023/04/Forward-proxy-vs-reverse-proxy-1.png" width="760" />

> Forward proxy скрывает клиента от сервера. Reverse proxy скрывает серверы от клиента.

---

## Load balancer

**Load balancer** распределяет запросы между несколькими серверами.

```text
                    ┌──────────────┐
              ┌───► │ Backend #1   │
              │     └──────────────┘
┌──────────┐  │
│ Client   │──┤     ┌──────────────┐
└──────────┘  ├───► │ Backend #2   │
              │     └──────────────┘
              │
              │     ┌──────────────┐
              └───► │ Backend #3   │
                    └──────────────┘
```

Зачем нужен:

- больше пропускная способность
- отказоустойчивость
- rolling deploy без остановки сервиса
- равномерная нагрузка

---
layout: two-cols-header
---

## Reverse proxy vs Load balancer

::left::

### Reverse proxy

Главная идея: **скрыть и защитить backend**.

```text
Client ──► nginx ──► backend
```

Может делать:

- TLS termination
- routing
- compression
- static files
- rate limiting

::right::

### Load balancer

Главная идея: **распределить нагрузку**.

```text
Client ──► LB ──┬──► backend #1
                ├──► backend #2
                └──► backend #3
```

На практике один компонент часто делает и то, и другое.

---

# HTTP протокол

---
layout: two-cols-header
---

## Что такое HTTP?

::left::

**HTTP** — протокол прикладного уровня для обмена сообщениями между клиентом и сервером.

HTTP определяет:

- как выглядит request
- как выглядит response
- какие есть методы
- как передавать headers
- как передавать body
- как сообщать об ошибках

::right::

```text
┌────────────┐      HTTP request       ┌────────────┐
│  Client    │ ──────────────────────► │  Server    │
└────────────┘                         └─────┬──────┘
      ▲                                      │
      │             HTTP response            │
      └──────────────────────────────────────┘
```

> HTTP — это язык, на котором клиент и сервер договариваются о запросах и ответах.

---

## HTTP — текстовый протокол

HTTP-сообщение можно прочитать глазами:

```http
GET /api/v1/products?limit=10 HTTP/1.1
Host: shop.example.com
Accept: application/json
User-Agent: Kotlin-HttpClient
```

Ответ тоже читаемый:

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-cache

[
  { "id": 1, "title": "Keyboard" },
  { "id": 2, "title": "Mouse" }
]
```

> Даже если библиотека всё скрывает, полезно понимать, что реально уходит по сети.

---

## URL

**URL** указывает, куда отправить запрос.

```text
https://api.example.com:443/api/v1/users/42?includeOrders=true#profile
│       │               │   │                │                  └─ fragment
│       │               │   │                └─ query string
│       │               │   └─ path
│       │               └─ port
│       └─ host
└─ scheme
```

| scheme | `https` | протокол |
|---|---|---|
| host | `api.example.com` | сервер |
| port | `443` | приложение на сервере |
| path | `/api/v1/users/42` | ресурс |
| query | `includeOrders=true` | дополнительные параметры |

---
layout: two-cols-header
---

## Endpoint

::left::

**Endpoint** — конкретная точка API: HTTP method + path.

```text
GET /api/v1/users
GET /api/v1/users/42
POST /api/v1/users
PATCH /api/v1/users/42
DELETE /api/v1/users/42
```

Один и тот же path с разными method — это разные операции.

::right::

```text
┌───────────────────────────────┐
│ API: Users                    │
├───────────────────────────────┤
│ GET    /users       list      │
│ GET    /users/{id}  details   │
│ POST   /users       create    │
│ PATCH  /users/{id}  update    │
│ DELETE /users/{id}  delete    │
└───────────────────────────────┘
```

> Endpoint — это не просто URL, а точка входа в конкретное действие API.

---

## Path params vs Query params

```http
GET /api/v1/users/42/orders?status=paid&limit=20
```

```text
/api/v1/users/42/orders
              │
              └─ path param: userId = 42

?status=paid&limit=20
 │           │
 ├─ query param: status = paid
 └─ query param: limit = 20
```

Правило большого пальца:

| Тип параметра | Когда использовать | Пример |
|---|---|---|
| Path param | идентифицирует ресурс | `/users/42` |
| Query param | фильтрует или настраивает запрос | `/users?active=true` |

---
layout: two-cols-header
---

## Headers

::left::

**Headers** — метаданные запроса или ответа.

Частые request headers:

- `Accept`
- `Content-Type`
- `Authorization`
- `User-Agent`
- `Cookie`

Частые response headers:

- `Content-Type`
- `Location`
- `Set-Cookie`
- `Cache-Control`

::right::

```http
GET /api/v1/profile HTTP/1.1
Host: api.example.com
Accept: application/json
Authorization: Bearer <token>
User-Agent: MyKotlinApp/1.0
```

```http
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
```

> Headers отвечают на вопрос: «как именно интерпретировать это сообщение?»

---
layout: two-cols-header
---

## Body

::left::

**Body** — полезная нагрузка сообщения.

Обычно body есть у запросов, которые отправляют данные:

- `POST`
- `PUT`
- `PATCH`

У `GET` body почти никогда не используют.

::right::

```http
POST /api/v1/users HTTP/1.1
Host: api.example.com
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

```http
HTTP/1.1 201 Created
Content-Type: application/json

{
  "id": 42,
  "name": "Alice"
}
```

> Body без `Content-Type` — как посылка без подписи, непонятно как её читать.

---
layout: two-cols-header
---

## Content-Type vs Accept

::left::

```http
POST /api/v1/users HTTP/1.1
Content-Type: application/json
Accept: application/json

{
  "name": "Alice"
}
```

```text
Content-Type: "Я отправляю JSON"
Accept:       "Ответь мне тоже JSON"
```

Частые значения:

```text
application/json
text/plain
text/html
application/octet-stream
multipart/form-data
```

::right::

| Header | Что означает |
|---|---|
| `Content-Type` | формат body, который мы отправляем |
| `Accept` | формат response, который мы хотим получить |

---

## HTTP methods

| Method | Смысл | Пример |
|---|---|---|
| `GET` | получить ресурс | `GET /users/42` |
| `POST` | создать ресурс или выполнить команду | `POST /users` |
| `PUT` | полностью заменить ресурс | `PUT /users/42` |
| `PATCH` | частично обновить ресурс | `PATCH /users/42` |
| `DELETE` | удалить ресурс | `DELETE /users/42` |

```text
Важно: в HTTP нет стандартного метода UPDATE.
Для обновления обычно используют PUT или PATCH.
```

---
layout: two-cols-header
---

## PUT vs PATCH

::left::

### PUT — полная замена

```http
PUT /api/v1/users/42
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com",
  "age": 25
}
```

Если поле пропущено — сервер может считать, что его нужно очистить или заменить значением по умолчанию.

::right::

### PATCH — частичное изменение

```http
PATCH /api/v1/users/42
Content-Type: application/json

{
  "email": "new-alice@example.com"
}
```

Меняется только то, что пришло в request body.

---
layout: two-cols-header
---

## Safe methods

::left::

**Safe method** не должен менять состояние сервера.

Safe:

- `GET`
- `HEAD`
- `OPTIONS`

Не safe:

- `POST`
- `PUT`
- `PATCH`
- `DELETE`

::right::

Плохо:

```http
GET /api/v1/users/42/delete
```

Почему плохо:

- браузер может предзагрузить ссылку
- crawler может пройти по ссылке
- proxy/cache могут повторить запрос
- пользователь не ожидает изменения данных от `GET`

---
layout: two-cols-header
---

## Idempotency

::left::

**Идемпотентность**: повторный одинаковый запрос даёт тот же итоговый эффект.

Идемпотентные:

- `GET`
- `PUT`
- `DELETE`

Обычно не идемпотентный:

- `POST`

::right::

```text
DELETE /users/42

1-й раз: пользователь удалён
2-й раз: пользователь уже удалён
Итоговое состояние одинаковое: пользователя нет
```

```text
POST /orders

1-й раз: создан заказ #100
2-й раз: создан заказ #101
Итоговое состояние разное
```

---

## Коды ответа: группы

| Группа | Смысл | Примеры |
|---|---|---|
| `1xx` | информационные | `100 Continue` |
| `2xx` | успех | `200 OK`, `201 Created`, `204 No Content` |
| `3xx` | перенаправление | `301 Moved Permanently`, `304 Not Modified` |
| `4xx` | ошибка клиента | `400 Bad Request`, `401 Unauthorized`, `404 Not Found` |
| `5xx` | ошибка сервера | `500 Internal Server Error`, `503 Service Unavailable` |

```text
4xx: клиент отправил что-то не то
5xx: сервер не смог обработать корректный запрос
```

---

## Самые частые status codes

| Code | Meaning | Когда использовать |
|---|---|---|
| `200 OK` | успех | получили ресурс или выполнили операцию |
| `201 Created` | создано | после успешного `POST` создания ресурса |
| `204 No Content` | успех без body | после `DELETE` или пустого ответа |
| `400 Bad Request` | плохой запрос | некорректный JSON, неверный тип параметра |
| `401 Unauthorized` | не авторизован | нет токена или токен неверный |
| `403 Forbidden` | запрещено | пользователь известен, но прав не хватает |
| `404 Not Found` | не найдено | ресурса или endpoint-а нет |
| `500 Internal Server Error` | ошибка сервера | unexpected exception |

---
layout: two-cols-header
---

## 401 vs 403

::left::

### 401 Unauthorized

Клиент **не доказал, кто он**.

```http
GET /api/v1/profile
Authorization: Bearer invalid-token
```

```http
HTTP/1.1 401 Unauthorized
```

Примеры:

- нет токена
- токен истёк
- токен неверный

::right::

### 403 Forbidden

Сервер знает пользователя, но **доступ запрещён**.

```http
DELETE /api/v1/admin/users/42
Authorization: Bearer user-token
```

```http
HTTP/1.1 403 Forbidden
```

Пример: обычный пользователь пытается выполнить admin-действие.

---
layout: two-cols-header
---

## HTTPS

::left::

**HTTPS = HTTP поверх TLS**.

TLS добавляет:

- шифрование
- проверку подлинности сервера
- защиту от подмены данных

Без HTTPS любой посредник в сети может читать трафик.

::right::

```text
HTTP:
Client ─── "password=123456" ───► Server
        видно провайдеру/Wi‑Fi/proxy

HTTPS:
Client ─── 🔒 encrypted data 🔒 ─► Server
        содержимое не видно посредникам
```

---

## Что защищает HTTPS, а что нет?

| HTTPS защищает | HTTPS не защищает |
|---|---|
| содержимое request/response | от багов в backend-коде |
| токены и пароли при передаче | от утечки токена из логов |
| от подмены ответа по дороге | от SQL injection |
| от прослушки публичного Wi‑Fi | от XSS на frontend-е |

```text
HTTPS обязателен, но он не заменяет валидацию, авторизацию и безопасное хранение секретов.
```


---

# Форматы обмена данными

---
layout: two-cols-header
---

## Зачем нужен формат данных?

::left::

Клиент и сервер часто написаны на разных языках:

- frontend — TypeScript
- mobile app — Kotlin / Swift
- backend — Kotlin / Java / Go
- analytics — Python

Им нужен общий способ договориться, **как выглядят данные**.

::right::

```text
Kotlin object
    │ serialization
    ▼
JSON / Protobuf / XML / CSV
    │ network
    ▼
TypeScript object / Swift object / Python dict
```

> Формат данных — это общий язык между разными программами.

---

# JSON

---
layout: two-cols-header
---

## JSON — JavaScript Object Notation

::left::

**JSON** — текстовый формат обмена данными.

Поддерживает базовые типы:

- object
- array
- string
- number
- boolean
- null

Популярен, потому что:

- читается человеком
- легко дебажить
- поддерживается почти везде
- хорошо подходит для HTTP API

::right::

```json
{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com",
  "active": true,
  "roles": ["USER", "ADMIN"],
  "profile": {
    "city": "Moscow"
  }
}
```

---

## JSON-объект и Kotlin data class

```json
{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com"
}
```

```kotlin
@Serializable
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String
)
```

```text
JSON field "id"    -> Kotlin property id
JSON field "name"  -> Kotlin property name
JSON field "email" -> Kotlin property email
```

> Обычно JSON-поля и свойства DTO называют одинаково, чтобы не усложнять маппинг.

---
layout: two-cols-header
---

## Сериализация и десериализация

::left::

**Сериализация** — превращаем объект в формат для передачи или хранения.

```kotlin
val user = UserResponse(
    id = 42,
    name = "Alice",
    email = "alice@example.com"
)

val json: String = Json.encodeToString(user)
```

```json
{"id":42,"name":"Alice","email":"alice@example.com"}
```

::right::

**Десериализация** — читаем формат обратно в объект.

```kotlin
val json = """
    {
      "id": 42,
      "name": "Alice",
      "email": "alice@example.com"
    }
""".trimIndent()

val user: UserResponse = Json.decodeFromString(json)
```

---

## kotlinx.serialization

Минимальная модель:

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class CreateUserRequest(
    val name: String,
    val email: String
)

@Serializable
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String
)
```

Использование:

```kotlin
val request = CreateUserRequest(
    name = "Alice",
    email = "alice@example.com"
)

val json = Json.encodeToString(request)
val decoded = Json.decodeFromString<CreateUserRequest>(json)
```

> Аннотация `@Serializable` говорит компилятору сгенерировать код для преобразования объекта.

---
layout: two-cols-header
---

## DTO

::left::

**DTO** — Data Transfer Object.

DTO нужен, чтобы описать данные, которые передаются через API.

```kotlin
@Serializable
data class UserResponse(
    val id: Long,
    val name: String,
    val avatarUrl: String?
)
```

DTO — это контракт внешнего API.

::right::

Domain model может быть другой:

```kotlin
data class User(
    val id: UserId,
    val fullName: FullName,
    val passwordHash: String,
    val deletedAt: Instant?
)
```

Почему не отдавать domain model наружу:

- можно случайно раскрыть приватные поля
- внутреннюю модель сложнее менять
- API-контракт должен быть стабильнее кода внутри

---

## Request DTO и Response DTO

Обычно для входа и выхода используют разные модели.

```kotlin
@Serializable
data class CreateUserRequest(
    val name: String,
    val email: String,
    val password: String
)

@Serializable
data class UserResponse(
    val id: Long,
    val name: String,
    val email: String
)
```

Почему пароль есть в request, но нет в response:

```text
Client ── password ──► Server
Client ◄─ user data ── Server
          без password
```

> Никогда не возвращайте пароли, хэши паролей, токены и внутренние служебные поля без необходимости.

---
layout: two-cols-header
---

## Nullable и default values в DTO

::left::

```kotlin
@Serializable
data class UpdateUserRequest(
    val name: String? = null,
    val email: String? = null,
    val avatarUrl: String? = null
)
```

Такой DTO удобен для `PATCH`: клиент может отправить только изменённые поля.

```json
{
  "email": "new-alice@example.com"
}
```

::right::

Но важно различать:

```text
поле отсутствует
поле есть и равно null
поле есть и содержит значение
```

Это особенно важно для частичного обновления.

```json
{}
```

```json
{ "avatarUrl": null }
```

---

## Типичные ошибки с JSON

```text
{
  "id": "42",
  "name": "Alice",
  "email": null,
}
```

Что здесь может пойти не так:

- `id` пришёл строкой, а Kotlin ждёт `Long`
- `email` пришёл `null`, а Kotlin ждёт `String`
- лишняя запятая после последнего поля
- отсутствует обязательное поле
- пришло неизвестное поле

```text
JSON валидный синтаксически ≠ данные валидны для вашего API.
```

---

## JSON: плюсы и минусы

| Плюсы | Минусы |
|---|---|
| читается человеком | больше размер, чем бинарные форматы |
| легко логировать и дебажить | медленнее бинарной сериализации |
| нативно подходит для Web API | нет строгой схемы по умолчанию |
| поддерживается везде | ошибки часто находятся только в runtime |

> Для публичных HTTP API JSON обычно отличный выбор.

---

# Protobuf

---
layout: two-cols-header
---

## Что такое Protobuf?

::left::

**Protocol Buffers** — бинарный формат сериализации от Google.

Главная идея: сначала описываем схему в `.proto`, потом генерируем код.

Особенности:

- бинарный формат
- строгие типы
- компактнее JSON
- быстрее JSON
- schema-first подход
- хорош для backend-to-backend коммуникации

::right::

```text
user.proto
    │ protoc / plugin
    ▼
Generated Kotlin / Java classes
    │ serialize
    ▼
Binary payload
    │ network
    ▼
Generated Go / Python / Java classes
```

---
layout: two-cols-header
---

## .proto-файл

::left::

```protobuf
syntax = "proto3";

package users;

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  bool active = 4;
}

message GetUserRequest {
  int64 id = 1;
}

service UserService {
  rpc GetUser(GetUserRequest) returns (User);
}
```

::right::

Важные детали:

- `message` описывает структуру данных
- `service` описывает RPC-сервис
- числа `1`, `2`, `3` — это field numbers, они важны для совместимости

---
layout: two-cols-header
---

## JSON vs Protobuf

::left::

### JSON

```json
{
  "id": 42,
  "name": "Alice",
  "email": "alice@example.com"
}
```

Плюсы:

- видно глазами
- удобно для браузера
- просто тестировать через curl/Postman

::right::

### Protobuf

```text
08 2A 12 05 41 6C 69 63 65 1A 11 ...
```

Плюсы:

- компактнее
- быстрее
- типизированный контракт
- удобен для генерации клиентов

---
layout: two-cols-header
---

## Совместимость в Protobuf

В Protobuf нельзя бездумно менять field numbers.

::left::


```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
}
```

Можно добавить новое поле:

```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string name = 2;
  string email = 3;
  string avatar_url = 4; // новое поле — OK
}
```

::right::

Плохо:

```protobuf
syntax = "proto3";

message User {
  int64 id = 1;
  string email = 2; // ❌ раньше здесь был name
}
```

> Field numbers — часть бинарного контракта. Их нельзя переиспользовать для другого смысла.

---

## Когда JSON, а когда Protobuf?

| Сценарий | Лучше выбрать |
|---|---|
| публичное REST API | JSON |
| API для браузера | JSON |
| простая интеграция с внешними клиентами | JSON |
| высокая нагрузка между микросервисами | Protobuf |
| нужен строгий контракт и генерация клиентов | Protobuf |
| gRPC API | Protobuf |

```text
JSON — удобно людям и публичным API.
Protobuf — удобно машинам и внутренним сервисам.
```

---

# REST API

---
layout: two-cols-header
---

## Что такое REST?

::left::

**REST** — архитектурный стиль для проектирования API.

В REST мы думаем не командами, а **ресурсами**:

- пользователи
- заказы
- товары
- комментарии
- задачи
- файлы

HTTP method говорит, что сделать с ресурсом.

::right::

```text
Ресурс: users

GET    /users       получить список
GET    /users/42    получить одного
POST   /users       создать
PATCH  /users/42    изменить
DELETE /users/42    удалить
```

> REST — это не библиотека и не фреймворк, а способ проектировать API.

---

## Ресурс

**Ресурс** — сущность, с которой работает API.

```text
/users
/orders
/products
/tasks
/files
/comments
```
---

## Ресурс может быть коллекцией или конкретным объектом:

```text
/users       коллекция пользователей
/users/42    конкретный пользователь

/orders      коллекция заказов
/orders/10   конкретный заказ
```

```text
┌───────────────┐
│ Collection    │ /users
└───────┬───────┘
        │ contains
        ▼
┌───────────────┐
│ Resource item │ /users/42
└───────────────┘
```

---
layout: two-cols-header
---

## RESTful endpoints

::left::

Хороший REST API использует существительные в URL и HTTP methods для действий.

```http
GET    /api/v1/users
GET    /api/v1/users/42
POST   /api/v1/users
PATCH  /api/v1/users/42
DELETE /api/v1/users/42
```

::right::

Плохо:

```http
GET  /api/v1/getUser?id=42
POST /api/v1/createUser
POST /api/v1/updateUserName
POST /api/v1/deleteUser
```

Почему плохо:

- действие дублируется в URL
- сложнее читать и документировать
- хуже используется семантика HTTP

---

## Method + URL = смысл операции

| Операция | REST endpoint |
|---|---|
| Получить всех пользователей | `GET /users` |
| Получить пользователя | `GET /users/{id}` |
| Создать пользователя | `POST /users` |
| Полностью заменить пользователя | `PUT /users/{id}` |
| Частично изменить пользователя | `PATCH /users/{id}` |
| Удалить пользователя | `DELETE /users/{id}` |

```text
Не нужно писать действие в URL, если оно уже выражено HTTP method-ом.
```

---
layout: two-cols-header
---

## Вложенные ресурсы

::left::

Иногда ресурс естественно принадлежит другому ресурсу.

```http
GET /users/42/orders
GET /users/42/orders/100
POST /users/42/orders
```

Это читается как:

```text
заказы пользователя 42
заказ 100 пользователя 42
создать заказ для пользователя 42
```

::right::

Но не стоит делать слишком глубокую вложенность:

```http
/users/42/orders/100/items/5/reviews/7/comments/2
```

Лучше проще:

```http
/comments/2
/reviews/7/comments
/orders/100/items
```

> URL должен помогать понять ресурс, а не превращаться в дерево всей базы данных.

---

## Версионирование API

API меняется: появляются новые поля, меняется поведение, старые клиенты остаются.

Частый вариант — версия в URL:

```http
GET /api/v1/users/42
GET /api/v2/users/42
```

```text
Client v1 ──► /api/v1/users/42
Client v2 ──► /api/v2/users/42
```

Что обычно можно менять без новой версии:

- добавить новое nullable/default поле в response
- добавить новый endpoint
- добавить новый optional query parameter

Что может сломать клиентов:

- удалить поле
- переименовать поле
- изменить тип поля
- изменить смысл status code

---
layout: two-cols-header
---

## Фильтрация

::left::

Фильтрация ограничивает набор ресурсов.

```http
GET /api/v1/users?active=true
GET /api/v1/orders?status=paid
GET /api/v1/products?category=books&minPrice=500
```

Query params хорошо подходят для фильтров.

::right::

```text
/users
  ├─ Alice   active=true
  ├─ Bob     active=false
  └─ Carol   active=true

GET /users?active=true
  ├─ Alice
  └─ Carol
```

---

## Сортировка

Сортировка определяет порядок элементов в коллекции.

```http
GET /api/v1/users?sort=name
GET /api/v1/users?sort=-createdAt
GET /api/v1/products?sort=price
GET /api/v1/products?sort=price,-rating
```

Один из частых форматов:

```text
sort=name       по возрастанию name
sort=-name      по убыванию name
sort=price,-rating
```

```text
Без явной сортировки порядок может быть нестабильным.
Сегодня пользователь на первой странице, завтра — на второй.
```

---
layout: two-cols-header
---

## Пагинация

::left::

Пагинация нужна, чтобы не отдавать слишком много данных за один раз.

```http
GET /api/v1/products?page=1&size=20
GET /api/v1/products?page=2&size=20
```

Без пагинации:

```http
GET /api/v1/products
```

может случайно вернуть миллион записей.

::right::

```json
{
  "items": [
    { "id": 1, "title": "Keyboard" },
    { "id": 2, "title": "Mouse" }
  ],
  "page": 1,
  "size": 20,
  "total": 153
}
```

> Любой endpoint со списком должен иметь стратегию ограничения размера ответа.

---
layout: two-cols-header
---

## Offset pagination vs Cursor pagination

::left::

### Offset / page pagination

```http
GET /products?page=3&size=20
GET /products?offset=40&limit=20
```

Плюсы:

- просто понять
- удобно для страниц в UI

Минусы:

- плохо работает на очень больших данных
- данные могут «прыгать», если список меняется

::right::

### Cursor pagination

```http
GET /products?limit=20&cursor=eyJpZCI6MTAwfQ
```

Плюсы:

- лучше для бесконечной ленты
- стабильнее при изменении данных
- эффективнее для больших таблиц

Минусы:

- сложнее реализовать
- нельзя легко перейти на страницу 100

---

## Пример REST API: TODO

```text
┌────────────────────────────────────────────┐
│ Tasks API                                  │
├──────────────────────┬─────────────────────┤
│ GET /tasks           │ список задач        │
│ GET /tasks/{id}      │ одна задача         │
│ POST /tasks          │ создать задачу      │
│ PATCH /tasks/{id}    │ обновить задачу     │
│ DELETE /tasks/{id}   │ удалить задачу      │
└──────────────────────┴─────────────────────┘
```

```json
{
  "id": 1,
  "title": "Prepare lecture",
  "completed": false
}
```

---

# OpenAPI и Swagger

---
layout: two-cols-header
---

## OpenAPI

::left::

**OpenAPI** — спецификация для описания HTTP API.

Она описывает:

- endpoints
- methods
- path/query parameters
- request body
- response body
- status codes
- authentication
- schemas

::right::

```yaml
openapi: 3.0.3
info:
  title: Tasks API
  version: 1.0.0
paths:
  /tasks/{id}:
    get:
      summary: Get task by id
      parameters:
        - name: id
          in: path
          required: true
          schema:
            type: integer
```

---
layout: two-cols-header
---

## Swagger UI

::left::

**Swagger UI** — визуальная документация, построенная из OpenAPI-спецификации.

Она позволяет:

- посмотреть список endpoints
- увидеть request/response schemas
- отправить тестовый запрос
- проверить status codes
- понять auth requirements

::right::

<img src="https://lh3.googleusercontent.com/bxSweRTTrthHMKU8cXBE4_1W29c4F9xdOneVwHmV9lyghxAkcbzm24sXke-1dhITBsc6_bPi4-nJI_KEoGUoySVl-w=s1280-w1280-h800" width="520" />

---

## Зачем документировать API?

```text
Без документации:
Frontend: "А что сюда отправлять?"
Backend:  "Посмотри код"
QA:       "А какие тут бывают ошибки?"
Mobile:   "Почему поле внезапно null?"
```

```text
С OpenAPI:
Frontend/Mobile/QA/Backend смотрят один контракт.
```

Документация помогает:

- синхронизировать команды
- тестировать API вручную
- генерировать клиентов
- находить breaking changes
- быстрее подключать новых разработчиков

---

# Ошибки и валидация

---
layout: two-cols-header
---

## Ошибки — часть API-контракта

::left::

API должно явно объяснять клиенту, **что пошло не так**.

Плохой ответ:

```http
HTTP/1.1 500 Internal Server Error

Something went wrong
```

Почему плохо:

- непонятно, что исправить
- нельзя нормально показать ошибку пользователю
- сложно дебажить клиенту
- все ошибки выглядят одинаково

::right::

Хороший ответ:

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": "VALIDATION_ERROR",
  "message": "Email has invalid format",
  "field": "email"
}
```

> Ошибки — такой же контракт, как успешные response DTO.

---
layout: two-cols-header
---

## Единый формат ошибки

::left::

Обычно API договаривается об одном формате ошибок.

```kotlin
@Serializable
data class ErrorResponse(
    val error: String,
    val message: String,
    val details: List<FieldError> = emptyList()
)

@Serializable
data class FieldError(
    val field: String,
    val message: String
)
```

::right::

Пример ответа:

```json
{
  "error": "VALIDATION_ERROR",
  "message": "Request body contains invalid fields",
  "details": [
    { "field": "email", "message": "Invalid email format" },
    { "field": "password", "message": "Password is too short" }
  ]
}
```

---
layout: two-cols-header
---

## 400 Bad Request

::left::

`400 Bad Request` — сервер не может понять запрос.

Примеры:

- невалидный JSON
- параметр должен быть числом, но пришла строка
- отсутствует обязательный query parameter
- некорректный `Content-Type`

::right::

```http
GET /api/v1/users/abc HTTP/1.1
```

```http
HTTP/1.1 400 Bad Request
Content-Type: application/json

{
  "error": "BAD_REQUEST",
  "message": "Path parameter 'id' must be a number"
}
```

> `400` — запрос технически некорректен, сервер не может его нормально обработать.

---
layout: two-cols-header
---

## 404 Not Found

::left::

`404 Not Found` — ресурс или endpoint не найден.

Два частых случая:

```http
GET /api/v1/users/999999
```

Пользователя нет.

```http
GET /api/v1/unknown-endpoint
```

Такого endpoint-а нет.

::right::

```http
HTTP/1.1 404 Not Found
Content-Type: application/json

{
  "error": "USER_NOT_FOUND",
  "message": "User with id 999999 was not found"
}
```

> Не найденный ресурс — это не `500`. Сервер работает нормально, просто данных нет.

---
layout: two-cols-header
---

## 409 Conflict

::left::

`409 Conflict` — запрос корректный, но конфликтует с текущим состоянием системы.

Примеры:

- email уже занят
- заказ уже оплачен
- задача уже завершена
- конфликт версий при обновлении

::right::

```http
POST /api/v1/users
Content-Type: application/json

{
  "email": "alice@example.com",
  "name": "Alice"
}
```

```http
HTTP/1.1 409 Conflict

{
  "error": "EMAIL_ALREADY_EXISTS",
  "message": "User with this email already exists"
}
```

---
layout: two-cols-header
---

## 422 Unprocessable Entity

::left::

`422` — request синтаксически корректный, но данные не проходят бизнес-валидацию.

```http
POST /api/v1/users
Content-Type: application/json

{
  "name": "A",
  "email": "not-an-email",
  "password": "123"
}
```

::right::

```http
HTTP/1.1 422 Unprocessable Entity
Content-Type: application/json

{
  "error": "VALIDATION_ERROR",
  "message": "Invalid user data",
  "details": [
    { "field": "name", "message": "Name is too short" },
    { "field": "email", "message": "Invalid email" },
    { "field": "password", "message": "Password is too short" }
  ]
}
```

---

## 400 vs 404 vs 409 vs 422

| Code | Когда использовать | Пример |
|---|---|---|
| `400` | запрос невозможно разобрать | `id=abc`, сломанный JSON |
| `404` | ресурс не найден | пользователя `42` нет |
| `409` | конфликт состояния | email уже занят |
| `422` | данные понятны, но невалидны | email неправильного формата |

```text
400: "Я не понял запрос"
404: "Я понял запрос, но такого ресурса нет"
409: "Запрос нормальный, но конфликтует с текущим состоянием"
422: "Запрос нормальный, но данные не проходят правила"
```

---
layout: two-cols-header
---

## Где валидировать input?

::left::

Валидация обычно идёт слоями:

```text
1. HTTP parsing
2. DTO deserialization
3. DTO validation
4. Business rules
5. Database constraints
```

Чем раньше нашли ошибку, тем лучше.

::right::

```text
Request
  │
  ▼
Parse path/query/body ── invalid syntax ──► 400
  │
  ▼
Validate DTO ─────────── invalid fields ──► 422
  │
  ▼
Check business rules ─── conflict ────────► 409
  │
  ▼
Process successfully ─────────────────────► 2xx
```

---

## Пример простой валидации

```kotlin
@Serializable
data class CreateUserRequest(
    val name: String,
    val email: String,
    val password: String
)

fun validate(request: CreateUserRequest): List<FieldError> = buildList {
    if (request.name.length < 2) {
        add(FieldError("name", "Name must contain at least 2 characters"))
    }
    if (!request.email.contains("@")) {
        add(FieldError("email", "Email must contain @"))
    }
    if (request.password.length < 8) {
        add(FieldError("password", "Password must contain at least 8 characters"))
    }
}
```

> В реальном проекте часто используют validation-библиотеки, но принцип тот же: не доверять входным данным.

---

# Авторизация и безопасность

---

<img src="https://preview.redd.it/every-single-time-v0-q75e2phwyzs71.png?auto=webp&s=4f0185c2c18a8af410ce43dbaafd000cc24e17ac" width="500" />

---
layout: two-cols-header
---

## Authentication vs Authorization

::left::

**Authentication** — проверяем, **кто ты**.

```text
Пользователь вводит login/password
        │
        ▼
Сервер проверяет учётные данные
        │
        ▼
Сервер понимает: это Alice
```

Вопрос authentication:

```text
Кто делает запрос?
```

::right::

**Authorization** — проверяем, **что тебе можно**.

```text
Alice хочет удалить пользователя
        │
        ▼
Сервер проверяет роль Alice
        │
        ▼
Разрешить или запретить действие
```

Вопрос authorization:

```text
Есть ли право на это действие?
```

---
layout: two-cols-header
---

## Authentication и Authorization на примере

::left::

```http
GET /api/v1/admin/users HTTP/1.1
Authorization: Bearer <token>
```

```text
1. Authentication
   token валиден?
   какому пользователю он принадлежит?

2. Authorization
   есть ли у пользователя роль ADMIN?
   можно ли ему читать /admin/users?
```

::right::

Возможные ответы:

| Ситуация | Status code |
|---|---|
| токена нет | `401 Unauthorized` |
| токен неверный | `401 Unauthorized` |
| пользователь известен, но не admin | `403 Forbidden` |
| пользователь admin | `200 OK` |

---
layout: two-cols-header
---

## Basic Auth

::left::

**Basic Auth** передаёт логин и пароль в header.

```http
Authorization: Basic base64(username:password)
```

Пример:

```text
username: alice
password: qwerty

alice:qwerty -> base64 -> YWxpY2U6cXdlcnR5
```

```http
Authorization: Basic YWxpY2U6cXdlcnR5
```

::right::

Важно:

- Base64 — это не шифрование
- Basic Auth можно использовать только поверх HTTPS
- пароль отправляется при каждом запросе
- неудобно для современных публичных API

```text
Base64 можно декодировать обратно:
YWxpY2U6cXdlcnR5 -> alice:qwerty
```

---
layout: two-cols-header
---

## Bearer Token

::left::

**Bearer token** — клиент доказывает доступ, передавая токен.

```http
GET /api/v1/profile HTTP/1.1
Authorization: Bearer eyJhbGciOiJIUzI1NiIs...
```

Идея:

```text
Кто владеет токеном — тот получает доступ.
```

::right::

Типичный flow:

```text
1. Client отправляет login/password
2. Server проверяет их
3. Server возвращает access token
4. Client отправляет token в Authorization header
5. Server проверяет token на каждом запросе
```

> Bearer token нужно защищать как пароль.

---

## Где передавать токен?

Лучше:

```http
Authorization: Bearer <access-token>
```

Плохо:

```http
GET /api/v1/profile?token=<access-token>
```

Почему query parameter плох для токенов:

- URL часто попадает в логи
- URL сохраняется в истории браузера
- URL может попасть в analytics
- URL может утечь через Referer header

```text
Секреты не должны жить в URL.
```

---
layout: two-cols-header
---

## JWT

::left::

**JWT** — JSON Web Token.

Обычно состоит из трёх частей:

```text
header.payload.signature
```

Примерно так:

```text
eyJhbGciOiJIUzI1NiJ9.
eyJzdWIiOiI0MiIsInJvbGUiOiJBRE1JTiJ9.
SflKxwRJSMeKKF2QT4fwpMeJf36POk6yJV_adQssw5c
```

::right::

Payload может содержать claims:

```json
{
  "sub": "42",
  "role": "ADMIN",
  "exp": 1735689600
}
```

Важно:

- payload не шифруется по умолчанию
- подпись защищает от подмены
- `exp` задаёт срок жизни
- нельзя хранить секретные данные в payload

---

## JWT: что проверяет сервер?

```text
Client ── Authorization: Bearer JWT ──► Server
```

Сервер должен проверить:

- подпись токена
- срок действия `exp`
- issuer `iss`, если используется
- audience `aud`, если используется
- права пользователя / роли
- что токен не отозван, если нужна revocation-логика

```text
Подписанный JWT ≠ автоматический доступ ко всему.
```

---
layout: two-cols-header
---

## API key

::left::

**API key** часто используют для service-to-service или доступа внешних интеграций.

```http
GET /api/v1/reports HTTP/1.1
X-API-Key: sk_live_123456789
```

или:

```http
Authorization: ApiKey sk_live_123456789
```

::right::

API key обычно:

- привязан к приложению или интеграции
- имеет ограниченные permissions
- может быть отозван
- может иметь rate limit
- должен храниться как секрет

> API key отвечает скорее на вопрос «какое приложение делает запрос?», а не «какой пользователь?».

---

## Хранение секретов

Секреты нельзя хранить прямо в коде:

```kotlin
// ❌ Плохо
const val API_KEY = "sk_live_123456789"
const val JWT_SECRET = "super-secret"
```

Лучше использовать:

- environment variables
- secret manager
- encrypted config
- переменные CI/CD
- локальные `.env` файлы, которые не попадают в git

```kotlin
val apiKey = System.getenv("API_KEY")
    ?: error("API_KEY environment variable is not configured")
```

---
layout: two-cols-header
---

## Что нельзя логировать?

::left::

Нельзя логировать:

- пароли
- access tokens
- refresh tokens
- API keys
- session cookies
- одноразовые коды
- персональные данные без необходимости

::right::

Плохо:

```text
INFO request headers: Authorization=Bearer eyJhbGciOiJIUzI1NiIs...
```

Лучше:

```text
INFO request userId=42 method=GET path=/profile status=200
```

```text
Authorization=Bearer ***redacted***
```

---

## Мини-чеклист безопасности API

- Использовать HTTPS
- Не передавать секреты в URL
- Проверять authentication на защищённых endpoint-ах
- Проверять authorization отдельно от authentication
- Ограничивать срок жизни токенов
- Не хранить секреты в репозитории
- Не логировать токены и пароли
- Валидировать все входные данные
- Возвращать аккуратные ошибки без stacktrace наружу

> Безопасность API — это не один механизм, а набор привычек.

---
layout: two-cols-header
---

## Зачем нужны API-инструменты?

::left::

API нужно уметь проверять отдельно от frontend-а или mobile app.

Инструменты помогают:

- быстро отправить request
- посмотреть raw response
- проверить status code
- проверить headers
- повторить проблемный запрос
- сохранить коллекцию запросов
- поделиться примером с командой

::right::

```text
Frontend не работает
        │
        ▼
Проверяем API напрямую
        │
        ├── API отвечает корректно -> проблема во frontend
        │
        └── API возвращает ошибку -> дебажим backend/API
```

> Хороший backend-разработчик умеет проверить endpoint без UI.

---

<img src="https://daniel.haxx.se/blog/wp-content/uploads/2021/10/mem-curl-client-libraries.png" width="550" />

---

## curl

`curl` — консольная утилита для HTTP-запросов.

```bash
curl -i https://api.example.com/health
```

`-i` показывает status line и headers:

```http
HTTP/2 200
content-type: application/json

{"status":"OK"}
```

POST с JSON:

```bash
curl -X POST https://api.example.com/users \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer <token>" \
  -d '{"name":"Alice","email":"alice@example.com"}'
```

---
layout: two-cols-header
---

## curl: что полезно помнить

::left::

Частые флаги:

```bash
-i    # показать headers ответа
-v    # verbose: детали соединения
-X    # указать HTTP method
-H    # добавить header
-d    # отправить request body
-o    # сохранить ответ в файл
```

::right::

Пример для отладки auth:

```bash
curl -i https://api.example.com/profile \
  -H "Authorization: Bearer <token>"
```

Пример с query params:

```bash
curl -i "https://api.example.com/users?active=true&limit=10"
```

> URL с `&` лучше брать в кавычки, иначе shell может интерпретировать его по-своему.

---
layout: two-cols-header
---

## Postman / Insomnia

::left::

GUI-инструменты для ручного тестирования API.

Удобны, когда нужно:

- собрать коллекцию запросов
- настроить environments
- передавать auth token
- красиво смотреть JSON
- делиться запросами с командой
- быстро проверять разные body

::right::

<img src="https://learn.microsoft.com/en-us/graph/images/postman-screenshot.png" width="760" />

---

## Swagger UI как инструмент тестирования

Swagger UI — это не только документация, но и быстрый способ проверить endpoint.

```text
1. Открыть endpoint
2. Нажать Try it out
3. Заполнить параметры
4. Нажать Execute
5. Посмотреть curl, status code и response body
```

Плюсы:

- не нужно вручную помнить URL
- видно schema request/response
- удобно демонстрировать API
- хорошо подходит для smoke testing

Минусы:

- не заменяет автоматические тесты
- зависит от актуальности OpenAPI-описания

---
layout: two-cols-header
---

## IntelliJ HTTP Client

::left::

В IntelliJ можно хранить HTTP-запросы прямо в проекте в `.http` файлах.

```http
### Health check
GET http://localhost:8080/health

### Get user
GET http://localhost:8080/users/42
Accept: application/json

### Create user
POST http://localhost:8080/users
Content-Type: application/json

{
  "name": "Alice",
  "email": "alice@example.com"
}
```

::right::

Плюсы:

- запросы версионируются в git
- удобно хранить рядом с кодом
- можно использовать переменные окружения
- легко повторять smoke checks
- подходит для учебных проектов

```text
api.http
  ├─ health check
  ├─ create user
  ├─ get user
  └─ delete user
```

---

## Что выбрать?

| Задача | Инструмент |
|---|---|
| быстро проверить из терминала | curl |
| показать запрос в README/issue | curl |
| ручное исследование API | Postman / Insomnia |
| документация + ручной вызов | Swagger UI |
| хранить примеры запросов в проекте | IntelliJ HTTP Client |

```text
Для курса удобно: Swagger UI для документации + .http файлы для практики.
```

---

# Backend-фреймворки для Kotlin

---
layout: two-cols-header
---

## Зачем нужен backend-фреймворк?

::left::

Можно написать HTTP-сервер руками, но в реальном проекте быстро понадобятся:

- routing
- работа с request/response
- JSON serialization
- validation
- dependency injection
- authentication
- database access
- logging
- configuration
- testing

::right::

```text
Без фреймворка:

Socket -> parse HTTP -> route -> parse JSON
       -> validate -> call service -> build response

С фреймворком:

routing {
    post("/users") { ... }
}
```

> Фреймворк берёт на себя инфраструктуру, чтобы мы писали бизнес-логику.

---

## Популярные варианты

| Framework | Кратко |
|---|---|
| **Spring Boot** | самый популярный enterprise-вариант на JVM |
| **Ktor** | Kotlin-first, лёгкий и гибкий |
| **http4k** | functional-подход: HTTP как функция |
| **Micronaut** | быстрый старт, DI на compile-time |
| **Quarkus** | cloud-native, Kubernetes/GraalVM focus |

```text
Все они могут делать HTTP API.
Различаются философией, весом, экосистемой и уровнем магии.
```

---

<img src="https://i.programmerhumor.io/2025/04/f3ccbf9930f1c78462ce275bc458b765285c530f27340e086cb69bfa0f6bb46c.webp" width="550" />

---
layout: two-cols-header
---

## Spring Boot

::left::

**Spring Boot** — стандарт де-факто для enterprise backend на JVM.

Плюсы:

- огромная экосистема
- много документации
- Spring Security
- Spring Data
- Actuator, metrics, tracing
- легко найти разработчиков

::right::

```kotlin
@RestController
@RequestMapping("/users")
class UserController(
    private val userService: UserService
) {
    @GetMapping("/{id}")
    fun getUser(@PathVariable id: Long): UserResponse =
        userService.getUser(id)
}
```

Минусы:

- много аннотаций
- много «магии»
- тяжелее для первого знакомства

---
layout: two-cols-header
---

## Ktor

::left::

**Ktor** — фреймворк от JetBrains, сделанный специально для Kotlin.

Плюсы:

- Kotlin-first API
- хорошо дружит с корутинами
- лёгкий старт
- явная настройка plugins
- подходит и для server, и для client
- удобно для учебных проектов

::right::

```kotlin
fun Application.module() {
    routing {
        get("/health") {
            call.respondText("OK")
        }

        get("/users/{id}") {
            val id = call.parameters["id"]?.toLongOrNull()
                ?: return@get call.respond(HttpStatusCode.BadRequest)

            call.respond(UserResponse(id = id, name = "Alice"))
        }
    }
}
```

---

## Ktor: plugins вместо «магии»

В Ktor возможности подключаются явно:

```kotlin
fun Application.module() {
    install(ContentNegotiation) {
        json()
    }

    install(StatusPages) {
        exception<IllegalArgumentException> { call, cause ->
            call.respond(
                HttpStatusCode.BadRequest,
                ErrorResponse("BAD_REQUEST", cause.message ?: "Bad request")
            )
        }
    }

    routing {
        get("/health") {
            call.respondText("OK")
        }
    }
}
```

> Хорошо видно, какие возможности включены в приложении.

---
layout: two-cols-header
---

## http4k

::left::

**http4k** смотрит на HTTP максимально просто:

```text
Request -> Response
```

Handler — это функция, которая принимает request и возвращает response.

Плюсы:

- функциональный стиль
- простая модель
- легко тестировать
- мало магии

::right::

```kotlin
val app: HttpHandler = { request: Request ->
    when (request.uri.path) {
        "/health" -> Response(Status.OK).body("OK")
        else -> Response(Status.NOT_FOUND)
    }
}
```

Подходит, если нравится functional programming и хочется максимально явного HTTP.

---
layout: two-cols-header
---

## Micronaut и Quarkus

> Оба фреймворка часто выбирают, когда важны startup time, memory footprint и cloud deployment.

::left::

### Micronaut

- быстрый startup
- dependency injection на compile-time
- меньше reflection
- хорош для микросервисов
- похож на Spring по стилю аннотаций

::right::

### Quarkus

- cloud-native подход
- оптимизация под containers
- Kubernetes-friendly
- GraalVM/native-image focus
- часто используется с Java, но поддерживает Kotlin

---

## Как выбрать фреймворк?

| Если нужно                                            | Можно выбрать |
|-------------------------------------------------------|---|
| enterprise, большая экосистема                        | Spring Boot |
| прототипирование, Kotlin-first и меньше магии | Ktor |
| функциональный минимализм                             | http4k |
| enterprise, быстрый startup и compile-time DI         | Micronaut |
| enterprise, cloud-native и native-image               | Quarkus |

```text
Для обучения Kotlin backend хорошо подходит Ktor:
он достаточно простой, явно показывает HTTP и не прячет всё за аннотациями.
```

---

# Итоги лекции

- как приложения общаются по сети
- IP, порты, socket, DNS, TCP/UDP
- модели TCP/IP и OSI
- клиент-серверную архитектуру
- HTTP request/response
- URL, endpoint, headers, body, query/path params
- HTTP methods, status codes, HTTPS
- JSON, DTO, сериализацию и Protobuf
- REST API и проектирование endpoint-ов
- OpenAPI и Swagger UI
- ошибки, валидацию и формат error response
- authentication, authorization, токены и API keys
- инструменты для проверки API
- Kotlin backend-фреймворки
