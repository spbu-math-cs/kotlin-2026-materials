# practice-10: Ktor Server, routing, request/response, ContentNegotiation, StatusPages, REST API

## План

1. Что такое Ktor Server и как устроено Ktor-приложение
2. Routing и объект `ApplicationCall`
3. Request/Response и работа с JSON через `ContentNegotiation`
4. Ошибки, валидация и `StatusPages`
5. Практический RESTful сервис для менеджмента задач
6. Тестирование API
7. Вопросы
8. Итоги

---

## Что такое Ktor Server и как устроено Ktor-приложение

### Зачем нужен Ktor

`Ktor` — это Kotlin-фреймворк для написания сетевых приложений. В контексте backend-разработки он часто используется для создания HTTP API, внутренних сервисов, webhook-обработчиков и небольших микросервисов.

Почему Ktor удобен:
- он написан на Kotlin и хорошо сочетается с корутинами;
- у него минималистичное API без тяжёлой магии;
- можно явно контролировать маршруты, плагины и сериализацию;
- удобно писать как маленькие учебные сервисы, так и production-сервисы.

Практический сценарий: нам нужен небольшой backend для управления задачами команды. Клиент отправляет HTTP-запросы, а сервер создаёт, читает, обновляет и удаляет задачи.

### Из каких частей состоит Ktor-приложение

Обычно в Ktor Server есть несколько основных слоёв:

| Часть | Назначение |
|---|---|
| `Application.module()` | Точка входа конфигурации сервера |
| plugins | Подключение сериализации, обработки ошибок, логирования и т.д. |
| routing | Описание HTTP-маршрутов |
| service/repository | Бизнес-логика и доступ к данным |
| DTO | Объекты запроса и ответа |

Минимальный пример:

```kotlin
import io.ktor.server.application.Application
import io.ktor.server.application.call
import io.ktor.server.response.respondText
import io.ktor.server.routing.get
import io.ktor.server.routing.routing

fun Application.module() {
    routing {
        get("/") {
            call.respondText("Hello, Ktor!")
        }
    }
}
```

Здесь:
- `module()` — функция, в которой мы настраиваем приложение;
- `routing { ... }` — блок маршрутов;
- `get("/")` — обработчик GET-запроса;
- `call` — текущий HTTP-запрос/ответ.

### Почему Ktor хорошо сочетается с Kotlin

Ktor строится вокруг Kotlin-идиом:
- DSL для роутинга;
- data classes для DTO;
- корутины для неблокирующей обработки;
- `kotlinx.serialization` для JSON.

Пример DTO:

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class TaskResponse(
    val id: Long,
    val title: String,
    val completed: Boolean,
)
```

### Сравнение с Java-подходом

Если сравнить с типичным Java backend-фреймворком, то в Ktor обычно меньше аннотаций и больше явной конфигурации в коде.

| Подход | Ktor | Типичный Java annotation-based стиль |
|---|---|---|
| Описание маршрутов | DSL `routing { get/post/... }` | Аннотации вроде `@GetMapping` |
| Конфигурация | Явно в `module()` | Часто через аннотации + автоконфиг |
| Стиль | Ближе к Kotlin DSL | Ближе к декларативным Java-аннотациям |
| Порог понимания | Нужно понять DSL и плагины | Нужно понять аннотации и lifecycle контейнера |

### Где это используется в реальных проектах

Ktor часто подходит для:
- внутренних API между сервисами;
- CRUD-сервисов;
- webhook endpoints;
- BFF (backend for frontend);
- учебных и prototyping-проектов.

### Подключение зависимостей

Чтобы запустить Ktor Server-проект, обычно нужны:
- Kotlin-плагин;
- плагин сериализации;
- Ktor plugin для Gradle;
- зависимости на server core, Netty engine, ContentNegotiation, JSON serialization и тестирование.

На момент подготовки этой практики в документации Ktor указана версия `3.4.2`.

### Пример `build.gradle.kts`

```kotlin
plugins {
    kotlin("jvm") version "2.2.20"
    kotlin("plugin.serialization") version "2.2.20"
    id("io.ktor.plugin") version "3.4.2"
}

repositories {
    mavenCentral()
}

group = "org.example"
version = "0.0.1"

application {
    mainClass = "io.ktor.server.netty.EngineMain"
}

