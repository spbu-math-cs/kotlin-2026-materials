---
info: Лекция 6 — Spring Framework и Spring Boot
class: text-base
---

# Лекция 6. Spring Framework и Spring Boot

---

<style>
.two-cols-header {
  column-gap: 20px;
}

/* hide slidev navigation toolbar except on hover */
.slidev-nav-controls,
#slidev-nav-controls,
.slidev-controls {
  opacity: 0;
  transition: opacity 0.2s;
}
.slidev-nav-controls:hover,
#slidev-nav-controls:hover,
.slidev-controls:hover {
  opacity: 1;
}
</style>


## Что мы узнаем за 1.5 часа

- зачем вообще нужен фреймворк для backend
- в чём разница между **Spring Framework** и **Spring Boot**
- что такое **IoC** и **Dependency Injection** на пальцах
- как устроен Spring MVC: контроллеры, DTO, валидация, обработка ошибок
- как Spring Data JPA избавляет от ручного SQL для CRUD
- как тестировать Spring-приложение
- чем Spring Boot отличается от Ktor, и когда выбирать что

> Стек слайдов: **Kotlin 2.2.20 + Spring Boot 3.5.6 + Spring Data JPA + H2**.

---

# Блок 1. Зачем нужен Spring

---
layout: two-cols-header
---

## Backend «голыми руками» — это больно

::left::

Пишем REST API на чистой JVM. Что **самим** придётся сделать?

- открыть TCP-сокет, разобрать HTTP-запрос
- маршрутизировать `/api/bookings/{id}`
- (де)сериализовать JSON, валидировать DTO
- работать с БД и пулом потоков
- читать конфигурацию из env
- ловить ошибки и возвращать осмысленные коды

::right::

```text
┌─────────────────────────────────────┐
│            ВАШ КОД                  │
│  (бизнес-логика бронирования)       │
│                                     │
│  ───────────────────────────────    │
│  HTTP-парсер | роутинг | thread     │
│  pool | JSON | валидация | пул      │
│  соединений | конфигурация | DI |   │
│  логи | health-check | метрики      │
└─────────────────────────────────────┘
       ↑↑↑ всё это — мусор, который   ↑↑↑
       ↑↑↑ ты не должен писать сам    ↑↑↑
```

> 90% времени уходит на инфраструктуру, а не на бизнес-логику.

---

## Что делает фреймворк

**Фреймворк** — это «инверсия рутинных решений».

Вы пишете только бизнес-логику и **декларативно** говорите фреймворку:

- «вот контроллер на пути `/api/bookings`»;
- «вот класс с бизнес-логикой, дай ему репозиторий»;
- «вот DTO, проверь обязательные поля».

Всё остальное — старт сервера, парсинг HTTP, JSON, DI-контейнер, транзакции, валидацию, обработку ошибок — берёт на себя фреймворк.

---

## Что Spring Boot делает за вас

| Что писали бы сами | Что делает Spring Boot |
|---|---|
| `ServerSocket.accept()` | embedded Tomcat запустится сам |
| HTTP-парсер | Spring MVC |
| Сериализация JSON | Jackson + `jackson-module-kotlin` |
| Валидация | Jakarta Validation |
| DI-граф | Spring IoC-контейнер |
| Транзакции | `@Transactional` + `JpaTransactionManager` |
| Конфигурация | `application.yml` + `@Value` / `@ConfigurationProperties` |

> Каждая строчка таблицы — это десятки часов работы, которые **уже сделаны** за вас.

---

## Откуда взялся Spring

```text
2003 ──► Spring Framework 1.0
         (ответ на громоздкость EJB 2.x)

2014 ──► Spring Boot 1.0
         (opinionated дистрибутив поверх Spring Framework)

2017 ──► Spring 5 + reactive (WebFlux)
2022 ──► Spring 6 + Spring Boot 3 (Jakarta EE 9, Java 17+)
2025 ──► Spring Boot 3.5.x (наша версия)
```

- Spring Framework сделал DI/IoC мейнстримом на JVM.
- Spring Boot убрал «ритуал» из Spring: больше не нужно вручную писать XML-конфиги, поднимать Tomcat, описывать DataSource.
- Сегодня Spring Boot — самый распространённый Java/Kotlin backend-фреймворк в индустрии.

---

## Мем №1: вручную или одной аннотацией?

<img src="./lecture-06/imgs/drake-tomcat.png" width="420" />

> Spring Boot не отменяет Tomcat — он просто конфигурирует его за вас.

---

## Что вы получите от Spring уже на 1-м курсе

- Сразу пишете **рабочий REST API**, не залипая в HTTP-парсере.
- Учитесь на индустриальных паттернах: layered architecture, DI, DTO, repository.
- Тот же код на 1-м курсе и в реальной работе — это редкая удача.
- На собеседованиях про Spring спрашивают **всегда**.

