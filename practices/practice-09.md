# practice-09: Flow, Channel, Actor

## План

1. `Flow` — интерфейс, cold vs hot, `StateFlow`, `value`, CAS, builders и операторы
2. `Channel` — интерфейс, зачем нужен, capacity, producers/consumers, `channelFlow`, backend-примеры
3. `Actor` — модель акторов, зачем нужна, actor builder в Kotlin, backend-примеры
4. Вопросы
5. Итоги

---

## Flow

### Что такое `Flow`

`Flow<T>` — это асинхронный поток значений. Он похож на `Sequence`, но работает с корутинами и может выдавать элементы не сразу, а со временем.

```kotlin
import kotlinx.coroutines.delay
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.flow

fun userEvents(): Flow<String> = flow {
    emit("user-created")
    delay(100)
    emit("user-updated")
    delay(100)
    emit("user-deleted")
}
```

Получение значений происходит через `collect`:

```kotlin
import kotlinx.coroutines.coroutineScope

suspend fun main(): Unit = coroutineScope {
    userEvents().collect { event ->
        println("Received: $event")
    }
}
```

### Зачем нужен `Flow`

`Flow` полезен, когда данные:
- приходят **постепенно**;
- требуют цепочки преобразований;
- должны обрабатываться декларативно;
- могут быть отменены вместе с корутиной.

Backend-кейсы:
- поток событий из Kafka-like источника;
- обработка записей из БД батчами;
- реакция на обновления кэша;
- streaming API с фильтрацией, маппингом и retry-логикой.

### Интерфейс `Flow`

Концептуально `Flow` очень маленький: у него есть метод `collect`, который запускает сбор данных.

```kotlin
public interface Flow<out T> {
    suspend fun collect(collector: FlowCollector<T>)
}
```

На практике мы редко реализуем `Flow` вручную — чаще используем builders и операторы.

### Cold `Flow`

Обычный `Flow` — **cold**: он ничего не делает, пока его не начали собирать.

```kotlin
import kotlinx.coroutines.flow.flow

fun requestFlow() = flow {
    println("Start request")
    emit("data")
}
```

Если вызвать `requestFlow()`, но не вызвать `collect`, код внутри builder-а не выполнится.

```kotlin
import kotlinx.coroutines.coroutineScope

suspend fun main(): Unit = coroutineScope {
    val flow = requestFlow()
    println("Flow created")

    flow.collect { println(it) }
    flow.collect { println("Second collector: $it") }
}
```

Важно: каждый новый `collect` заново запускает upstream-логику.

### Hot `Flow`: `StateFlow`

`StateFlow` — это **hot**-поток состояния. Он существует независимо от наличия collector-ов и всегда хранит **текущее значение**.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.MutableStateFlow

suspend fun main(): Unit = coroutineScope {
    val state = MutableStateFlow(0)

    // StateFlow никогда не завершает collect сам по себе — корутину нужно отменять явно
    val job = launch {
        state.collect { value ->
            println("State = $value")
        }
    }

    state.value = 1
    state.value = 2
    delay(100) // даём корутине шанс обработать оба обновления
    job.cancel()
}
```

Свойства `StateFlow`:
- у него всегда есть текущее значение;
- новый collector сразу получает последнее состояние;
- он хорошо подходит для in-memory состояния приложения.

### `value` и атомарные обновления

У `MutableStateFlow` можно читать и менять текущее значение через `value`.

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow

val counter = MutableStateFlow(0)

counter.value = 1
println(counter.value)
```

Но если несколько корутин обновляют значение одновременно, простое чтение и запись могут привести к гонке.

Проблемный код:

```kotlin
val counter = MutableStateFlow(0)

fun incrementUnsafe() {
    counter.value = counter.value + 1
}
```

Безопаснее использовать атомарные операции, например `update`, а концептуально это похоже на CAS (compare-and-set): обновление должно примениться только если состояние ещё актуально.