dependencies {
    implementation("io.ktor:ktor-server-core:3.4.2")
    implementation("io.ktor:ktor-server-netty:3.4.2")
    implementation("io.ktor:ktor-server-content-negotiation:3.4.2")
    implementation("io.ktor:ktor-serialization-kotlinx-json:3.4.2")
    implementation("ch.qos.logback:logback-classic:1.5.18")

    testImplementation("io.ktor:ktor-server-test-host:3.4.2")
    testImplementation(kotlin("test"))
}
```

### Что делает каждая зависимость

| Зависимость | Зачем нужна |
|---|---|
| `ktor-server-core` | Базовые API Ktor Server |
| `ktor-server-netty` | HTTP engine для запуска приложения |
| `ktor-server-content-negotiation` | Поддержка сериализации/десериализации тела запроса и ответа |
| `ktor-serialization-kotlinx-json` | JSON через `kotlinx.serialization` |
| `ktor-server-test-host` | Тестирование API через `testApplication` |
| `logback-classic` | Логи приложения |

### Почему версии лучше держать согласованными

У Ktor-модулей обычно должна быть одна и та же версия. Если смешать разные версии `ktor-server-core`, `ktor-server-netty` и плагинов, можно получить ошибки совместимости на этапе запуска или тестов.

Для учебного проекта есть два нормальных подхода:
- явно указывать одну и ту же версию во всех Ktor-зависимостях;
- или использовать Ktor Gradle plugin, который помогает держать конфигурацию согласованной.

### Упражнения: Ktor Server и структура приложения

1. Объясните своими словами, чем `Application.module()` отличается от обычной функции Kotlin.
2. Посмотрите на код ниже и перечислите, какие части отвечают за конфигурацию сервера, а какие — за бизнес-логику:

    ```kotlin
    fun Application.module() {
        install(ContentNegotiation) {
            json()
        }

        routing {
            get("/ping") {
                call.respond(mapOf("status" to "ok"))
            }
        }
    }
    ```

3. Напишите минимальный Ktor-модуль с endpoint `GET /health`, который отвечает строкой `OK`.
4. Письменно сравните Ktor и Spring Boot по ощущениям разработчика: где меньше магии, а где больше готовой инфраструктуры?

---

## Routing и объект `ApplicationCall`

### Что такое routing

Routing — это механизм сопоставления HTTP-запроса с обработчиком. Маршрут обычно определяется:
- HTTP-методом (`GET`, `POST`, `PATCH`, `DELETE`);
- путём (`/tasks`, `/tasks/{id}`);
- иногда дополнительными условиями.

Пример:

```kotlin
import io.ktor.server.application.call
import io.ktor.server.response.respondText
import io.ktor.server.routing.delete
import io.ktor.server.routing.get
import io.ktor.server.routing.patch
import io.ktor.server.routing.post
import io.ktor.server.routing.routing

fun Application.configureRouting() {
    routing {
        get("/tasks") {
            call.respondText("all tasks")
        }

        post("/tasks") {
            call.respondText("create task")
        }

        patch("/tasks/{id}") {
            call.respondText("update task")
        }

        delete("/tasks/{id}") {
            call.respondText("delete task")
        }
    }
}
```

### Группировка маршрутов

Чтобы структура API была читаемой, маршруты обычно группируют:

```kotlin
import io.ktor.server.routing.route
import io.ktor.server.routing.routing

fun Application.configureRouting() {
    routing {
        route("/tasks") {
            get {
                // GET /tasks
            }

            get("/{id}") {
                // GET /tasks/{id}
            }

            post {
                // POST /tasks
            }
        }
    }
}
```

Такой стиль особенно полезен, когда маршрутов много.

### Что такое `ApplicationCall`

`ApplicationCall` — объект, представляющий текущий HTTP-вызов. Через него мы получаем доступ к:
- параметрам пути;
- query-параметрам;
- заголовкам;
- телу запроса;
- ответу.

Самое частое, что используют на практике:

```kotlin
val id = call.parameters["id"]
val sort = call.request.queryParameters["sort"]
val requestId = call.request.headers["X-Request-Id"]
```

### Path parameters и query parameters

Разница:

| Тип параметра | Пример | Когда использовать |
|---|---|---|
| Path parameter | `/tasks/42` | Идентификатор конкретного ресурса |
| Query parameter | `/tasks?completed=true` | Фильтры, пагинация, сортировка |

Пример чтения параметров:

```kotlin
import io.ktor.http.HttpStatusCode
import io.ktor.server.application.call
import io.ktor.server.response.respond
import io.ktor.server.routing.get