> Минус один: фреймворк делает много «магии». Часть лекции мы потратим именно на то, чтобы эта магия перестала быть магией.

---

# Блок 2. Spring Framework vs Spring Boot

---
layout: two-cols-header
---

## Spring Framework — это «зонтик»

::left::

Spring Framework — это **много модулей**, каждый из которых решает одну задачу:

- **Spring Core** — IoC-контейнер, DI
- **Spring MVC** — REST/HTTP-слой
- **Spring WebFlux** — реактивный аналог
- **Spring Data** — JPA, MongoDB, Redis…
- **Spring Security** — auth/authz
- **Spring AOP** — аспекты, прокси
- **Spring Test** — тестовая инфраструктура
- **Spring Messaging / Integration / Batch / Cloud / …**

::right::

```text
       Spring Framework (большой набор)
   ┌──────┬──────┬──────┬──────┬──────┐
   │ Core │ MVC  │ Data │ Sec  │ Test │
   └──────┴──────┴──────┴──────┴──────┘

       Spring Boot (готовый бутерброд)
   ┌─────────────────────────────────┐
   │  starter + autoconfig + tomcat  │
   └─────────────────────────────────┘
```

> Сам Spring Framework требует много ручной настройки. Spring Boot — это готовый «дистрибутив поверх».

---

## Что добавляет Spring Boot

- **Starter-зависимости** — одна строка вместо десяти.
- **Autoconfiguration** — увидел библиотеку на classpath → сконфигурил автоматически.
- **Embedded server** — приложение само запускает Tomcat/Jetty/Undertow.
- **Sensible defaults** — любые настройки можно переопределить через `application.yml` / переменные окружения.
- **Production-ready** — `/actuator/health`, метрики, логи, graceful shutdown.

```kotlin
@SpringBootApplication
class BookingApplication

fun main(args: Array<String>) {
    runApplication<BookingApplication>(*args)
}
```

> Это **полноценное** веб-приложение. Серьёзно. Запускайте.

---

## Что такое starter

`starter` — это «зависимость-зонтик». Один артефакт — согласованный набор библиотек.

```kotlin
implementation(
    "org.springframework.boot:spring-boot-starter-web"
)
```

Под капотом приедут: `spring-web` + `spring-webmvc`, embedded Tomcat, `jackson-databind`, `spring-boot-starter-json`, `spring-boot-starter-logging`.

---

## Самые частые starter-ы

| Starter | Что внутри |
|---|---|
| `…-starter-web` | Spring MVC + Tomcat + Jackson |
| `…-starter-validation` | Jakarta Validation |
| `…-starter-data-jpa` | Hibernate + Spring Data JPA |
| `…-starter-test` | JUnit 5, MockMvc, Mockito, AssertJ |
| `…-starter-security` | Spring Security |
| `…-starter-actuator` | health, metrics, info |

> Всё версионировано Spring Boot BOM — версии гарантированно совместимы.

---

## Autoconfiguration — «магия» с условиями

Spring Boot смотрит на classpath и **условно** подключает свои конфигурации.

```text
classpath содержит h2.jar
        + property spring.datasource.url не задана
   ─►   AutoConfig: создать H2 DataSource в памяти

classpath содержит spring-webmvc + tomcat-embed-core
   ─►   AutoConfig: запустить embedded Tomcat
        + сконфигурировать DispatcherServlet
```

Технически это десятки `@Configuration`-классов с аннотациями вида:

```kotlin
@Configuration
@ConditionalOnClass(DataSource::class)
@ConditionalOnMissingBean(DataSource::class)
class DataSourceAutoConfiguration { /* ... */ }
```

> Идея: **«я подключусь, только если ты сам ничего не сконфигурировал».**

---

# Блок 3. Hello, Spring Boot

---
layout: two-cols-header
---

## Минимальный проект

::left::

Структура минимального Spring Boot-приложения:

```text
booking/
├── build.gradle.kts
└── src/
    └── main/
        ├── kotlin/
        │   └── com/example/booking/
        │       ├── BookingApplication.kt
        │       ├── BookingController.kt
        │       ├── BookingService.kt
        │       └── BookingRepository.kt
        └── resources/
            └── application.yml
```

::right::

`BookingApplication.kt`:

```kotlin
package com.example.booking

import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class BookingApplication

fun main(args: Array<String>) {
    runApplication<BookingApplication>(*args)
}
```

> Один файл, один класс, один `main` — и у вас уже есть REST-сервер.

---

## Что делает `@SpringBootApplication`

```kotlin
@SpringBootApplication
class BookingApplication
```

Это **мета-аннотация**, под капотом она равна:

```kotlin
@Configuration
@EnableAutoConfiguration
@ComponentScan
class BookingApplication
```