```kotlin
import kotlinx.coroutines.flow.MutableStateFlow
import kotlinx.coroutines.flow.update

val counter = MutableStateFlow(0)

fun incrementSafe() {
    counter.update { oldValue -> oldValue + 1 }
}
```

Идея CAS:
- читаем старое значение;
- пытаемся заменить его на новое;
- если кто-то уже изменил состояние, повторяем попытку.

Это особенно важно в backend-сервисах с общим in-memory состоянием: счётчики, кеш статистики, rate limiting, snapshot состояния.

### Cold vs Hot

| Свойство | `Flow` | `StateFlow` |
|---|---|---|
| Тип | Cold | Hot |
| Запускается без collector-а | Нет | Да, существует независимо |
| Хранит последнее значение | Нет | Да |
| Каждый `collect` заново выполняет source | Да | Нет |
| Типичный сценарий | Поток запросов, чтение из БД, streaming pipeline | Текущее состояние сервиса, конфигурация, кэш состояния |

### Builders для `Flow`

Часто используются:

| Builder | Когда использовать |
|---|---|
| `flow {}` | Обычный последовательный поток |
| `flowOf(...)` | Небольшой фиксированный набор значений |
| `asFlow()` | Превратить коллекцию/диапазон в flow |
| `channelFlow {}` | Когда нужно отправлять данные из нескольких корутин |

Примеры:

```kotlin
import kotlinx.coroutines.flow.asFlow
import kotlinx.coroutines.flow.flow
import kotlinx.coroutines.flow.flowOf

val a = flowOf(1, 2, 3)
val b = listOf("a", "b", "c").asFlow()
val c = flow {
    emit(10)
    emit(20)
}
```

### Операторы `Flow`

Сила `Flow` — в композиции операторов.

```kotlin
import kotlinx.coroutines.flow.*

fun activeUserNames(ids: Flow<Long>): Flow<String> =
    ids
        .map { id -> loadUser(id) }
        .filter { user -> user.isActive }
        .map { user -> user.name }

suspend fun loadUser(id: Long): User = User(id, "user-$id", id % 2L == 0L)

data class User(val id: Long, val name: String, val isActive: Boolean)
```

Backend-пример: пришёл поток id пользователей, мы обогатили его данными из БД, отфильтровали неактивных и отдали дальше.

### Пример из реального backend-кода

Ниже — поток событий аудита, который фильтрует только ошибки и отправляет их в обработчик уведомлений:

```kotlin
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.filter
import kotlinx.coroutines.flow.map

fun errorAlerts(events: Flow<AuditEvent>): Flow<String> =
    events
        .filter { event -> event.level == LogLevel.ERROR }
        .map { event -> "ALERT: ${event.message}" }

data class AuditEvent(
    val message: String,
    val level: LogLevel,
)

enum class LogLevel {
    INFO,
    WARN,
    ERROR,
}
```

Это удобно тем, что pipeline читается сверху вниз и легко расширяется дополнительными шагами: `retry`, `debounce`, `buffer`, `catch`.

### Сравнение с Java

| Kotlin `Flow` | Java |
|---|---|
| Асинхронный stream поверх корутин | Часто используют `Stream`, `CompletableFuture`, Reactive Streams-библиотеки |
| Ленивая и декларативная обработка последовательности | `Stream` удобен, но не предназначен для suspend-операций |
| Хорошо работает с отменой корутин | В Java отмена обычно сложнее и менее единообразна |

`Stream` в Java хорошо подходит для синхронных преобразований коллекций, а `Flow` — для асинхронных последовательностей во времени.

### Упражнения: Flow

1. Напишите `flow`, который раз в 200 мс эмитит числа от 1 до 5, а затем завершается. Соберите его через `collect`.

2. Реализуйте pipeline:
   - на вход приходит `Flow<Int>`;
   - оставьте только чётные числа;
   - умножьте их на 10;
   - выведите результат.

