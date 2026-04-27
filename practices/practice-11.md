# practice-11: Spring Boot, DI, REST API, validation, testing

## План

1. Что такое Spring Boot и как устроено приложение
2. Dependency Injection и IoC-контейнер
3. REST API на Spring MVC
4. DTO, валидация и обработка ошибок
5. Практический сервис бронирования переговорных комнат
6. Тестирование Spring-приложения
7. Вопросы
8. Итоги

---

## Что такое Spring Boot и как устроено приложение

### Зачем нужен Spring

`Spring` — это экосистема для разработки backend-приложений на JVM. В Kotlin он часто используется для REST API, микросервисов, внутренних корпоративных систем, интеграций с базами данных и очередями.

`Spring Boot` упрощает старт проекта: он автоматически настраивает веб-сервер, JSON-сериализацию, обработку HTTP-запросов, DI-контейнер и интеграции с другими библиотеками.

Практический сценарий этой практики: сделаем backend для бронирования переговорных комнат. Пользователь сможет посмотреть комнаты, создать бронирование и получить ошибку, если выбранное время уже занято.

### Из каких частей состоит Spring Boot-приложение

Обычно Spring Boot backend состоит из нескольких слоёв:

| Слой | Назначение |
|---|---|
| `Application` | точка входа приложения |
| `Controller` | HTTP-слой: принимает запросы и возвращает ответы |
| `Service` | бизнес-логика |
| `Repository` | доступ к данным |
| DTO | объекты request/response |
| Exception handler | единая обработка ошибок |

Минимальная точка входа:

```kotlin
import org.springframework.boot.autoconfigure.SpringBootApplication
import org.springframework.boot.runApplication

@SpringBootApplication
class BookingApplication

fun main(args: Array<String>) {
    runApplication<BookingApplication>(*args)
}
```

Аннотация `@SpringBootApplication` включает сразу несколько механизмов:
- поиск Spring-компонентов в текущем пакете и подпакетах;
- автоконфигурацию;
- регистрацию настроек приложения.

### Подключение зависимостей

Пример `build.gradle.kts` для простого Spring Boot REST API:

```kotlin
plugins {
    kotlin("jvm") version "2.2.20"
    kotlin("plugin.spring") version "2.2.20"
    id("org.springframework.boot") version "3.5.6"
    id("io.spring.dependency-management") version "1.1.7"
}

repositories {
    mavenCentral()
}

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

Что делает каждая зависимость:

| Зависимость | Зачем нужна |
|---|---|
| `spring-boot-starter-web` | Spring MVC, embedded Tomcat, JSON через Jackson |
| `spring-boot-starter-validation` | валидация DTO через Jakarta Validation |
| `spring-boot-starter-data-jpa` | Spring Data JPA, repositories и интеграция с Hibernate |
| `jackson-module-kotlin` | корректная сериализация Kotlin-классов |
| `kotlin-reflect` | reflection для Spring и Jackson |
| `h2` | локальная embedded database для разработки и тестов |
| `spring-boot-starter-test` | JUnit, MockMvc, assertions и тестовая инфраструктура |

### Сравнение с Ktor

| Вопрос | Spring Boot | Ktor |
|---|---|---|
| Стиль | аннотации и контейнер | DSL и явная конфигурация |
| DI | встроенный IoC-контейнер | обычно подключают отдельно |
| Автоконфигурация | много готового из коробки | больше ручной настройки |
| Порог входа | нужно понять lifecycle Spring | нужно понять routing и plugins |
| Типичный use case | enterprise backend, микросервисы | лёгкие API, Kotlin-first сервисы |

### Где Spring используется в реальных проектах

Spring часто выбирают, когда нужны:
- REST API с большим количеством endpoint-ов;
- интеграция с базами данных;
- транзакции;
- безопасность;
- метрики, health-checks, конфигурация окружений;
- понятная архитектура для больших команд.

### Упражнения: Spring Boot и структура приложения

1. Объясните своими словами, что делает `@SpringBootApplication`.
2. Создайте минимальное Spring Boot-приложение с функцией `main`.
3. Почему backend обычно делят на controller, service и repository, а не пишут всю логику в одном классе?
4. Сравните Spring Boot и Ktor: в каком фреймворке больше явной конфигурации, а в каком больше автоконфигурации?

---

## Dependency Injection и IoC-контейнер

### Что такое DI

`Dependency Injection` — это подход, при котором объект не создаёт свои зависимости вручную, а получает их снаружи.

Плохой вариант:

```kotlin
class BookingController {
    private val bookingService = BookingService(RoomRepository(), BookingRepository())
}
```

Проблемы:
- контроллер жёстко связан с конкретными реализациями;
- сложно тестировать;
- сложно заменить зависимость.

Spring-подход:

```kotlin
import org.springframework.web.bind.annotation.RestController