| Аннотация | Что включает |
|---|---|
| `@Configuration` | класс может содержать `@Bean`-методы |
| `@EnableAutoConfiguration` | включить весь механизм autoconfig |
| `@ComponentScan` | найти все `@Component`-классы в этом пакете и подпакетах |

> Поэтому пакет `BookingApplication` обычно — корень всего проекта: компонент-скан идёт от него.

---

## `application.yml` — конфигурация без перекомпиляции

<style scoped>
pre, code { font-size: 0.85em; }
</style>

```yaml
server:
  port: 8080

spring:
  application:
    name: booking
  datasource:
    url: jdbc:h2:mem:booking-db
    username: sa
  h2:
    console:
      enabled: true
      path: /h2-console
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

Любое значение переопределяется переменной окружения (`SERVER_PORT=9090`), флагом (`--server.port=9090`) или профилем `application-prod.yml`.

---

## `build.gradle.kts` — стек 2025 года

```kotlin
plugins {
    kotlin("jvm") version "2.2.20"
    kotlin("plugin.spring") version "2.2.20"
    id("org.springframework.boot") version "3.5.6"
    id("io.spring.dependency-management") version "1.1.7"
}

repositories { mavenCentral() }

dependencies {
    implementation("org.springframework.boot:spring-boot-starter-web")
    implementation("org.springframework.boot:spring-boot-starter-validation")
    implementation("org.springframework.boot:spring-boot-starter-data-jpa")
    implementation("com.fasterxml.jackson.module:jackson-module-kotlin")
    implementation("org.jetbrains.kotlin:kotlin-reflect")
    runtimeOnly("com.h2database:h2")
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}
```

> Выучите наизусть: это «нулевая точка» любого Spring Boot-проекта на Kotlin.

---

# Блок 4. IoC + Dependency Injection

---
layout: two-cols-header
---

## Сценка из жизни без DI

::left::

```kotlin
class BookingController {
    private val bookingService =
        BookingService(
            roomRepository = RoomRepository(
                entityManager = JpaEntityManager(
                    dataSource = HikariDataSource(
                        url = "jdbc:h2:mem:booking",
                        // ...
                    )
                )
            ),
            bookingRepository = BookingRepository(/* ... */),
        )
}
```

::right::

Что в этом плохо:

- `BookingController` **знает**, как создать `DataSource`. Не его дело.
- Невозможно подменить `RoomRepository` фейком в тесте.
- Любое изменение конструктора `RoomRepository` ломает весь файл.
- Жёсткая зависимость от **конкретных реализаций**, а не контрактов.
- Граф зависимостей — каша из `new`. Спагетти.

> Это классический пример «новизмов» — когда `new` (или конструктор) расползается по всему коду.

---

## IoC — «кто кого создаёт»

**Inversion of Control** — это переворот вопроса «кто кого создаёт».

```text
Без IoC:                       С IoC (контейнер):

Controller ──new──► Service     Container
Service    ──new──► Repository    │ создаёт всё
Repository ──new──► EntityMgr     │ и связывает
                                  ▼
                              Controller ◄── Service ◄── Repository
```

- **Без IoC**: ваш код активно создаёт зависимости.
- **С IoC**: контейнер создаёт зависимости и **отдаёт их вам** через конструктор.

Контейнер у Spring — `ApplicationContext`. Объекты, которыми он управляет, называются **bean-ами**.

---

## DI — частный случай IoC

**Dependency Injection** — конкретная техника инверсии: «зависимости приходят в объект снаружи».

В Kotlin со Spring используют **constructor injection**:

```kotlin
import org.springframework.stereotype.Service

@Service
class BookingService(
    private val rooms: RoomRepository,
    private val bookings: BookingRepository,
) {
    fun listForRoom(roomId: Long): List<Booking> =
        bookings.findAllByRoomId(roomId)
}
```

- `@Service` помечает класс как bean.
- Spring сам создаст `RoomRepository`, `BookingRepository` и передаст их в конструктор.
- Поля можно сделать `val` — у нас иммутабельная сборка.

---

## Граф бинов собирается автоматически

```text
                    ApplicationContext

  ┌───────────────────┐
  │ BookingController │ ───┐
  └───────────────────┘    │ зависит от
                           ▼
                ┌───────────────────┐
                │ BookingService    │ ───┐ зависит от
                └───────────────────┘    │
                                         ▼
                ┌───────────────────┐  ┌───────────────────┐
                │ RoomRepository    │  │ BookingRepository │
                └───────────────────┘  └───────────────────┘
                          │                    │
                          └──────────┬─────────┘
                                     ▼
                            ┌──────────────────┐
                            │ EntityManager    │
                            └──────────────────┘