3. Объясните письменно:
   - почему обычный `Flow` называют cold;
   - почему `StateFlow` называют hot;
   - в каком случае для backend лучше взять `StateFlow`, а не просто `flow {}`.

4. Есть `MutableStateFlow<Int>` со значением счётчика запросов. Покажите:
   - небезопасное обновление через `value = value + 1`;
   - безопасное обновление через `update`.
   Объясните, где здесь возникает проблема гонки.

5. Напишите функцию, которая принимает `Flow<String>` с логами и возвращает новый `Flow<String>`, содержащий только сообщения уровня `ERROR`.

---

## Channel

### Что такое `Channel`

`Channel` — это примитив для **обмена данными между корутинами**. Его можно представить как асинхронную очередь: одна корутина отправляет элементы, другая получает их.

Типичный сценарий в backend: одна часть системы производит события (`producer`), а другая последовательно их обрабатывает (`consumer`).

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel

suspend fun main(): Unit = coroutineScope {
    val channel = Channel<String>()

    launch {
        channel.send("event-1")
        channel.send("event-2")
        channel.close()
    }

    launch {
        for (event in channel) {
            println("Handle: $event")
        }
    }
}
```

Здесь:
- `send` отправляет элемент в канал;
- `receive` или цикл `for (x in channel)` получает элементы;
- `close()` закрывает канал, сообщая потребителю, что новых элементов больше не будет.

### Зачем использовать `Channel`

`Channel` полезен, когда нужно:
- **развязать producer и consumer** по времени;
- организовать **очередь задач**;
- передавать данные между корутинами без общего изменяемого состояния;
- ограничить поток данных и контролировать **backpressure**.

Backend-кейсы:
- очередь фоновых задач на отправку email;
- буферизация входящих событий перед записью в БД;
- сериализация запросов к медленному внешнему API;
- обработка логов или метрик отдельной корутиной.

### Интерфейс `Channel`

Упрощённо у канала есть две стороны:

| Роль | Интерфейс | Основные операции |
|---|---|---|
| Отправитель | `SendChannel<T>` | `send`, `trySend`, `close` |
| Получатель | `ReceiveChannel<T>` | `receive`, `tryReceive`, `cancel` |
| Полный канал | `Channel<T>` | объединяет обе стороны |

Это полезно для проектирования API: можно отдавать наружу только ту часть, которая действительно нужна.

```kotlin
import kotlinx.coroutines.channels.ReceiveChannel
import kotlinx.coroutines.channels.SendChannel

class EventProducer(private val out: SendChannel<String>) {
    suspend fun publish(event: String) {
        out.send(event)
    }
}

class EventConsumer(private val input: ReceiveChannel<String>) {
    suspend fun consumeAll() {
        for (event in input) {
            println("Consumed: $event")
        }
    }
}
```

### Capacity и поведение канала

Важная настройка канала — **вместимость** (`capacity`). Она определяет, сколько элементов можно буферизовать.

| Вариант | Поведение | Когда использовать |
|---|---|---|
| `Channel()` / `Channel.RENDEZVOUS` | Без буфера: `send` ждёт `receive` | Когда нужна строгая синхронизация |
| `Channel(n)` | Буфер на `n` элементов | Когда producer временно быстрее consumer |
| `Channel.UNLIMITED` | Практически неограниченный буфер | Осторожно: риск роста памяти |
| `Channel.CONFLATED` | Хранит только последнее значение | Для сигналов и обновлений состояния |

Пример буферизованного канала:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel

suspend fun main(): Unit = coroutineScope {
    val channel = Channel<Int>(capacity = 2)

    launch {
        repeat(5) {
            println("Sending $it")
            channel.send(it)
            println("Sent $it")
        }
        channel.close()
    }

    launch {
        for (x in channel) {
            delay(300)
            println("Received $x")
        }
    }
}
```

Первые элементы попадут в буфер быстро, а затем `send` начнёт приостанавливаться, пока consumer не освободит место.

### `send`/`receive` vs `trySend`/`tryReceive`