get("/tasks/{id}") {
    val id = call.parameters["id"]?.toLongOrNull()
        ?: return@get call.respond(HttpStatusCode.BadRequest, "Invalid task id")

    call.respond(mapOf("taskId" to id))
}

get("/tasks") {
    val completed = call.request.queryParameters["completed"]?.toBooleanStrictOrNull()
    call.respond(mapOf("completed" to completed))
}
```

### Реальный смысл роутинга

В production-сервисе routing часто отвечает только за HTTP-слой:
- разобрать входной запрос;
- вызвать сервис;
- вернуть HTTP-ответ.

Бизнес-логика не должна жить прямо в route-блоке, иначе код быстро превратится в трудно поддерживаемый контроллер с большим количеством ветвлений.

Плохой стиль:

```kotlin
post("/tasks") {
    val body = call.receive<CreateTaskRequest>()

    if (body.title.isBlank()) {
        call.respond(HttpStatusCode.BadRequest, "title is blank")
        return@post
    }

    // здесь же работа с данными, поиск, валидация, правила домена, логирование...
}
```

Лучше так:

```kotlin
post("/tasks") {
    val body = call.receive<CreateTaskRequest>()
    val created = taskService.create(body)
    call.respond(HttpStatusCode.Created, created)
}
```

### Упражнения: routing и `ApplicationCall`

1. Напишите endpoint `GET /tasks/{id}`, который:
   - читает `id` из path parameter;
   - возвращает `400 Bad Request`, если `id` не число;
   - иначе возвращает JSON с этим `id`.
2. Чем path parameter отличается от query parameter на примере API задач?
3. Что выведет сервер в каждом случае?

    ```kotlin
    get("/tasks/{id}") {
        println(call.parameters["id"])
    }
    ```

    Для запросов:
   - `GET /tasks/10`
   - `GET /tasks/abc`
   - `GET /tasks`

4. Перепишите набор маршрутов для `/users`, используя `route("/users") { ... }` вместо отдельных путей.

---

## Request/Response и работа с JSON через `ContentNegotiation`

### Зачем нужен `ContentNegotiation`

REST API почти всегда обменивается JSON. Чтобы Ktor умел:
- читать JSON из request body;
- превращать Kotlin-объекты в JSON в response body,

нужно подключить плагин `ContentNegotiation`.

Пример настройки:

```kotlin
import io.ktor.serialization.kotlinx.json.json
import io.ktor.server.application.Application
import io.ktor.server.application.install
import io.ktor.server.plugins.contentnegotiation.ContentNegotiation

fun Application.configureSerialization() {
    install(ContentNegotiation) {
        json()
    }
}
```

### DTO для запроса и ответа

Для REST API удобно разделять:
- внутреннюю доменную модель;
- объекты запроса;
- объекты ответа.

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class CreateTaskRequest(
    val title: String,
    val description: String? = null,
)

@Serializable
data class UpdateTaskRequest(
    val title: String? = null,
    val description: String? = null,
    val completed: Boolean? = null,
)

@Serializable
data class TaskResponse(
    val id: Long,
    val title: String,
    val description: String?,
    val completed: Boolean,
)
```

Такой подход важен в реальных проектах: доменная сущность меняется по своим причинам, а публичный API должен быть стабильнее.

### Получение request body

Через `call.receive<T>()` можно десериализовать JSON в Kotlin-объект:

```kotlin
import io.ktor.http.HttpStatusCode
import io.ktor.server.application.call
import io.ktor.server.request.receive
import io.ktor.server.response.respond
import io.ktor.server.routing.post

post("/tasks") {
    val request = call.receive<CreateTaskRequest>()

    val response = TaskResponse(
        id = 1L,
        title = request.title,
        description = request.description,
        completed = false,
    )

    call.respond(HttpStatusCode.Created, response)
}
```

Если клиент прислал JSON неправильной структуры, Ktor выбросит исключение десериализации. Обычно такие случаи обрабатывают через `StatusPages`.

### Возврат ответа

Сервер отвечает через `call.respond(...)`.

```kotlin
get("/tasks") {
    val tasks = listOf(
        TaskResponse(1, "Learn Ktor", "Read routing and plugins", false),
        TaskResponse(2, "Write REST API", null, true),
    )

    call.respond(tasks)
}
```

При подключённом `ContentNegotiation` список автоматически превратится в JSON.