@RestController
class BookingController(
    private val bookingService: BookingService,
)
```

Spring сам создаст `BookingService` и передаст его в конструктор.

### Основные аннотации компонентов

| Аннотация | Где использовать |
|---|---|
| `@Component` | универсальный Spring-компонент |
| `@Service` | класс с бизнес-логикой |
| `@Repository` | класс доступа к данным |
| `@RestController` | HTTP-контроллер, который возвращает JSON |
| `@Configuration` | класс с ручной конфигурацией bean-ов |

Пример сервисного слоя:

```kotlin
import org.springframework.stereotype.Service

@Service
class BookingService(
    private val roomRepository: RoomRepository,
    private val bookingRepository: BookingRepository,
) {
    fun getRooms(): List<RoomResponse> = roomRepository.findAll().map(Room::toResponse)
}
```

Пример repository с локальной БД через Spring Data JPA:

```kotlin
import org.springframework.data.jpa.repository.JpaRepository

interface RoomRepository : JpaRepository<Room, Long>
```

Spring сам создаст реализацию этого интерфейса во время запуска приложения. Нам не нужно вручную писать `findAll()`, `findById()` или SQL для базовых операций.

### Constructor injection

В Kotlin для Spring обычно используют constructor injection:

```kotlin
@Service
class NotificationService(
    private val sender: EmailSender,
)
```

Почему это удобно:
- зависимости видны сразу в сигнатуре класса;
- поля можно сделать `val`;
- проще писать unit-тесты;
- меньше скрытого состояния.

### Kotlin и Spring: зачем нужен plugin.spring

Spring часто создаёт прокси-классы для `@Service`, `@Transactional`, `@Configuration`. В Kotlin классы и методы по умолчанию `final`, а Spring-прокси иногда требуют открытых классов.

Плагин `kotlin("plugin.spring")` автоматически делает нужные Spring-классы `open` там, где это требуется.

### Упражнения: DI и IoC

1. Напишите `RoomRepository` с методом `findAll()` и пометьте его подходящей Spring-аннотацией.
2. Напишите `RoomService`, который получает `RoomRepository` через конструктор.
3. Почему `constructor injection` обычно лучше, чем создание зависимостей через `new` или прямой вызов конструктора внутри класса?
4. Что произойдёт, если класс `BookingService` не пометить `@Service`, но попытаться внедрить его в `BookingController`?
5. Почему `interface RoomRepository : JpaRepository<Room, Long>` не нужно помечать `@Repository`?

---

## REST API на Spring MVC

### Controller как HTTP-слой

`Controller` отвечает за транспортный слой:
- принимает HTTP-запрос;
- читает path/query/body параметры;
- вызывает service;
- возвращает DTO и HTTP-код.

Пример простого контроллера:

```kotlin
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api/rooms")
class RoomController(
    private val bookingService: BookingService,
) {
    @GetMapping
    fun getRooms(): List<RoomResponse> = bookingService.getRooms()
}
```

`@RestController` означает, что результат метода будет записан в HTTP response body, обычно как JSON.

### Path parameters и query parameters

| Тип параметра | Пример | Когда использовать |
|---|---|---|
| Path parameter | `/api/bookings/42` | идентификатор конкретного ресурса |
| Query parameter | `/api/bookings?roomId=1` | фильтры, сортировка, пагинация |

Пример:

```kotlin
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.RequestParam

@GetMapping("/{id}")
fun getBookingById(@PathVariable id: Long): BookingResponse = bookingService.getBookingById(id)

@GetMapping
fun getBookings(@RequestParam roomId: Long?): List<BookingResponse> = bookingService.getBookings(roomId)
```

### Request body

Для JSON-тела запроса используется `@RequestBody`:

```kotlin
import org.springframework.http.HttpStatus
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.ResponseStatus