- `send` и `receive` — **suspend**-операции, они могут ждать;
- `trySend` и `tryReceive` — **не suspend**, они возвращают результат сразу.

```kotlin
import kotlinx.coroutines.channels.Channel

fun main() {
    val channel = Channel<String>(capacity = 1)

    println(channel.trySend("first").isSuccess)
    println(channel.trySend("second").isSuccess) // всегда false: буфер вместимостью 1 уже заполнен
}
```

Такой API удобен, если нельзя приостанавливать текущий код, например внутри callback или при интеграции со старым Java API.

### `channelFlow`

`channelFlow` — это builder, который позволяет строить `Flow`, отправляя элементы через `send` из нескольких корутин.

Он полезен, когда у нас есть несколько источников данных, а наружу хочется отдать **один поток**.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.flow.Flow
import kotlinx.coroutines.flow.channelFlow

fun mergedEvents(): Flow<String> = channelFlow {
    launch {
        repeat(3) {
            delay(100)
            send("db-event-$it")
        }
    }

    launch {
        repeat(3) {
            delay(150)
            send("cache-event-$it")
        }
    }
}
```

В backend это похоже на агрегацию событий из БД, кэша и внешнего API в единый stream.

### Пример из реального backend-кода

Представим сервис, который принимает запросы на генерацию отчётов, но хочет обрабатывать их по одному, чтобы не перегружать внешний сервис.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel

private const val REPORT_QUEUE_CAPACITY = 100

class ReportService(
    private val externalClient: ExternalReportClient,
) : AutoCloseable {
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.IO)
    private val reportRequests = Channel<ReportRequest>(capacity = REPORT_QUEUE_CAPACITY)

    init {
        scope.launch {
            for (request in reportRequests) {
                try {
                    externalClient.generate(request)
                } catch (e: Exception) {
                    println("Failed to generate report ${request.id}: ${e.message}")
                }
            }
        }
    }

    suspend fun submit(request: ReportRequest) {
        reportRequests.send(request)
    }

    override fun close() {
        reportRequests.close()
        // Не отменяем scope принудительно: корутина-обработчик сама завершится,
        // когда for-loop дочитает оставшиеся элементы из закрытого канала.
        // scope.cancel() здесь привёл бы к потере запросов, уже лежащих в буфере.
    }
}

data class ReportRequest(val id: Long)

interface ExternalReportClient {
    suspend fun generate(request: ReportRequest)
}
```

Почему это полезно:
- входящие запросы складываются в очередь;
- обработка идёт последовательно и предсказуемо;
- нет гонок на общем состоянии;
- можно контролировать размер очереди.

### Сравнение с Java

| Kotlin | Java |
|---|---|
| `Channel` интегрирован с корутинами | Часто используют `BlockingQueue`, `ExecutorService`, `CompletableFuture` |
| `send`/`receive` не блокируют поток, а приостанавливают корутину | `put`/`take` в `BlockingQueue` обычно блокируют поток |
| Удобно комбинируется с `Flow` | Обычно приходится вручную связывать очередь и async-обработку |

Упрощённое сравнение:

```kotlin
// Kotlin
val channel = Channel<String>()
launch {
    channel.send("task")
}
```

В Java аналог обычно строят через `BlockingQueue<String>` и вызов `put("task")`, который, в отличие от корутинного `send`, как правило блокирует поток.

### Упражнения: Channel

1. Напишите программу с двумя корутинами:
   - первая отправляет в канал числа от 1 до 5;
   - вторая читает их и печатает квадраты.
   Используйте `close()` и цикл `for (x in channel)`.

2. Измените пример так, чтобы канал имел `capacity = 2`. Понаблюдайте по логам, в какой момент `send` начинает ждать consumer.

3. Объясните письменно:
   - чем отличается `Channel.CONFLATED` от обычного буферизованного канала;
   - в каких backend-сценариях потеря промежуточных значений допустима, а в каких — нет.