### HTTP-коды ответа

Для CRUD API полезно придерживаться стандартных кодов:

| Сценарий | Код |
|---|---|
| Успешное чтение ресурса | `200 OK` |
| Успешное создание | `201 Created` |
| Успешное удаление без тела | `204 No Content` |
| Некорректный запрос | `400 Bad Request` |
| Ресурс не найден | `404 Not Found` |

Пример удаления:

```kotlin
import io.ktor.http.HttpStatusCode
import io.ktor.server.response.respond
import io.ktor.server.routing.delete

delete("/tasks/{id}") {
    val deleted = true

    if (deleted) {
        call.respond(HttpStatusCode.NoContent)
    } else {
        call.respond(HttpStatusCode.NotFound)
    }
}
```

### Java vs Kotlin DTO

В Kotlin DTO обычно компактнее за счёт `data class`.

| Kotlin | Java |
|---|---|
| `data class CreateTaskRequest(val title: String)` | отдельный класс, поля, конструктор, геттеры, `equals/hashCode` |
| nullable-типы встроены в типовую систему | nullability часто выражается только договорённостями или аннотациями |

### Практическое замечание про API-дизайн

Даже в учебном сервисе полезно сразу придерживаться правил:
- не возвращать «сырые» строки там, где ожидается JSON;
- использовать явные статус-коды;
- отделять DTO от внутренней модели;
- не смешивать транспортный слой и бизнес-логику.

### Упражнения: request/response и JSON

1. Создайте `@Serializable` класс `CreateTaskRequest` с полями `title` и `description`.
2. Напишите endpoint `POST /tasks`, который принимает JSON и отвечает объектом задачи со сгенерированным `id`.
3. Какой статус-код логичнее вернуть в каждом случае?
   - задача успешно создана;
   - клиент прислал `title = ""`;
   - задача с таким `id` не найдена.
4. Объясните, зачем разделять `CreateTaskRequest` и `TaskResponse`, даже если поля почти одинаковые.

---

## Ошибки, валидация и `StatusPages`

### Зачем нужна централизованная обработка ошибок

Если в каждом route писать отдельные `try/catch`, код быстро станет шумным. В Ktor для этого используют `StatusPages`.

Этот плагин помогает:
- перехватывать исключения;
- маппить их в HTTP-ответы;
- централизованно формировать JSON-ошибки.

### Модель ошибки

Обычно API отвечает не просто строкой, а структурированным JSON:

```kotlin
import kotlinx.serialization.Serializable

@Serializable
data class ErrorResponse(
    val message: String,
)
```

### Свои исключения

Для читаемого кода полезно завести доменные исключения:

```kotlin
class TaskNotFoundException(taskId: Long) : RuntimeException("Task $taskId not found")
class ValidationException(message: String) : RuntimeException(message)
```

### Настройка `StatusPages`

```kotlin
import io.ktor.http.HttpStatusCode
import io.ktor.server.application.Application
import io.ktor.server.application.install
import io.ktor.server.plugins.statuspages.StatusPages
import io.ktor.server.response.respond

fun Application.configureStatusPages() {
    install(StatusPages) {
        exception<ValidationException> { call, cause ->
            call.respond(HttpStatusCode.BadRequest, ErrorResponse(cause.message ?: "Validation failed"))
        }

        exception<TaskNotFoundException> { call, cause ->
            call.respond(HttpStatusCode.NotFound, ErrorResponse(cause.message ?: "Task not found"))
        }

        exception<Throwable> { call, cause ->
            call.respond(HttpStatusCode.InternalServerError, ErrorResponse(cause.message ?: "Unexpected error"))
        }
    }
}
```

### Простая валидация входных данных

Даже если сервис маленький, валидировать вход нужно сразу. Например, title задачи не должен быть пустым.

```kotlin
private const val MIN_TITLE_LENGTH = 3

fun validateCreateTask(request: CreateTaskRequest) {
    if (request.title.isBlank()) {
        throw ValidationException("Title must not be blank")
    }

    if (request.title.length < MIN_TITLE_LENGTH) {
        throw ValidationException("Title must contain at least $MIN_TITLE_LENGTH characters")
    }
}
```

Обратите внимание: строковое число `3` вынесено в именованную константу — это лучше, чем магическое значение в условии.

### Почему это важно в реальном API