@PostMapping
@ResponseStatus(HttpStatus.CREATED)
fun createBooking(@RequestBody request: CreateBookingRequest): BookingResponse {
    return bookingService.createBooking(request)
}
```

### ResponseEntity

Если нужно явно управлять статусом и заголовками, можно использовать `ResponseEntity`:

```kotlin
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.DeleteMapping
import org.springframework.web.bind.annotation.PathVariable

@DeleteMapping("/{id}")
fun deleteBooking(@PathVariable id: Long): ResponseEntity<Unit> {
    bookingService.deleteBooking(id)
    return ResponseEntity.noContent().build()
}
```

### HTTP-коды для CRUD API

| Сценарий | Код |
|---|---|
| Успешное чтение | `200 OK` |
| Успешное создание | `201 Created` |
| Успешное удаление без тела | `204 No Content` |
| Ошибка валидации | `400 Bad Request` |
| Ресурс не найден | `404 Not Found` |
| Конфликт бизнес-правил | `409 Conflict` |

Для бронирований `409 Conflict` хорошо подходит, если комната уже занята в выбранный интервал.

### Упражнения: REST API на Spring MVC

1. Напишите `RoomController` с endpoint `GET /api/rooms`.
2. Напишите endpoint `GET /api/bookings/{id}`, который получает `id` через `@PathVariable`.
3. Чем `@RequestParam roomId: Long?` отличается от `@PathVariable id: Long`?
4. Какой HTTP-код лучше вернуть, если пользователь пытается забронировать уже занятую комнату? Объясните почему.

---

## DTO, валидация и обработка ошибок

### DTO для публичного API

DTO отделяют внешний API от внутренней модели. Это важно, потому что доменная модель может меняться по внутренним причинам, а контракт API должен быть стабильным.

```kotlin
import java.time.LocalDateTime

data class RoomResponse(
    val id: Long,
    val name: String,
    val capacity: Int,
)

data class CreateBookingRequest(
    val roomId: Long,
    val userName: String,
    val startsAt: LocalDateTime,
    val endsAt: LocalDateTime,
)

data class BookingResponse(
    val id: Long,
    val roomId: Long,
    val roomName: String,
    val userName: String,
    val startsAt: LocalDateTime,
    val endsAt: LocalDateTime,
)
```

### Jakarta Validation

Чтобы Spring автоматически проверял входные DTO, используются аннотации валидации и `@Valid`.

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

В Kotlin важно писать `@field:NotBlank`, а не просто `@NotBlank`, чтобы аннотация попала на поле, которое проверяет validation framework.

В контроллере:

```kotlin
import jakarta.validation.Valid
import org.springframework.web.bind.annotation.RequestBody

fun createBooking(@Valid @RequestBody request: CreateBookingRequest): BookingResponse {
    return bookingService.createBooking(request)
}
```

### Доменные исключения

Некоторые ошибки нельзя проверить только аннотациями. Например:
- комната не найдена;
- время окончания раньше времени начала;
- бронирование пересекается с существующим.

```kotlin
class RoomNotFoundException(roomId: Long) : RuntimeException("Room $roomId not found")
class BookingNotFoundException(bookingId: Long) : RuntimeException("Booking $bookingId not found")
class InvalidBookingTimeException : RuntimeException("Booking end time must be after start time")
class RoomAlreadyBookedException(roomId: Long) : RuntimeException("Room $roomId is already booked for this time")
```

### Единый формат ошибки

```kotlin
import java.time.Instant

data class ErrorResponse(
    val message: String,
    val timestamp: Instant = Instant.now(),
)
```

### `@RestControllerAdvice`

`@RestControllerAdvice` позволяет централизованно обрабатывать исключения для всех контроллеров.

```kotlin
import org.springframework.http.HttpStatus
import org.springframework.web.bind.MethodArgumentNotValidException
import org.springframework.web.bind.annotation.ExceptionHandler
import org.springframework.web.bind.annotation.ResponseStatus
import org.springframework.web.bind.annotation.RestControllerAdvice

@RestControllerAdvice
class ApiExceptionHandler {
    @ExceptionHandler(RoomNotFoundException::class, BookingNotFoundException::class)
    @ResponseStatus(HttpStatus.NOT_FOUND)
    fun handleNotFound(exception: RuntimeException): ErrorResponse {
        return ErrorResponse(message = exception.message ?: "Resource not found")
    }