```

> Контейнер строит граф **в правильном порядке**, кеширует синглтоны и при необходимости обнаруживает циклы.

---

## Зачем `kotlin("plugin.spring")`

Spring часто оборачивает ваши `@Service`/`@Configuration`/`@Transactional` в **прокси** (например, для транзакций или AOP).

Чтобы прокси работал классически (через CGLIB), классы и методы должны быть **`open`** — а в Kotlin по умолчанию всё `final`.

Плагин `kotlin("plugin.spring")` автоматически делает классы с аннотациями `@Component`, `@Configuration`, `@Service`, `@RestController`, `@Transactional`, `@Async` — `open`.

```kotlin
plugins {
    kotlin("plugin.spring") version "2.2.20"
}
```

> Без этого плагина вы рано или поздно получите ошибку «Class … must be `open`».

---

## Мем №2: handwritten DI vs Spring DI

<img src="./lecture-06/imgs/distracted-di.png" width="500" />

> «Я сам напишу DI-контейнер за выходные» © каждый второй джун, который через год переходит на Spring.

---

# Блок 5. Стереотипы и слои

---
layout: two-cols-header
---

## Аннотации-стереотипы

::left::

| Аннотация | Где использовать |
|---|---|
| `@Component` | универсальный bean |
| `@Service` | бизнес-логика |
| `@Repository` | доступ к данным |
| `@RestController` | HTTP-эндпоинт (REST) |
| `@Configuration` | bean-фабрика, ручная конфигурация |

::right::

Технически `@Service`, `@Repository`, `@RestController`, `@Configuration` — это всё **специализации `@Component`**.

```kotlin
@Component
@Target(...)
annotation class Service
```

Зачем тогда столько?
- Семантика для читателя кода: видя `@Repository`, я знаю «это persistence».
- Spring добавляет **дополнительное поведение** для специализаций (например, `@Repository` оборачивает SQL-исключения).
- Помогает инструментам (IDE, плагин Spring Explyt) подсвечивать слои.

---

## Слоистая архитектура

```text
   ┌──────────────────────────────────┐
   │  @RestController (Controller)    │  ← HTTP, JSON, статусы
   └──────────────────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────┐
   │  @Service (бизнес-логика)        │  ← правила, транзакции
   └──────────────────────────────────┘
                  │
                  ▼
   ┌──────────────────────────────────┐
   │  @Repository (Spring Data JPA)   │  ← запросы к БД
   └──────────────────────────────────┘
                  │
                  ▼
            ┌──────────┐
            │    DB    │
            └──────────┘
```

---

## Зачем разделять на слои

- **Тестируемость**: каждый слой можно мокировать.
- **Заменяемость**: H2 → PostgreSQL — меняется только конфигурация.
- **Понятность**: новый человек в команде видит, где искать что.
- **Контракты**: контроллер общается с миром, сервис — с моделью, репозиторий — с БД. Эти контракты не должны протекать друг в друга.

> Антипаттерн: SQL-запрос в контроллере. Это сразу значит, что у вас нет ни сервиса, ни архитектуры.

---

## Где живёт DTO

DTO (`Data Transfer Object`) — это объекты-«перевозчики» между слоями.

```text
            HTTP                       JPA Entity
   client ───────►  CreateBookingRequest ──► Booking
   client ◄───────  BookingResponse     ◄── Booking
```

- DTO — стабильный **внешний контракт** API.
- Entity — **внутренняя** модель базы данных.
- Их жизненные циклы развязаны: можно поменять схему БД, не ломая клиентов.

---

## Конфигурация-как-bean

`@Configuration` — для случаев, когда нужно создать bean руками:

```kotlin
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration
import com.fasterxml.jackson.databind.ObjectMapper
import com.fasterxml.jackson.module.kotlin.jacksonObjectMapper

@Configuration
class JsonConfig {
    @Bean
    fun objectMapper(): ObjectMapper = jacksonObjectMapper()
}
```

> На практике 99% bean-ов — это просто `@Service`/`@Repository`/`@RestController`. `@Configuration` нужен для редких случаев интеграции со сторонними библиотеками.

---

# Блок 6. Spring MVC: REST API

---
layout: two-cols-header
---

## `@RestController` за 30 секунд

::left::

```kotlin
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/bookings")
class BookingController(
    private val service: BookingService,
) {

    @GetMapping
    fun list(): List<BookingResponse> =
        service.findAll()
}
```

::right::

- `@RestController` = `@Controller` + `@ResponseBody`. Возвращаемое значение **всегда** идёт в response body как JSON.
- `@RequestMapping("/api/bookings")` — общий префикс для всех методов.
- `@GetMapping` без аргумента → путь `/api/bookings`.

> Запустите `BookingApplication.main()` и откройте `http://localhost:8080/api/bookings`. Серьёзно. Уже работает.

---

## Mapping-аннотации