4. Реализуйте очередь задач `TaskQueue`, которая:
   - принимает строки с именами задач;
   - обрабатывает их в одной корутине;
   - логирует ошибки;
   - корректно закрывается через `close()`.

5. Вопрос на понимание: почему `Channel.UNLIMITED` опасен в сервисе с высоким входящим трафиком?

---

## Actor

### Что такое actor-модель

Actor — это сущность, которая:
- имеет своё внутреннее состояние;
- получает сообщения;
- обрабатывает их **последовательно**;
- может отправлять новые сообщения другим actor-ам.

Главная идея: **не делить состояние между потоками**, а передавать сообщения. Это уменьшает число гонок и упрощает reasoning о коде.

### Зачем нужен actor

Actor полезен, когда нужно:
- сериализовать доступ к общему состоянию;
- избежать явных `Mutex` и ручной синхронизации;
- построить простую message-driven обработку;
- инкапсулировать состояние в одной корутине.

Backend-кейсы:
- счётчик метрик;
- менеджер сессий;
- последовательная запись в кэш или файл;
- ограничение параллелизма при работе с внешним API.

### Actor в Kotlin

В Kotlin actor обычно строится поверх канала: у нас есть корутина, которая читает сообщения из почтового ящика (`Channel`) и меняет своё состояние.

Идея выглядит так:

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.Channel

sealed interface CounterMessage

data object Increment : CounterMessage
class GetValue(val replyTo: CompletableDeferred<Int>) : CounterMessage

// Возвращаем SendChannel — снаружи нет смысла читать из канала actor-а
fun CoroutineScope.counterActor(): SendChannel<CounterMessage> {
    val mailbox = Channel<CounterMessage>()

    launch {
        var counter = 0
        for (message in mailbox) {
            when (message) {
                Increment -> counter++
                is GetValue -> message.replyTo.complete(counter)
            }
        }
    }

    return mailbox
}
```

Здесь состояние `counter` живёт только внутри одной корутины, поэтому доступ к нему автоматически сериализован.

### Builder-функция actor

В `kotlinx.coroutines` есть builder `actor`, который создаёт actor в более компактной форме.

> **Предупреждение:** `actor { }` помечен как `@ObsoleteCoroutinesApi` и может быть удалён в будущих версиях библиотеки. При компиляции вы увидите предупреждение. Идиоматичный вариант — использовать `launch { for (msg in mailbox) { ... } }`, как в примере выше.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.actor

sealed interface CacheMessage

data class Put(val key: String, val value: String) : CacheMessage
class Get(val key: String, val replyTo: CompletableDeferred<String?>) : CacheMessage

@OptIn(ObsoleteCoroutinesApi::class)
fun CoroutineScope.cacheActor() = actor<CacheMessage> {
    val cache = mutableMapOf<String, String>()

    for (message in channel) {
        when (message) {
            is Put -> cache[message.key] = message.value
            is Get -> message.replyTo.complete(cache[message.key])
        }
    }
}
```

Такой подход удобен, когда логика действительно message-driven и мы хотим скрыть внутреннюю структуру данных.

### Пример backend-сценария

Представим ограничитель запросов к внешнему API. Мы хотим, чтобы все обновления счётчика шли через один actor.

```kotlin
import kotlinx.coroutines.*
import kotlinx.coroutines.channels.actor

private const val MAX_REQUESTS_PER_MINUTE = 100

sealed interface RateLimiterMessage
class TryAcquire(val replyTo: CompletableDeferred<Boolean>) : RateLimiterMessage
object Reset : RateLimiterMessage

@OptIn(ObsoleteCoroutinesApi::class)
fun CoroutineScope.rateLimiterActor() = actor<RateLimiterMessage> {
    var usedRequests = 0

    for (message in channel) {
        when (message) {
            is TryAcquire -> {
                val allowed = usedRequests < MAX_REQUESTS_PER_MINUTE
                if (allowed) {
                    usedRequests++
                }
                message.replyTo.complete(allowed)
            }
            Reset -> usedRequests = 0
        }
    }
}
```