Хорошая обработка ошибок нужна потому что:
- клиенту нужен предсказуемый формат ответа;
- frontend и другие сервисы зависят от стабильных кодов и сообщений;
- централизованная обработка упрощает поддержку.

### Упражнения: ошибки и `StatusPages`

1. Создайте класс `ValidationException` и настройте `StatusPages`, чтобы он возвращал `400 Bad Request`.
2. Что лучше вернуть клиенту: строку ошибки или JSON-объект `ErrorResponse`? Объясните почему.
3. Найдите проблему в коде:

    ```kotlin
    post("/tasks") {
        val request = call.receive<CreateTaskRequest>()
        if (request.title.isBlank()) {
            throw RuntimeException("bad input")
        }
        call.respond(HttpStatusCode.Created)
    }
    ```

    Какие здесь есть недостатки с точки зрения API-дизайна?

4. Добавьте правило: описание задачи не должно быть длиннее 500 символов. Какой код и текст ошибки вы вернёте при нарушении?

---

## Практический RESTful сервис для менеджмента задач

### Требования к сервису

Сделаем простой сервис управления задачами со следующими endpoint-ами:

| Метод | Путь | Назначение |
|---|---|---|
| `GET` | `/tasks` | Получить все задачи |
| `GET` | `/tasks/{id}` | Получить одну задачу |
| `POST` | `/tasks` | Создать задачу |
| `PATCH` | `/tasks/{id}` | Частично обновить задачу |
| `DELETE` | `/tasks/{id}` | Удалить задачу |

Для простоты хранить данные будем в памяти, без базы данных. Это хороший учебный этап: сначала отрабатываем HTTP API и архитектуру, потом подключаем БД.

### Доменные модели и DTO

```kotlin
import kotlinx.serialization.Serializable

private const val INITIAL_TASK_ID = 1L

data class Task(
    val id: Long,
    val title: String,
    val description: String?,
    val completed: Boolean,
)

@Serializable
data class CreateTaskRequest(
    val title: String,
    val description: String? = null,
)

@Serializable
data class UpdateTaskRequest(
    val title: String? = null,
    val description: String? = null,
    val completed: Boolean? = null,
)

@Serializable
data class TaskResponse(
    val id: Long,
    val title: String,
    val description: String?,
    val completed: Boolean,
)

fun Task.toResponse(): TaskResponse = TaskResponse(
    id = id,
    title = title,
    description = description,
    completed = completed,
)
```

### In-memory repository

```kotlin
class TaskRepository {
    private val tasks = linkedMapOf<Long, Task>()
    private var nextId = INITIAL_TASK_ID

    fun findAll(): List<Task> = tasks.values.toList()

    fun findById(id: Long): Task? = tasks[id]

    fun create(title: String, description: String?): Task {
        val task = Task(
            id = nextId++,
            title = title,
            description = description,
            completed = false,
        )
        tasks[task.id] = task
        return task
    }

    fun update(id: Long, task: Task): Task {
        tasks[id] = task
        return task
    }

    fun delete(id: Long): Boolean = tasks.remove(id) != null
}
```

Почему `linkedMapOf`:
- порядок элементов будет предсказуемым;
- на практике это удобно для учебных примеров и тестов.

### Service-слой

```kotlin
private const val MIN_TITLE_LENGTH = 3
private const val MAX_DESCRIPTION_LENGTH = 500

class TaskService(
    private val repository: TaskRepository,
) {
    fun getAll(): List<TaskResponse> = repository.findAll().map(Task::toResponse)

    fun getById(id: Long): TaskResponse {
        val task = repository.findById(id) ?: throw TaskNotFoundException(id)
        return task.toResponse()
    }

    fun create(request: CreateTaskRequest): TaskResponse {
        validateTitle(request.title)
        validateDescription(request.description)

        return repository
            .create(
                title = request.title.trim(),
                description = request.description?.trim(),
            )
            .toResponse()
    }

    fun update(id: Long, request: UpdateTaskRequest): TaskResponse {
        val current = repository.findById(id) ?: throw TaskNotFoundException(id)

        val newTitle = request.title?.trim() ?: current.title
        val newDescription = request.description?.trim() ?: current.description
        val newCompleted = request.completed ?: current.completed

        validateTitle(newTitle)
        validateDescription(newDescription)

        val updated = current.copy(
            title = newTitle,
            description = newDescription,
            completed = newCompleted,
        )

        return repository.update(id, updated).toResponse()
    }

    fun delete(id: Long) {
        val deleted = repository.delete(id)
        if (!deleted) {
            throw TaskNotFoundException(id)
        }
    }

    private fun validateTitle(title: String) {
        if (title.isBlank()) {
            throw ValidationException("Title must not be blank")
        }
        if (title.length < MIN_TITLE_LENGTH) {
            throw ValidationException("Title must contain at least $MIN_TITLE_LENGTH characters")
        }
    }

    private fun validateDescription(description: String?) {
        if (description != null && description.length > MAX_DESCRIPTION_LENGTH) {
            throw ValidationException("Description must be shorter than $MAX_DESCRIPTION_LENGTH characters")
        }
    }
}
```