| Аннотация | HTTP-метод |
|---|---|
| `@GetMapping` | GET |
| `@PostMapping` | POST |
| `@PutMapping` | PUT |
| `@PatchMapping` | PATCH |
| `@DeleteMapping` | DELETE |

```kotlin
@GetMapping("/{id}")
fun get(@PathVariable id: Long): BookingResponse =
    service.getById(id)

@PostMapping
fun create(@RequestBody req: CreateBookingRequest): BookingResponse =
    service.create(req)
```

---

## Параметры — три типа

| Где живут данные | Аннотация | Пример |
|---|---|---|
| URL-путь | `@PathVariable` | `/api/bookings/42` |
| Query string | `@RequestParam` | `/api/bookings?roomId=1` |
| Тело запроса | `@RequestBody` | JSON в POST |

```kotlin
@GetMapping("/{id}")
fun get(@PathVariable id: Long): BookingResponse = ...

@GetMapping
fun list(@RequestParam roomId: Long?): List<BookingResponse> = ...

@PostMapping
fun create(@RequestBody @Valid request: CreateBookingRequest): BookingResponse = ...
```

> Эти три аннотации покрывают 95% реальной работы.

---

## ResponseEntity — когда нужен полный контроль

```kotlin
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity

@DeleteMapping("/{id}")
fun delete(@PathVariable id: Long): ResponseEntity<Unit> {
    service.delete(id)
    return ResponseEntity.noContent().build() // 204
}

@PostMapping
fun create(@RequestBody @Valid request: CreateBookingRequest)
    : ResponseEntity<BookingResponse> {
    val response = service.create(request)
    return ResponseEntity.status(HttpStatus.CREATED).body(response) // 201
}
```

> Если статус и заголовки стандартные — возвращайте просто DTO. `ResponseEntity` — для случаев «мне нужно явно что-то поменять».

---

## HTTP-коды для CRUD

<style scoped>
table { font-size: 0.72em; }
table td, table th { padding-top: 0.25em; padding-bottom: 0.25em; }
blockquote { font-size: 0.85em; margin-top: 0.5em; }
</style>

| Сценарий | Код |
|---|---|
| Успешное чтение | `200 OK` |
| Успешное создание | `201 Created` |
| Успешное удаление | `204 No Content` |
| Невалидный JSON / DTO | `400 Bad Request` |
| Не аутентифицирован | `401 Unauthorized` |
| Запрещено | `403 Forbidden` |
| Ресурс не найден | `404 Not Found` |
| Конфликт бизнес-правил | `409 Conflict` |
| Внутренняя ошибка | `500 Internal Server Error` |

> «Комната уже занята» — это **409 Conflict**, а не 400.

---

## Пример запроса и ответа

```http
POST /api/bookings HTTP/1.1
Content-Type: application/json

{
  "roomId": 1,
  "userName": "Anna",
  "startsAt": "2026-05-10T10:00:00",
  "endsAt": "2026-05-10T11:00:00"
}
```

Ответ:

```http
HTTP/1.1 201 Created
Content-Type: application/json
```

```json
{
  "id": 42,
  "roomId": 1,
  "roomName": "Tesla",
  "userName": "Anna",
  "startsAt": "2026-05-10T10:00:00",
  "endsAt": "2026-05-10T11:00:00"
}
```

---

## Lifecycle одного HTTP-запроса

```text
client ──HTTP──► Tomcat
                   │
                   ▼
              DispatcherServlet  (front controller Spring MVC)
                   │  выбор HandlerMapping
                   ▼
              HandlerAdapter
                   │  Jackson + jackson-module-kotlin: JSON → DTO
                   ▼
              BookingController.create(@Valid CreateBookingRequest)
                   │  если @Valid не прошёл → MethodArgumentNotValidException
                   ▼
              BookingService.create(...)
                   │  бизнес-логика, транзакция
                   ▼
              BookingRepository.save(...)  (Hibernate → SQL → H2)
                   ▼
              BookingResponse (DTO)
                   │  Jackson: DTO → JSON
                   ▼
              HTTP 201 + body
```

> На практике: 99% багов в HTTP-слое — это **сериализация** (JSON ↔ DTO) или **валидация**.

---

# Блок 7. DTO + Validation + ошибки

---

## Зачем нужны DTO

- API не должен зависеть от схемы БД.
- В JPA-Entity легко прокинуть лишние поля наружу (например, hash пароля).
- DTO — это **публичный контракт**, его правят с осторожностью.
- Можно делать разные DTO для request и response, для list-view и detail-view.

```text
   База:                         Снаружи (JSON):

   Booking                       BookingResponse
   ├─ id          ◄── маппинг ──►├─ id
   ├─ roomId                     ├─ roomId
   ├─ userName                   ├─ roomName     (+ из Room)
   ├─ startsAt                   ├─ userName
   ├─ endsAt                     ├─ startsAt
   ├─ createdAt   (внутреннее)   └─ endsAt
   └─ deletedAt   (внутреннее)
```