Преимущества:
- состояние `usedRequests` обновляется в одном месте;
- нет гонки на счётчике;
- логику можно расширить: таймауты, статистика, приоритеты.

### Actor vs `Mutex`

| Подход | Идея | Плюсы | Минусы |
|---|---|---|---|
| `Mutex` | Несколько корутин делят состояние, но берут lock | Просто добавить к существующему коду | Нужно следить за критическими секциями |
| `Actor` | Состояние живёт в одной корутине, остальные шлют сообщения | Меньше shared mutable state, проще reasoning | Не всегда удобно для простых случаев |

Если нужно быстро защитить небольшой участок кода — `Mutex` может быть проще. Если есть самостоятельный stateful-компонент с очередью команд — actor часто выразительнее.

### Где actor особенно полезен в backend

1. **Сериализация команд** — например, обновление одного in-memory индекса.
2. **Single writer** — одна корутина пишет в файл или сокет.
3. **Stateful service** — кэш, quota manager, session registry.
4. **Изоляция ошибок** — actor можно явно ограничить и контролировать через scope.

### Упражнения: Actor

1. Реализуйте actor-счётчик, который поддерживает сообщения:
   - `Increment`
   - `Decrement`
   - `GetValue`
   Проверьте, что при множестве параллельных отправителей итоговое значение корректно.

2. Напишите actor для простого in-memory кэша `String -> String` с командами `Put`, `Get`, `Remove`.

3. Объясните письменно:
   - чем actor отличается от обычного `Channel`;
   - когда достаточно просто канала, а когда лучше выделить отдельный actor с состоянием.

4. Перед вами задача: ограничить обращения к внешнему API до 10 запросов одновременно. Что выберете и почему:
   - `Semaphore`;
   - `Channel`;
   - actor?
   Дайте краткое архитектурное объяснение.

5. Подумайте, какие проблемы возникнут, если actor обрабатывает сообщения слишком медленно, а входящий поток сообщений высокий.

---

## Вопросы

1. В чём принципиальная разница между `Channel` и `Flow`, если оба используются для передачи данных между корутинами?
2. Почему обычный `Flow` удобен для декларативных pipeline, а `Channel` — для очередей и координации producer/consumer?
3. В каких backend-сценариях стоит использовать `StateFlow`, а в каких это будет плохим выбором?
4. Почему actor помогает бороться с shared mutable state, и чем он отличается от подхода с `Mutex`?
5. Какие риски несут `Channel.UNLIMITED` и бесконтрольный hot-stream в серверном приложении?
6. Если в сервисе есть входящий поток событий, их буферизация и обновление общего состояния, как бы вы разделили роли между `Flow`, `Channel` и actor?

---

## Итоги

1. `Flow` описывает асинхронную последовательность значений и хорошо подходит для декларативных pipeline обработки данных.
2. Обычный `Flow` — cold, а `StateFlow` — hot и хранит текущее состояние, что важно для stateful backend-компонентов.
3. Для обновления `MutableStateFlow` в конкурентной среде лучше использовать атомарные операции вроде `update`. `StateFlow.collect` никогда не завершается сам по себе — корутину с collector-ом нужно отменять явно.
4. `Channel` — это асинхронная очередь между корутинами, особенно полезная для producer/consumer и контроля backpressure.
5. Вместимость канала (`capacity`) напрямую влияет на поведение системы под нагрузкой и должна выбираться осознанно. `Channel.UNLIMITED` опасен при высоком входящем трафике.
6. Actor — это способ инкапсулировать состояние и обрабатывать сообщения последовательно, уменьшая число гонок и упрощая архитектуру. Предпочтительная реализация — `launch { for (msg in mailbox) { ... } }`, а не устаревший builder `actor {}`.
7. В реальных backend-проектах `Channel`, `Flow` и actor часто используются вместе: очередь событий, поток преобразований и безопасный stateful-компонент.