    @ExceptionHandler(InvalidBookingTimeException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleInvalidTime(exception: InvalidBookingTimeException): ErrorResponse {
        return ErrorResponse(message = exception.message ?: "Invalid booking time")
    }

    @ExceptionHandler(RoomAlreadyBookedException::class)
    @ResponseStatus(HttpStatus.CONFLICT)
    fun handleConflict(exception: RoomAlreadyBookedException): ErrorResponse {
        return ErrorResponse(message = exception.message ?: "Booking conflict")
    }

    @ExceptionHandler(MethodArgumentNotValidException::class)
    @ResponseStatus(HttpStatus.BAD_REQUEST)
    fun handleValidation(exception: MethodArgumentNotValidException): ErrorResponse {
        val message = exception.bindingResult.fieldErrors.firstOrNull()?.defaultMessage
            ?: "Validation failed"
        return ErrorResponse(message = message)
    }
}
```

### Упражнения: DTO, валидация и ошибки

1. Добавьте в `CreateBookingRequest` поле `userName` и запретите пустое значение через validation annotation.
2. Почему для Kotlin DTO нужна форма `@field:NotBlank`?
3. Напишите исключение `RoomAlreadyBookedException` и обработайте его как `409 Conflict`.
4. Что лучше вернуть клиенту при ошибке: HTML-страницу, строку или JSON-объект? Объясните для REST API.
5. Как обработать случай, когда `endsAt <= startsAt`, если стандартной аннотации для этого нет?

---

## Практический сервис бронирования переговорных комнат

### Требования к сервису

Сервис должен поддерживать следующие endpoint-ы:

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/api/rooms` | получить список комнат |
| `GET` | `/api/bookings` | получить список бронирований, опционально с фильтром `roomId` |
| `GET` | `/api/bookings/{id}` | получить бронирование по id |
| `POST` | `/api/bookings` | создать бронирование |
| `DELETE` | `/api/bookings/{id}` | удалить бронирование |

Данные будем хранить не в `Map`, а в локальной H2 database. Это всё ещё удобно для практики: не нужно поднимать отдельный PostgreSQL, но код уже похож на реальный backend со слоем persistence.

### Настройка локальной H2 DB

Для учебного проекта можно добавить `application.yml`:

```yaml
spring:
  datasource:
    url: jdbc:h2:mem:booking-db
    driver-class-name: org.h2.Driver
    username: sa
    password:
  h2:
    console:
      enabled: true
      path: /h2-console
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true
```

Что здесь важно:
- `jdbc:h2:mem:booking-db` создаёт базу в памяти при запуске приложения;
- `/h2-console` позволяет посмотреть таблицы через браузер;
- `ddl-auto: update` просит Hibernate создать или обновить таблицы по entity-классам;
- `show-sql: true` показывает SQL-запросы в логах.

### JPA entity

```kotlin
import jakarta.persistence.Column
import jakarta.persistence.Entity
import jakarta.persistence.GeneratedValue
import jakarta.persistence.GenerationType
import jakarta.persistence.Id
import jakarta.persistence.Table
import java.time.LocalDateTime

@Entity
@Table(name = "rooms")
class Room(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false, unique = true)
    val name: String,

    @Column(nullable = false)
    val capacity: Int,
)

@Entity
@Table(name = "bookings")
class Booking(
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    val id: Long = 0,

    @Column(nullable = false)
    val roomId: Long,

    @Column(nullable = false)
    val userName: String,

    @Column(nullable = false)
    val startsAt: LocalDateTime,

    @Column(nullable = false)
    val endsAt: LocalDateTime,
)
```

### Маппинг в DTO

```kotlin
fun Room.toResponse(): RoomResponse = RoomResponse(
    id = id,
    name = name,
    capacity = capacity,
)

fun Booking.toResponse(room: Room): BookingResponse = BookingResponse(
    id = id,
    roomId = roomId,
    roomName = room.name,
    userName = userName,
    startsAt = startsAt,
    endsAt = endsAt,
)
```

### Spring Data JPA repositories

`RoomRepository` и `BookingRepository` в реальном Spring-приложении чаще всего являются интерфейсами. Spring Data JPA генерирует реализацию автоматически.

```kotlin
import org.springframework.data.jpa.repository.JpaRepository

interface RoomRepository : JpaRepository<Room, Long>

interface BookingRepository : JpaRepository<Booking, Long> {
    fun findAllByRoomId(roomId: Long): List<Booking>
}
```

Что мы получили бесплатно:
- `findAll()`;
- `findById(id)`;
- `save(entity)`;
- `deleteById(id)`;
- `existsById(id)`.

Метод `findAllByRoomId` Spring Data JPA разберёт по имени и превратит в SQL-запрос по полю `roomId`.

### Стартовые данные

Чтобы при запуске приложения в базе уже были комнаты, можно добавить `CommandLineRunner`:

```kotlin
import org.springframework.boot.CommandLineRunner
import org.springframework.context.annotation.Bean
import org.springframework.context.annotation.Configuration

@Configuration
class DemoDataConfiguration {
    @Bean
    fun demoRooms(roomRepository: RoomRepository): CommandLineRunner {
        return CommandLineRunner {
            if (roomRepository.count() == 0L) {
                roomRepository.saveAll(
                    listOf(
                        Room(name = "Aurora", capacity = 6),
                        Room(name = "Orion", capacity = 12),
                        Room(name = "Vega", capacity = 4),
                    ),
                )
            }
        }
    }
}
```

### Проверка пересечения бронирований

Два временных интервала пересекаются, если начало одного раньше конца другого, а конец одного позже начала другого.

```kotlin
private fun Booking.overlaps(startsAt: LocalDateTime, endsAt: LocalDateTime): Boolean {
    return this.startsAt < endsAt && this.endsAt > startsAt
}
```

Примеры:

| Существующее бронирование | Новое бронирование | Пересекаются? |
|---|---|---|
| `10:00-11:00` | `10:30-11:30` | да |
| `10:00-11:00` | `11:00-12:00` | нет |
| `10:00-11:00` | `09:00-10:00` | нет |
| `10:00-11:00` | `09:30-10:30` | да |

### Service-слой

```kotlin
import org.springframework.stereotype.Service

@Service
class BookingService(
    private val roomRepository: RoomRepository,
    private val bookingRepository: BookingRepository,
) {
    fun getRooms(): List<RoomResponse> = roomRepository.findAll().map(Room::toResponse)

    fun getBookings(roomId: Long?): List<BookingResponse> {
        val bookings = if (roomId == null) {
            bookingRepository.findAll()
        } else {
            bookingRepository.findAllByRoomId(roomId)
        }

        return bookings.map { booking ->
            val room = roomRepository.findById(booking.roomId).orElseThrow { RoomNotFoundException(booking.roomId) }
            booking.toResponse(room)
        }
    }

    fun getBookingById(id: Long): BookingResponse {
        val booking = bookingRepository.findById(id).orElseThrow { BookingNotFoundException(id) }
        val room = roomRepository.findById(booking.roomId).orElseThrow { RoomNotFoundException(booking.roomId) }
        return booking.toResponse(room)
    }

    fun createBooking(request: CreateBookingRequest): BookingResponse {
        val room = roomRepository.findById(request.roomId).orElseThrow { RoomNotFoundException(request.roomId) }
        validateBookingTime(request.startsAt, request.endsAt)
        validateRoomIsAvailable(request.roomId, request.startsAt, request.endsAt)

        val booking = bookingRepository.save(
            Booking(
                roomId = request.roomId,
                userName = request.userName.trim(),
                startsAt = request.startsAt,
                endsAt = request.endsAt,
            ),
        )

        return booking.toResponse(room)
    }

    fun deleteBooking(id: Long) {
        if (!bookingRepository.existsById(id)) {
            throw BookingNotFoundException(id)
        }
        bookingRepository.deleteById(id)
    }

    private fun validateBookingTime(startsAt: LocalDateTime, endsAt: LocalDateTime) {
        if (!endsAt.isAfter(startsAt)) {
            throw InvalidBookingTimeException()
        }
    }

    private fun validateRoomIsAvailable(roomId: Long, startsAt: LocalDateTime, endsAt: LocalDateTime) {
        val hasConflict = bookingRepository.findAllByRoomId(roomId)
            .any { booking -> booking.overlaps(startsAt, endsAt) }

        if (hasConflict) {
            throw RoomAlreadyBookedException(roomId)
        }
    }
}
```

### Controller

```kotlin
import jakarta.validation.Valid
import org.springframework.http.HttpStatus
import org.springframework.http.ResponseEntity
import org.springframework.web.bind.annotation.DeleteMapping
import org.springframework.web.bind.annotation.GetMapping
import org.springframework.web.bind.annotation.PathVariable
import org.springframework.web.bind.annotation.PostMapping
import org.springframework.web.bind.annotation.RequestBody
import org.springframework.web.bind.annotation.RequestMapping
import org.springframework.web.bind.annotation.RequestParam
import org.springframework.web.bind.annotation.ResponseStatus
import org.springframework.web.bind.annotation.RestController

@RestController
@RequestMapping("/api")
class BookingController(
    private val bookingService: BookingService,
) {
    @GetMapping("/rooms")
    fun getRooms(): List<RoomResponse> = bookingService.getRooms()

    @GetMapping("/bookings")
    fun getBookings(@RequestParam roomId: Long?): List<BookingResponse> {
        return bookingService.getBookings(roomId)
    }

    @GetMapping("/bookings/{id}")
    fun getBookingById(@PathVariable id: Long): BookingResponse {
        return bookingService.getBookingById(id)
    }

    @PostMapping("/bookings")
    @ResponseStatus(HttpStatus.CREATED)
    fun createBooking(@Valid @RequestBody request: CreateBookingRequest): BookingResponse {
        return bookingService.createBooking(request)
    }

    @DeleteMapping("/bookings/{id}")
    fun deleteBooking(@PathVariable id: Long): ResponseEntity<Unit> {
        bookingService.deleteBooking(id)
        return ResponseEntity.noContent().build()
    }
}
```

### Примеры запросов

Создание бронирования:

```http
POST /api/bookings
Content-Type: application/json

{
  "roomId": 1,
  "userName": "Anna",
  "startsAt": "2030-04-10T10:00:00",
  "endsAt": "2030-04-10T11:00:00"
}
```

Ответ:

```json
{
  "id": 1,
  "roomId": 1,
  "roomName": "Aurora",
  "userName": "Anna",
  "startsAt": "2030-04-10T10:00:00",
  "endsAt": "2030-04-10T11:00:00"
}
```

Попытка создать пересекающееся бронирование:

```http
POST /api/bookings
Content-Type: application/json

{
  "roomId": 1,
  "userName": "Sergey",
  "startsAt": "2030-04-10T10:30:00",
  "endsAt": "2030-04-10T11:30:00"
}
```

Ответ:

```json
{
  "message": "Room 1 is already booked for this time",
  "timestamp": "2030-04-10T09:00:00Z"
}
```

### Упражнения: сервис бронирования

1. Подключите `spring-boot-starter-data-jpa` и `h2`, настройте `application.yml` для локальной H2 DB.
2. Опишите `Room` и `Booking` как JPA entity.
3. Реализуйте `RoomRepository` и `BookingRepository` как интерфейсы-наследники `JpaRepository`.
4. Добавьте стартовые комнаты через `CommandLineRunner`.
5. Реализуйте `POST /api/bookings` с проверкой существования комнаты.
6. Добавьте проверку, что `endsAt` строго позже `startsAt`.
7. Реализуйте проверку пересечения бронирований для одной комнаты через `findAllByRoomId`.
8. Edge case: разрешено ли бронирование `10:00-11:00`, если уже есть бронирование `11:00-12:00`? Объясните и проверьте тестом.

---

## Тестирование Spring-приложения

### Integration-тест service-слоя с локальной H2 DB

Так как repositories теперь работают через Spring Data JPA, полезно проверить service вместе с локальной H2 DB.

```kotlin
import java.time.LocalDateTime
import kotlin.test.Test
import kotlin.test.assertFailsWith
import org.junit.jupiter.api.BeforeEach
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.context.SpringBootTest

@SpringBootTest
class BookingServiceTest(
    @Autowired private val roomRepository: RoomRepository,
    @Autowired private val bookingRepository: BookingRepository,
    @Autowired private val bookingService: BookingService,
) {
    @BeforeEach
    fun setUp() {
        bookingRepository.deleteAll()
        roomRepository.deleteAll()
        roomRepository.save(Room(name = "Aurora", capacity = 6))
    }

    @Test
    fun `create booking rejects overlapping interval`() {
        val room = roomRepository.findAll().single()
        val startsAt = LocalDateTime.parse("2030-04-10T10:00:00")
        val endsAt = LocalDateTime.parse("2030-04-10T11:00:00")

        bookingService.createBooking(
            CreateBookingRequest(
                roomId = room.id,
                userName = "Anna",
                startsAt = startsAt,
                endsAt = endsAt,
            ),
        )

        assertFailsWith<RoomAlreadyBookedException> {
            bookingService.createBooking(
                CreateBookingRequest(
                    roomId = room.id,
                    userName = "Sergey",
                    startsAt = LocalDateTime.parse("2030-04-10T10:30:00"),
                    endsAt = LocalDateTime.parse("2030-04-10T11:30:00"),
                ),
            )
        }
    }
}
```

### Web-тест контроллера через MockMvc

`MockMvc` позволяет отправить HTTP-запрос в Spring MVC без реального сетевого порта.

```kotlin
import org.junit.jupiter.api.Test
import org.springframework.beans.factory.annotation.Autowired
import org.springframework.boot.test.autoconfigure.web.servlet.AutoConfigureMockMvc
import org.springframework.boot.test.context.SpringBootTest
import org.springframework.http.MediaType
import org.springframework.test.web.servlet.MockMvc
import org.springframework.test.web.servlet.post

@SpringBootTest
@AutoConfigureMockMvc
class BookingControllerTest(
    @Autowired private val mockMvc: MockMvc,
) {
    @Test
    fun `create booking returns 201`() {
        mockMvc.post("/api/bookings") {
            contentType = MediaType.APPLICATION_JSON
            content = """
                {
                  "roomId": 1,
                  "userName": "Anna",
                  "startsAt": "2030-04-10T10:00:00",
                  "endsAt": "2030-04-10T11:00:00"
                }
            """.trimIndent()
        }.andExpect {
            status { isCreated() }
            jsonPath("$.roomName") { value("Aurora") }
            jsonPath("$.userName") { value("Anna") }
        }
    }
}
```

### Что стоит тестировать

| Что тестируем | Пример |
|---|---|
| happy path | бронирование успешно создаётся и сохраняется в H2 |
| валидацию DTO | `userName` пустой |
| бизнес-валидацию | `endsAt <= startsAt` |
| конфликт | пересечение бронирований |
| not found | комната или бронирование не найдены |
| фильтрацию | `GET /api/bookings?roomId=1` |
| persistence | `BookingRepository.findAllByRoomId(roomId)` возвращает нужные записи |

### Упражнения: тестирование

1. Напишите integration-тест, который проверяет, что бронирование соседнего интервала `11:00-12:00` после `10:00-11:00` разрешено.
2. Напишите тест на `InvalidBookingTimeException`.
3. Напишите `MockMvc`-тест для `GET /api/rooms`.
4. Напишите `MockMvc`-тест, который проверяет `400 Bad Request` при пустом `userName`.
5. Напишите repository-тест для `findAllByRoomId`.
6. Почему тест с `@SpringBootTest` обычно медленнее, чем unit-тест без Spring-контекста?

---

## Вопросы

1. В чём основная идея IoC-контейнера Spring?
2. Почему бизнес-логику лучше держать в `@Service`, а не в `@RestController`?
3. Чем ошибка валидации отличается от бизнес-конфликта?
4. Почему пересечение временных интервалов — это бизнес-правило, а не задача controller-а?
5. Почему Spring Data JPA repository может быть интерфейсом без собственной реализации?
6. Чем локальная H2 DB удобна для практики и чем она отличается от production PostgreSQL?
7. Какие плюсы и минусы у автоконфигурации Spring Boot?

---

## Итоги

1. Spring Boot позволяет быстро поднять JVM backend с REST API, JSON и встроенным web-сервером.
2. DI помогает явно описывать зависимости и упрощает тестирование классов.
3. Controller должен отвечать за HTTP-слой, а service — за бизнес-правила.
4. DTO и централизованная обработка ошибок делают API стабильнее и удобнее для клиентов.
5. Spring Data JPA позволяет заменить ручные in-memory repositories на реальные repositories поверх локальной DB.
6. Проверка пересечений бронирований показывает, как реальные бизнес-правила живут в service-слое.
7. Integration-тесты с H2 и `MockMvc`-тесты проверяют разные уровни приложения и дополняют друг друга.