---

## Jakarta Validation — аннотации

```kotlin
import jakarta.validation.constraints.Future
import jakarta.validation.constraints.NotBlank
import jakarta.validation.constraints.Positive
import java.time.LocalDateTime

data class CreateBookingRequest(
    @field:Positive(message = "Room id must be positive")
    val roomId: Long,

    @field:NotBlank(message = "User name must not be blank")
    val userName: String,

    @field:Future(message = "Start time must be in the future")
    val startsAt: LocalDateTime,

    @field:Future(message = "End time must be in the future")
    val endsAt: LocalDateTime,
)
```

> В Kotlin **обязательно** писать `@field:NotBlank`, а не просто `@NotBlank`. Без префикса `field:` аннотация попадёт на параметр конструктора, и валидатор её не увидит.

---

## `@Valid` запускает проверку

```kotlin
import jakarta.validation.Valid
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody

@PostMapping
fun create(
    @Valid @RequestBody request: CreateBookingRequest,
): BookingResponse = service.create(request)
```

Если валидация падает — Spring сам кидает `MethodArgumentNotValidException`. Дальше его поймает наш `@RestControllerAdvice` и превратит в красивый JSON 400.

> Аннотация `@Valid` — **триггер** валидации. Без неё аннотации `@field:NotBlank` ничего не делают.

---
layout: two-cols-header
---

## Доменные исключения

::left::

Не всё можно проверить аннотациями. Бизнес-правила — отдельная история.

```kotlin
class RoomNotFoundException(roomId: Long) :
    RuntimeException("Room $roomId not found")

class BookingNotFoundException(bookingId: Long) :
    RuntimeException("Booking $bookingId not found")

class InvalidBookingTimeException :
    RuntimeException(
        "Booking end time must be after start time"
    )

class RoomAlreadyBookedException(roomId: Long) :
    RuntimeException(
        "Room $roomId is already booked for this time"
    )
```

::right::

Маппинг исключений на HTTP-коды:

| Исключение | Код |
|---|---|
| `RoomNotFoundException` | `404` |
| `BookingNotFoundException` | `404` |
| `InvalidBookingTimeException` | `400` |
| `RoomAlreadyBookedException` | `409` |
| `MethodArgumentNotValidException` | `400` |

> Дружеское правило: контроллер не знает про исключения. Он просто вызывает сервис. Перехват — в `@RestControllerAdvice`.

---

## `@RestControllerAdvice` — доменные исключения

```kotlin
import org.springframework.http.HttpStatus
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.ResponseStatus
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class ApiExceptionHandler {

    @ExceptionHandler(
        RoomNotFoundException::class,
        BookingNotFoundException::class,
    )
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(e: RuntimeException) =
        ErrorResponse(e.message ?: "Not found")

    @ExceptionHandler(RoomAlreadyBookedException::class)
    @ResponseStatus(HttpStatus.CONFLICT)
    fun handleConflict(e: RoomAlreadyBookedException) =
        ErrorResponse(e.message ?: "Conflict")
}
```

> Один класс для всех контроллеров: HTTP-коды живут в одном месте.

---

## `@RestControllerAdvice` — валидация и ответ

```kotlin
import org.springframework.http.HttpStatus
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.ResponseStatus
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class ApiValidationHandler {

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidation(e: MethodArgumentNotValidException): ErrorResponse {
        val msg = e.bindingResult.fieldErrors.firstOrNull()
            ?.defaultMessage ?: "Validation failed"
        return ErrorResponse(msg)
    }
}

data class ErrorResponse(
    val message: String,
    val timestamp: java.time.Instant = java.time.Instant.now(),
)
```

> Два advice — для наглядности. На практике хватит одного класса.

---

## Мем №3: `@Valid` спасёт продакшн

<img src="./lecture-06/imgs/rollsafe-valid.png" width="500" />

> Шутка наполовину. Реально: 70% «прод упал на пользовательском вводе» лечится дисциплиной DTO + `@Valid`.

---

# Блок 8. Spring Data JPA

---

## Entity — это класс с аннотациями JPA

```kotlin
import jakarta.persistence.*
import java.time.LocalDateTime

@Entity
@Table(name = "bookings")
class Booking(
    @Id @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,
    @Column(nullable = false) val roomId: Long,
    @Column(nullable = false) val userName: String,
    @Column(nullable = false) val startsAt: LocalDateTime,
    @Column(nullable = false) val endsAt: LocalDateTime,
)
```

> Модель плоская: `roomId: Long`. На практике вы увидите `@ManyToOne` / `@JoinColumn`.

---

## Репозиторий — это интерфейс