Здесь важна архитектурная идея:
- `repository` хранит данные;
- `service` содержит правила и валидацию;
- `routing` работает только как HTTP-слой.

### Маршруты для API задач

```kotlin
import io.ktor.http.HttpStatusCode
import io.ktor.server.application.call
import io.ktor.server.request.receive
import io.ktor.server.response.respond
import io.ktor.server.routing.delete
import io.ktor.server.routing.get
import io.ktor.server.routing.patch
import io.ktor.server.routing.post
import io.ktor.server.routing.route
import io.ktor.server.routing.routing

fun Application.configureTaskRouting(taskService: TaskService) {
    routing {
        route("/tasks") {
            get {
                call.respond(taskService.getAll())
            }

            get("/{id}") {
                val id = call.parameters["id"]?.toLongOrNull()
                    ?: throw ValidationException("Task id must be a number")

                call.respond(taskService.getById(id))
            }

            post {
                val request = call.receive<CreateTaskRequest>()
                val created = taskService.create(request)
                call.respond(HttpStatusCode.Created, created)
            }

            patch("/{id}") {
                val id = call.parameters["id"]?.toLongOrNull()
                    ?: throw ValidationException("Task id must be a number")

                val request = call.receive<UpdateTaskRequest>()
                call.respond(taskService.update(id, request))
            }

            delete("/{id}") {
                val id = call.parameters["id"]?.toLongOrNull()
                    ?: throw ValidationException("Task id must be a number")

                taskService.delete(id)
                call.respond(HttpStatusCode.NoContent)
            }
        }
    }
}
```

### Полная сборка приложения

```kotlin
import io.ktor.serialization.kotlinx.json.json
import io.ktor.server.application.Application
import io.ktor.server.application.install
import io.ktor.server.plugins.contentnegotiation.ContentNegotiation
import io.ktor.server.plugins.statuspages.StatusPages

fun Application.module() {
    val repository = TaskRepository()
    val taskService = TaskService(repository)

    install(ContentNegotiation) {
        json()
    }

    install(StatusPages) {
        exception<ValidationException> { call, cause ->
            call.respond(io.ktor.http.HttpStatusCode.BadRequest, ErrorResponse(cause.message ?: "Validation failed"))
        }

        exception<TaskNotFoundException> { call, cause ->
            call.respond(io.ktor.http.HttpStatusCode.NotFound, ErrorResponse(cause.message ?: "Task not found"))
        }
    }

    configureTaskRouting(taskService)
}
```

### Примеры запросов и ответов

Создание задачи:

```http
POST /tasks
Content-Type: application/json

{
  "title": "Write practice",
  "description": "Prepare Ktor seminar"
}
```

Ответ:

```json
{
  "id": 1,
  "title": "Write practice",
  "description": "Prepare Ktor seminar",
  "completed": false
}
```

Частичное обновление:

```http
PATCH /tasks/1
Content-Type: application/json

{
  "completed": true
}
```

Ответ:

```json
{
  "id": 1,
  "title": "Write practice",
  "description": "Prepare Ktor seminar",
  "completed": true
}
```

### Что здесь похоже на реальный backend

Этот сервис упрощённый, но уже содержит важные backend-идеи:
- RESTful маршруты;
- корректные HTTP-коды;
- DTO и доменные модели;
- выделенный service-слой;
- централизованную обработку ошибок;
- подготовку к будущему подключению БД.

### Упражнения: RESTful task manager