```kotlin
import org.springframework.data.jpa.repository.JpaRepository

interface RoomRepository : JpaRepository<Room, Long>

interface BookingRepository : JpaRepository<Booking, Long> {
    fun findAllByRoomId(roomId: Long): List<Booking>
}
```

Что мы получили **бесплатно**:

- `findAll()`, `findById(id)`, `save(entity)`, `deleteById(id)`, `count()` — все базовые CRUD-операции.
- `findAllByRoomId(roomId)` — Spring Data JPA сгенерирует SQL-запрос **по имени метода**. Это называется *derived query*.
- Типизация по `Booking, Long`: компилятор не даст вам передать `String` вместо id.

> Реализацию интерфейса Spring сгенерирует во время старта приложения.

---

## Derived queries — префиксы

| Префикс | Действие |
|---|---|
| `findBy…` | SELECT |
| `existsBy…` | SELECT EXISTS |
| `countBy…` | SELECT COUNT |
| `deleteBy…` | DELETE |

```kotlin
fun findAllByRoomIdAndStartsAtBetween(
    roomId: Long,
    from: LocalDateTime,
    to: LocalDateTime,
): List<Booking>
```

> Spring Data JPA разберёт имя метода и сгенерирует SQL.

---

## Derived queries — фразы в имени

| Слово в имени | Что значит |
|---|---|
| `…RoomId` | где `roomId = ?` |
| `…StartsAtAfter` | где `startsAt > ?` |
| `…StartsAtBetween` | где `startsAt BETWEEN ? AND ?` |
| `…OrderByStartsAtAsc` | + `ORDER BY startsAt ASC` |

> Когда имена становятся длиннее экрана — пишите `@Query("SELECT b FROM Booking b WHERE …")`.

---

## H2 + `/h2-console`

H2 — embedded SQL-БД на Java. Идеальна для учебных проектов и тестов.

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:booking-db
  h2:
    console:
      enabled: true
      path: /h2-console
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

После запуска:
- откройте `http://localhost:8080/h2-console`,
- JDBC URL: `jdbc:h2:mem:booking-db`,
- видите ваши таблицы и можете писать SQL руками.

> `ddl-auto: update` — Hibernate создаст или достроит таблицы по entity. Для прода так делать **не нужно** — там используют миграции (Flyway, Liquibase).

---

## CommandLineRunner для seed-данных

```kotlin
import org.springframework.boot.CommandLineRunner
import org.springframework.stereotype.Component

@Component
class DataSeeder(
    private val rooms: RoomRepository,
) : CommandLineRunner {

    override fun run(vararg args: String) {
        if (rooms.count() == 0L) {
            rooms.saveAll(listOf(
                Room(name = "Tesla",   capacity = 8),
                Room(name = "Edison",  capacity = 4),
                Room(name = "Newton",  capacity = 12),
            ))
        }
    }
}
```

> Любой `@Component`, реализующий `CommandLineRunner`, выполнится после старта приложения.

---

# Блок 9. Тесты

---

## Пирамида тестов

```text
              ▲
              │  e2e
              │  (HTTP + DB + всё реально)
            ──┼──
              │  интеграционные
              │  (@SpringBootTest + H2)
            ──┼──
              │  юнит
              │  (один класс, без Spring)
              ▼
```

- **Юнит**: чистый `BookingService` с моками. Быстро, но не ловит интеграционные баги.
- **Интеграция**: поднимаем реальный контекст Spring + H2. Медленнее, но проверяет, что аннотации, mapping, валидация и БД действительно работают.
- **e2e**: куда меньше, обычно отдельный CI-стенд.

> На практике-11 мы пишем **интеграционные** тесты — они дают максимум уверенности при минимуме боли.

---

## `@SpringBootTest` + H2

```kotlin
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.test.web.servlet.MockMvc

@SpringBootTest
@AutoConfigureMockMvc
class BookingApiTest {

    @Autowired
    lateinit var mockMvc: MockMvc

    @Autowired
    lateinit var bookings: BookingRepository
}
```

- `@SpringBootTest` поднимает **полный** ApplicationContext.
- `@AutoConfigureMockMvc` подключает `MockMvc` — фейковый HTTP-клиент против вашего DispatcherServlet **без реального сетевого слоя**.
- БД будет H2 — она настроена в `application.yml`.

---

## MockMvc через Kotlin DSL

```kotlin
import org.springframework.test.web.servlet.post
import org.springframework.test.web.servlet.get
import com.fasterxml.jackson.databind.ObjectMapper

@Test
fun `create booking returns 201 and body`() {
    val request = CreateBookingRequest(
        roomId = 1,
        userName = "Anna",
        startsAt = LocalDateTime.now().plusHours(1),
        endsAt = LocalDateTime.now().plusHours(2),
    )

    mockMvc.post("/api/bookings") {
        contentType = MediaType.APPLICATION_JSON
        content = ObjectMapper().writeValueAsString(request)
    }.andExpect {
        status { isCreated() }
        jsonPath("$.userName") { value("Anna") }
        jsonPath("$.roomId")   { value(1) }
    }
}
```

> Kotlin-DSL у MockMvc — это просто extension-функции `post {}` и `andExpect {}`. Никакой магии, всё видно по импортам.

---

## Что тестируем в Spring-приложении

| Сценарий | Тип теста |
|---|---|
| Happy path: создал → достал → удалил | интеграционный |
| Валидация DTO: пустой `userName` → 400 | интеграционный (`MockMvc`) |
| Бизнес-конфликт: пересекающееся бронирование → 409 | интеграционный |
| Ресурс не найден: `GET /api/bookings/999` → 404 | интеграционный |
| Фильтр `?roomId=` | интеграционный |
| Хитрая бизнес-логика без БД | юнит, без Spring |

> Юнит-тесты быстрее в десятки раз, но контроллеры обычно проще тестировать через MockMvc.

---

## Мем №4: статика — это плохо

<img src="./lecture-06/imgs/this-is-fine.png" width="380" />

> Если ваш `BookingService` это `object` со статическим состоянием — у вас не сервис, а глобальная переменная с амбициями.

---

# Блок 10. Spring Boot vs Ktor

---

## Сравнение коротко

<style scoped>
table { font-size: 0.85em; }
</style>

| Критерий | Spring Boot | Ktor |
|---|---|---|
| Стиль | аннотации + контейнер | DSL + явная конфигурация |
| DI | встроенный IoC | подключают отдельно (Koin / руками) |
| Автоконфигурация | очень много из коробки | минимум, всё руками |
| Корутины | есть (`suspend` в контроллерах), но не нативно | нативно с самого старта |
| Экосистема | огромная (Data, Security, Cloud…) | скромнее, но Kotlin-first |
| Старт проекта | start.spring.io за минуту | пара десятков строк руками |
| Порог входа | выше: lifecycle, autoconfig, AOP | ниже: один Routing |
| Где живёт | enterprise, банки, корпорат | стартапы, Kotlin-first сервисы |

---

## Когда что выбирать

**Spring Boot:**
- большой бизнес-домен
- много интеграций (БД, очереди, безопасность, метрики)
- нужны транзакции, AOP, Spring Security
- команда в которой больше 2-х человек

**Ktor:**
- маленький Kotlin-first сервис
- хотите иметь полный контроль над каждой строчкой
- активно используете корутины
- готовы сами выбирать DI/serialization/persistence

> На 1-м курсе важнее изучить **Spring Boot** — он встретится вам везде. К Ktor вернётесь, когда будете писать pet-проекты в выходной.

---

## Что почитать дальше

- **docs.spring.io/spring-boot/reference** — официальная документация (начните с Getting Started).
- **start.spring.io** — генератор стартового проекта.
- **kotlinlang.org/docs/jvm-spring-boot-restful.html** — туториал JetBrains «Spring Boot + Kotlin».
- **baeldung.com** — лучший каталог практических статей по Spring.
- **«Spring in Action», 6th edition** — современная книга по Spring 6 / Boot 3.
- На практике-11 мы соберём всё это вживую: REST API бронирования переговорок.

---

# Блок 11. Q&A

---

## Вопросы для самопроверки

1. Что такое **IoC**? Как Spring инвертирует управление?
2. Чем `@Service` отличается от `@Component`?
3. Зачем в `build.gradle.kts` плагин `kotlin("plugin.spring")`?
4. Почему `JpaRepository` — **интерфейс**, а не класс?
5. Чем валидация (`@field:NotBlank`) отличается от **бизнес-конфликта** (`RoomAlreadyBookedException`)?
6. Как одной аннотацией мы говорим Spring «проверь этот DTO»?
7. Какие три механизма включает `@SpringBootApplication`?
8. Какой HTTP-код вы вернёте, если комната уже занята? Почему именно его?
9. Плюсы и минусы autoconfiguration: когда она помогает, а когда мешает?
10. Когда выбрать Spring Boot, а когда Ktor?

> Если на половину пунктов ответили уверенно — практика-11 пойдёт легко.

---

## Мем №5: вечная дилемма

<img src="./lecture-06/imgs/two-buttons-service.png" width="260" />

> Правда жизни: между `@Service` и `@Component` Spring почти не делает разницы — выбирайте по **семантике для читателя**.

---

# Спасибо!

## Вопросы?

- Сегодня на лекции: что такое Spring и как он устроен.
- На следующей практике (№11): пишем своё бронирование переговорок на Spring Boot 3.5.6 + Kotlin 2.2.20 + JPA + H2.
- Делитесь идеями для следующих лекций — это материал в активной разработке.