1. Реализуйте `GET /tasks`, `GET /tasks/{id}`, `POST /tasks`, `PATCH /tasks/{id}`, `DELETE /tasks/{id}` для in-memory хранилища.
2. Добавьте query-параметр `completed=true/false` в `GET /tasks`, чтобы можно было фильтровать задачи по статусу.
3. Измените модель так, чтобы у задачи появилось поле `priority` со значениями `LOW`, `MEDIUM`, `HIGH`. Какие DTO и проверки нужно обновить?
4. Что должен делать `PATCH /tasks/{id}`, если клиент прислал пустой JSON `{}`? Объясните и реализуйте выбранное поведение.
5. Добавьте endpoint `POST /tasks/{id}/complete`, который помечает задачу выполненной. Обсудите: это хороший REST-дизайн или лучше обойтись `PATCH`?

---

## Тестирование API

### Зачем тестировать HTTP API

Даже если сервис маленький, тесты помогают проверить:
- правильные статус-коды;
- сериализацию JSON;
- обработку ошибок;
- edge cases вроде невалидного `id`.

### Идея теста в Ktor

В Ktor можно поднимать тестовое приложение и отправлять запросы прямо в тесте.

```kotlin
import io.ktor.client.request.delete
import io.ktor.client.request.get
import io.ktor.client.request.patch
import io.ktor.client.request.post
import io.ktor.client.request.setBody
import io.ktor.client.statement.bodyAsText
import io.ktor.http.ContentType
import io.ktor.http.HttpHeaders
import io.ktor.http.HttpStatusCode
import io.ktor.http.contentType
import io.ktor.server.testing.testApplication
import kotlin.test.Test
import kotlin.test.assertEquals

class TaskApiTest {
    @Test
    fun `create task returns 201`() = testApplication {
        application {
            module()
        }

        val response = client.post("/tasks") {
            contentType(ContentType.Application.Json)
            setBody(
                """
                {
                  "title": "Learn Ktor",
                  "description": "Write first API"
                }
                """.trimIndent()
            )
        }

        assertEquals(HttpStatusCode.Created, response.status)
    }

    @Test
    fun `get unknown task returns 404`() = testApplication {
        application {
            module()
        }

        val response = client.get("/tasks/999")
        assertEquals(HttpStatusCode.NotFound, response.status)
    }
}
```

### Что особенно полезно проверить

| Тест | Что проверяет |
|---|---|
| `POST /tasks` | создание и `201 Created` |
| `GET /tasks/{id}` | успешное чтение существующей задачи |
| `GET /tasks/abc` | валидацию `id` и `400 Bad Request` |
| `DELETE /tasks/{id}` | `204 No Content` |
| невалидный JSON | корректную обработку ошибки десериализации |

### Типичные edge cases

- пустой `title`;
- слишком длинное `description`;
- `id`, который не является числом;
- удаление уже удалённой задачи;
- частичное обновление только одного поля.

### Практическая польза тестов

В реальном проекте API-тесты защищают от регрессий: кто-то поменял DTO или статус-код — тест это сразу покажет.

### Упражнения: тестирование API

1. Напишите тест на `POST /tasks`, который проверяет, что сервер возвращает `201 Created`.
2. Напишите тест на `GET /tasks/unknown`, где `unknown` — не число. Ожидаемый результат: `400 Bad Request`.
3. Добавьте тест на сценарий: создать задачу, затем получить её по `id`, затем удалить.
4. Объясните, почему для CRUD API важно проверять не только тело ответа, но и HTTP-статус.

---

## Вопросы

1. Почему в Ktor полезно выносить бизнес-логику из route-блоков в service-слой?
2. Когда использовать path parameters, а когда query parameters?
3. Зачем нужен `ContentNegotiation`, если можно вручную формировать JSON?
4. Почему `StatusPages` лучше множества локальных `try/catch` в каждом endpoint-е?
5. Чем `PATCH` отличается от `PUT`, и какой из них лучше подходит для частичного обновления задачи?
6. Какие ограничения у in-memory хранилища и почему в production обычно нужна база данных?

---

## Итоги

1. Ktor Server позволяет писать HTTP API в Kotlin с явной и компактной конфигурацией.
2. `routing` и `ApplicationCall` образуют HTTP-слой, который должен быть тонким и предсказуемым.
3. `ContentNegotiation` и `kotlinx.serialization` делают работу с JSON удобной и типобезопасной.
4. `StatusPages` помогает централизованно обрабатывать ошибки и возвращать стабильный формат ответов.
5. Даже простой RESTful сервис стоит строить слоями: routing, service, repository, DTO.
6. API-тесты нужны, чтобы проверять статус-коды, сериализацию, валидацию и защиту от регрессий.
