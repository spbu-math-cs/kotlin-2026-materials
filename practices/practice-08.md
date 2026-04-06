# practice-08: корутины — suspend, контекст, scope, Job, билдеры

## План

1. CPS и suspend-функции — напоминание: CPS-трансформация, точки приостановки, приостановка vs блокировка
2. CoroutineContext — элементы контекста, диспатчеры, сложение контекстов
3. CoroutineScope — structured concurrency, связь scope и контекста, GlobalScope vs coroutineScope
4. Job — жизненный цикл, отмена, иерархия parent-child, SupervisorJob
5. Билдеры корутин — launch, async, coroutineScope, runBlocking
6. Вопросы
7. Итоги

---

## CPS и suspend-функции

### Напоминание: что такое suspend

Ключевое слово `suspend` — маркер для компилятора: эта функция **может приостановить** выполнение корутины, не блокируя поток.

```kotlin
suspend fun fetchUser(id: Int): User {
    val response = httpClient.get("/users/$id")  // точка приостановки
    val avatar = downloadAvatar(response.avatarUrl)  // ещё одна
    return User(response.name, avatar)
}
```

Вызывать `suspend`-функцию можно только:
- из другой `suspend`-функции;
- из корутины (внутри билдера — `launch`, `async` и т.д.).

### CPS-трансформация

Компилятор добавляет скрытый параметр `Continuation` к каждой `suspend`-функции:

```kotlin
// Что мы пишем:
suspend fun fetchUser(id: Int): User

// Что видит JVM после компиляции:
fun fetchUser(id: Int, continuation: Continuation<User>): Any?
```

- `Continuation` — объект, хранящий **где мы остановились** (label), **локальные переменные** и **контекст корутины**.
- Возвращаемый `Any?` — либо результат, либо маркер `COROUTINE_SUSPENDED`.

### Конечный автомат

Компилятор разбивает `suspend`-функцию по точкам приостановки на состояния:

```kotlin
suspend fun loadData(): String {
    val user = fetchUser()         // точка приостановки ①
    val orders = fetchOrders(user) // точка приостановки ②
    return "$user: $orders"
}
```

Превращается примерно в:

```kotlin
fun loadData(cont: Continuation<String>): Any? {
    when (cont.label) {
        0 -> {
            cont.label = 1
            val result = fetchUser(cont)
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        1 -> {
            val user = cont.result as User
            cont.label = 2
            cont.user = user
            val result = fetchOrders(user, cont)
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        2 -> {
            val orders = cont.result as List<Order>
            return "${cont.user}: $orders"
        }
    }
}
```

### Приостановка vs блокировка

| | Блокировка (`Thread.sleep`, `future.get()`) | Приостановка (`delay`, `suspend`-вызов) |
|---|---|---|
| Поток | Занят, но простаивает | Освобождается для другой работы |
| Ресурсы | Тратятся впустую | Корутина ждёт в памяти (~сотни байт) |
| Масштабируемость | Ограничена числом потоков | Тысячи корутин на нескольких потоках |

### suspendCoroutine — мост к callback-миру

Любой callback-based API можно обернуть в `suspend`-функцию:

```kotlin
suspend fun fetchUser(id: Int): User =
    suspendCoroutine { continuation ->
        // вызываем старый callback-API
        fetchUserAsync(id) { user ->
            continuation.resume(user) // возобновляем корутину с результатом
        }
    }
```

Для отменяемых корутин используйте `suspendCancellableCoroutine`.

### Упражнения: CPS и suspend

1. Определите все точки приостановки в следующем коде. Сколько состояний будет в конечном автомате?

    ```kotlin
    suspend fun process() {
        val a = fetchA()
        val b = if (a > 0) fetchB(a) else fetchC()
        println("Result: ${transform(b)}")
    }
    // fetchA, fetchB, fetchC, transform — все suspend
    ```

2. Объясните письменно: почему `delay(1000)` не блокирует поток, а `Thread.sleep(1000)` — блокирует? Что происходит с потоком в каждом случае?

3. Напишите `suspend`-обёртку для следующего callback-API:

    ```kotlin
    fun loadDataAsync(callback: (Result<String>) -> Unit) {
        thread {
            Thread.sleep(500)
            callback(Result.success("data"))
        }
    }
    ```

    Используйте `suspendCancellableCoroutine` и обработайте как успех, так и ошибку.

---

## CoroutineContext

### Что такое CoroutineContext

`CoroutineContext` — это **неизменяемая ассоциативная структура** «ключ → элемент», привязанная к каждой корутине. Она определяет **как** и **где** корутина выполняется.

Основные элементы:

| Ключ | Элемент | Назначение |
|---|---|---|
| `Job` | `Job` / `SupervisorJob` | Управление жизненным циклом, отмена |
| `ContinuationInterceptor` | `CoroutineDispatcher` | В каком потоке выполнять |
| `CoroutineName` | `CoroutineName` | Имя для отладки |
| `CoroutineExceptionHandler` | `CoroutineExceptionHandler` | Обработка необработанных исключений |

### Доступ к контексту

Внутри корутины контекст доступен через `coroutineContext`:

```kotlin
import kotlinx.coroutines.*

suspend fun showContext() {
    val ctx = coroutineContext
    println("Job: ${ctx[Job]}")
    println("Dispatcher: ${ctx[ContinuationInterceptor]}")
    println("Name: ${ctx[CoroutineName]}")
}
```

### Сложение контекстов

Контексты комбинируются оператором `+`. Правый элемент **перезаписывает** левый по тому же ключу:

```kotlin
val ctx = Dispatchers.IO + CoroutineName("loader") + CoroutineExceptionHandler { _, e ->
    println("Caught: $e")
}
// ctx содержит: Dispatcher=IO, Name="loader", ExceptionHandler=...
```

Порядок важен при конфликте ключей:

```kotlin
val ctx1 = CoroutineName("first") + CoroutineName("second")
println(ctx1[CoroutineName]) // CoroutineName(second) — правый побеждает
```

### Стандартные диспатчеры

| Диспатчер | Пул потоков | Когда использовать |
|---|---|---|
| `Dispatchers.Default` | Общий пул, размер = кол-во CPU ядер | CPU-интенсивные задачи (парсинг, сортировка) |
| `Dispatchers.IO` | Отдельный пул, до 64 потоков | Блокирующий I/O (файлы, БД, сеть) |
| `Dispatchers.Main` | Главный поток (Android/UI) | Обновление UI |
| `Dispatchers.Unconfined` | Без привязки к потоку | Тесты, специальные случаи |

> `Dispatchers.Default` и `Dispatchers.IO` **разделяют** потоки (shared pool), но IO может создавать дополнительные потоки для блокирующих операций.

### withContext — смена диспатчера

`withContext` переключает контекст (обычно — диспатчер) для блока кода:

```kotlin
suspend fun loadFromDb(): List<User> = withContext(Dispatchers.IO) {
    // этот блок выполняется на IO-диспатчере
    database.queryAll()
}

suspend fun processUsers() {
    val users = loadFromDb()          // IO
    val sorted = withContext(Dispatchers.Default) {
        users.sortedBy { it.name }    // CPU
    }
    println(sorted)
}
```

### Упражнения: CoroutineContext

1. Запустите следующий код и объясните вывод — в каком потоке выполняется каждый `println`:

    ```kotlin
    fun main() = runBlocking(CoroutineName("main")) {
        println("1: ${Thread.currentThread().name} — ${coroutineContext[CoroutineName]}")

        launch(Dispatchers.Default) {
            println("2: ${Thread.currentThread().name} — ${coroutineContext[CoroutineName]}")
        }

        launch(Dispatchers.IO + CoroutineName("io-task")) {
            println("3: ${Thread.currentThread().name} — ${coroutineContext[CoroutineName]}")
        }

        launch(Dispatchers.Unconfined) {
            println("4: ${Thread.currentThread().name}")
            delay(10)
            println("5: ${Thread.currentThread().name}") // какой поток теперь?
        }
    }
    ```

2. Напишите функцию, которая принимает `CoroutineContext` и печатает все его известные элементы (`Job`, `CoroutineName`, `ContinuationInterceptor`). Протестируйте на разных контекстах.

3. Объясните письменно: почему `Dispatchers.Unconfined` не рекомендуется для production-кода? Что произойдёт с потоком после `delay` внутри `Unconfined`-корутины?

---

## CoroutineScope

### Зачем нужен CoroutineScope

`CoroutineScope` — это **граница жизни** корутин. Он гарантирует **structured concurrency**: все дочерние корутины завершатся (или будут отменены) до того, как завершится scope.

```kotlin
interface CoroutineScope {
    val coroutineContext: CoroutineContext
}
```

Scope — это по сути обёртка над контекстом. Билдеры `launch` и `async` — это extension-функции на `CoroutineScope`.

### Structured concurrency

Ключевые правила:
1. **Дочерняя корутина наследует контекст** родительского scope (+ может переопределить элементы).
2. **Родитель ждёт** завершения всех дочерних корутин.
3. **Отмена родителя** отменяет всех детей.
4. **Ошибка в ребёнке** отменяет родителя (и всех остальных детей).

```kotlin
suspend fun loadDashboard() = coroutineScope {
    val user = async { fetchUser() }       // дочерняя корутина 1
    val orders = async { fetchOrders() }   // дочерняя корутина 2

    // coroutineScope дождётся обеих корутин
    Dashboard(user.await(), orders.await())
}
// Если fetchUser() упадёт — fetchOrders() будет отменена автоматически
```

### Способы создания scope

| Способ | Описание | Когда использовать |
|---|---|---|
| `coroutineScope { }` | Suspend-функция, создаёт scope и ждёт всех детей | Декомпозиция suspend-функций |
| `CoroutineScope(context)` | Фабрика, создаёт scope с заданным контекстом | Привязка к жизненному циклу компонента |
| `runBlocking { }` | Блокирует текущий поток до завершения | `main()`, тесты |
| `GlobalScope` | Scope без привязки к жизненному циклу | **Почти никогда** (утечки!) |

### coroutineScope vs GlobalScope

```kotlin
// ✅ Structured: если outer отменён — inner тоже отменится
suspend fun structured() = coroutineScope {
    launch { delay(1000); println("inner") }
}

// ❌ Unstructured: inner живёт сам по себе, утечка!
suspend fun unstructured() {
    GlobalScope.launch { delay(1000); println("inner") }
}
```

`GlobalScope` не привязан ни к какому жизненному циклу — корутины в нём живут до завершения приложения или до собственной отмены. Это частый источник утечек.

### Создание своего scope

Типичный паттерн для компонента с жизненным циклом (ViewModel, сервис):

```kotlin
class UserService : AutoCloseable {
    // Свой scope с SupervisorJob — ошибка одного ребёнка не убивает остальных
    private val scope = CoroutineScope(SupervisorJob() + Dispatchers.Default + CoroutineName("UserService"))

    fun refreshUsers() {
        scope.launch {
            val users = fetchUsers()
            cache.update(users)
        }
    }

    override fun close() {
        scope.cancel() // отменяем все корутины при закрытии
    }
}
```

### Упражнения: CoroutineScope

1. Запустите код и объясните порядок вывода:

    ```kotlin
    fun main() = runBlocking {
        println("1: start")

        coroutineScope {
            launch {
                delay(200)
                println("2: child 1")
            }
            launch {
                delay(100)
                println("3: child 2")
            }
            println("4: coroutineScope body")
        }

        println("5: after coroutineScope")
    }
    ```

2. Замените `coroutineScope` на `GlobalScope.launch` в примере выше. Что изменится? Почему `"5: after coroutineScope"` может напечататься раньше, чем дочерние корутины?

3. Напишите класс `TaskRunner`, который:
    - создаёт свой `CoroutineScope` с `SupervisorJob` и `Dispatchers.Default`;
    - имеет метод `submit(block: suspend () -> Unit)`, запускающий задачу в scope;
    - имеет метод `shutdown()`, отменяющий все задачи;
    - имеет метод `awaitAll()`, ожидающий завершения всех запущенных задач.

---

## Job

### Что такое Job

`Job` — это **дескриптор** корутины, через который можно управлять её жизненным циклом: ожидать завершения, отменять, проверять состояние.

```kotlin
val job: Job = launch {
    delay(1000)
    println("Done")
}

job.join()   // ожидаем завершения
job.cancel() // отменяем
```

### Жизненный цикл Job

```
                          cancel()
        ┌──────────────────────────────────────┐
        │                                      ▼
    ┌───────┐  start()  ┌────────┐         ┌───────────┐   finish   ┌───────────┐
    │  New  │ ────────► │ Active │ ──────► │Completing │ ────────► │ Completed │
    └───────┘           └────────┘         └───────────┘           └───────────┘
                            │                                           ▲
                            │ cancel()    ┌────────────┐   finish      │
                            └────────────►│ Cancelling │──────────────►│
                                          └────────────┘    ┌──────────┘
                                                            │
                                                       ┌────────────┐
                                                       │ Cancelled  │
                                                       └────────────┘
```

| Состояние | `isActive` | `isCompleted` | `isCancelled` |
|---|---|---|---|
| New (lazy) | `false` | `false` | `false` |
| Active | `true` | `false` | `false` |
| Completing | `true` | `false` | `false` |
| Cancelling | `false` | `false` | `true` |
| Cancelled | `false` | `true` | `true` |
| Completed | `false` | `true` | `false` |

### Отмена корутин

Отмена в корутинах — **кооперативная**. Корутина должна **проверять** отмену:

```kotlin
val job = launch {
    repeat(1000) { i ->
        if (!isActive) return@launch  // проверяем отмену вручную
        println("Working $i...")
        Thread.sleep(100) // ⚠️ не suspend — не проверяет отмену!
    }
}

delay(500)
job.cancel()
```

Все стандартные `suspend`-функции (`delay`, `yield`, `withContext`, ...) проверяют отмену автоматически и бросают `CancellationException`:

```kotlin
val job = launch {
    repeat(1000) { i ->
        println("Working $i...")
        delay(100) // ✅ проверяет отмену, бросит CancellationException
    }
}

delay(500)
job.cancelAndJoin() // отменяем и ждём завершения
println("Cancelled")
```

> `CancellationException` — особое исключение: оно **не считается ошибкой** и не отменяет родительскую корутину.

### Иерархия Job-ов (parent-child)

Каждая корутина, запущенная внутри scope, становится **дочерней** по отношению к Job этого scope:

```kotlin
val parentJob = launch {
    val child1 = launch { delay(1000); println("child1 done") }
    val child2 = launch { delay(2000); println("child2 done") }

    println("Children: ${coroutineContext[Job]?.children?.toList()?.size}") // 2
}
```

Правила иерархии:
- Родитель **ждёт** завершения всех детей.
- `parent.cancel()` → отменяет **всех** детей.
- Ошибка в ребёнке → отменяет **родителя** (и всех братьев).

### SupervisorJob

`SupervisorJob` нарушает последнее правило: ошибка в одном ребёнке **не отменяет** остальных:

```kotlin
val supervisor = CoroutineScope(SupervisorJob() + Dispatchers.Default)

supervisor.launch {
    throw RuntimeException("Ошибка в задаче 1") // не убьёт задачу 2
}

supervisor.launch {
    delay(1000)
    println("Задача 2 завершена") // выполнится!
}
```

Аналогично работает `supervisorScope`:

```kotlin
suspend fun loadDashboard() = supervisorScope {
    val user = async { fetchUser() }
    val recommendations = async { fetchRecommendations() } // может упасть

    // Даже если recommendations упадёт, user не будет отменён
    Dashboard(
        user.await(),
        recommendations.runCatching { await() }.getOrDefault(emptyList()),
    )
}
```

### Упражнения: Job

1. Запустите код и объясните вывод. Что произойдёт, если убрать `delay(100)` из дочерней корутины?

    ```kotlin
    fun main() = runBlocking {
        val job = launch {
            repeat(10) { i ->
                println("Working $i")
                delay(100)
            }
        }

        delay(350)
        println("Cancelling...")
        job.cancelAndJoin()
        println("Done")
    }
    ```

2. Перепишите пример выше, заменив `delay(100)` на `Thread.sleep(100)`. Что изменится? Как сделать так, чтобы отмена всё равно работала (подсказка: `isActive` или `ensureActive()`)?

3. Напишите программу, демонстрирующую разницу между `Job` и `SupervisorJob`:
    - запустите 3 дочерние корутины;
    - вторая корутина бросает исключение через 500 мс;
    - покажите, что с обычным `Job` все корутины отменяются, а с `SupervisorJob` — только упавшая.

4. Объясните письменно: почему `CancellationException` не считается ошибкой? Что произойдёт, если поймать `CancellationException` и не пробросить его дальше?

---

## Билдеры корутин

### Обзор билдеров

Билдеры — это функции, которые **создают и запускают** корутины.

| Билдер | Возвращает | Ожидает детей | Контекст вызова | Назначение |
|---|---|---|---|---|
| `launch` | `Job` | Нет (fire-and-forget) | `CoroutineScope` | Запуск без результата |
| `async` | `Deferred<T>` | Нет (результат через `await`) | `CoroutineScope` | Запуск с результатом |
| `coroutineScope` | `T` | Да (ждёт всех детей) | `suspend` | Structured concurrency |
| `supervisorScope` | `T` | Да (ждёт всех детей) | `suspend` | Structured concurrency без каскадной отмены |
| `runBlocking` | `T` | Да (блокирует поток) | Любой | Мост из блокирующего мира |

### launch — fire-and-forget

`launch` запускает корутину и возвращает `Job`. Результат не возвращается:

```kotlin
fun main() = runBlocking {
    val job = launch {
        delay(500)
        println("Task completed")
    }

    println("Task launched, waiting...")
    job.join() // ожидаем завершения
    println("Done")
}
```

Параметры `launch`:

```kotlin
launch(
    context = Dispatchers.IO + CoroutineName("bg-task"),
    start = CoroutineStart.LAZY, // не запускать сразу
) {
    // ...
}
```

| `CoroutineStart` | Поведение |
|---|---|
| `DEFAULT` | Запуск немедленно |
| `LAZY` | Запуск при `join()` или `start()` |
| `ATOMIC` | Запуск немедленно, не отменяется до первой точки приостановки |
| `UNDISPATCHED` | Запуск немедленно в текущем потоке, без диспатчинга |

### async — запуск с результатом

`async` возвращает `Deferred<T>` — наследник `Job` с методом `await()`:

```kotlin
fun main() = runBlocking {
    val deferred: Deferred<Int> = async {
        delay(500)
        42
    }

    println("Computing...")
    val result = deferred.await() // приостанавливается до получения результата
    println("Result: $result")    // Result: 42
}
```

### Параллельная декомпозиция с async

Главная сила `async` — **параллельный** запуск нескольких задач:

```kotlin
suspend fun loadDashboard(): Dashboard = coroutineScope {
    // Запускаем параллельно
    val user = async { fetchUser() }           // ~500 мс
    val orders = async { fetchOrders() }       // ~800 мс
    val recommendations = async { fetchRecs() } // ~300 мс

    // Общее время ≈ max(500, 800, 300) = 800 мс, а не 1600 мс
    Dashboard(user.await(), orders.await(), recommendations.await())
}
```

> ⚠️ **Частая ошибка** — последовательный `async`:
>
> ```kotlin
> // ❌ Нет параллелизма! await() вызывается сразу
> val user = async { fetchUser() }.await()
> val orders = async { fetchOrders() }.await()
>
> // ✅ Правильно: сначала запустить все, потом await
> val user = async { fetchUser() }
> val orders = async { fetchOrders() }
> Dashboard(user.await(), orders.await())
> ```

### Обработка ошибок в async

Исключение в `async` **не бросается** при запуске — оно **откладывается** до вызова `await()`:

```kotlin
val deferred = async {
    throw IllegalStateException("Oops")
}

// Исключение бросится здесь:
try {
    deferred.await()
} catch (e: IllegalStateException) {
    println("Caught: ${e.message}")
}
```

> Но! В structured concurrency исключение в `async` всё равно **отменяет родительский scope**. Используйте `supervisorScope`, если хотите изолировать ошибки.

### coroutineScope — structured concurrency builder

`coroutineScope` — это `suspend`-функция, которая создаёт новый scope и **ждёт завершения всех дочерних корутин**:

```kotlin
suspend fun fetchUserWithOrders(userId: Int): UserWithOrders = coroutineScope {
    val user = async { userService.getUser(userId) }
    val orders = async { orderService.getOrders(userId) }
    UserWithOrders(user.await(), orders.await())
}
// Функция вернёт результат только когда обе задачи завершатся
```

Ключевые свойства:
- **Не блокирует** поток (в отличие от `runBlocking`).
- **Ждёт** всех дочерних корутин.
- **Пробрасывает** исключения из детей.
- Наследует контекст вызывающей корутины.

### runBlocking — мост из блокирующего мира

`runBlocking` **блокирует текущий поток** до завершения корутины. Используется для:
- функции `main()`;
- тестов;
- интеграции с блокирующим кодом.

```kotlin
fun main() = runBlocking {
    // Здесь мы в корутине, можем вызывать suspend-функции
    val result = fetchData()
    println(result)
}
```

> ⚠️ **Никогда** не вызывайте `runBlocking` внутри корутины — это заблокирует поток и может привести к deadlock.

### Сравнение: когда что использовать

```
main() или тест?
  └─► runBlocking

Нужен результат?
  ├─ Да → async { ... } + await()
  └─ Нет → launch { ... }

Нужно запустить параллельные задачи внутри suspend-функции?
  └─► coroutineScope { async { } + async { } }

Ошибка в одной задаче не должна убивать остальные?
  └─► supervisorScope { ... }
```

### Упражнения: Билдеры корутин

1. Напишите функцию, которая параллельно загружает данные из трёх «сервисов» (эмулируйте через `delay`) и возвращает объединённый результат. Замерьте время выполнения и убедитесь, что оно примерно равно максимальному из трёх `delay`:

    ```kotlin
    suspend fun fetchA(): String { delay(300); return "A" }
    suspend fun fetchB(): String { delay(500); return "B" }
    suspend fun fetchC(): String { delay(200); return "C" }

    suspend fun fetchAll(): Triple<String, String, String> = TODO()
    ```

2. Напишите функцию `retryAsync`, которая:
    - принимает `times: Int` и `block: suspend () -> T`;
    - пытается выполнить `block` до `times` раз;
    - при неудаче ждёт `delay(100 * attempt)` перед повтором;
    - если все попытки исчерпаны — бросает последнее исключение.

    ```kotlin
    suspend fun <T> retryAsync(times: Int, block: suspend () -> T): T = TODO()
    ```

3. Объясните, что произойдёт в каждом случае и почему:

    ```kotlin
    // Случай A
    fun main() = runBlocking {
        launch {
            throw RuntimeException("Boom")
        }
        delay(1000)
        println("This line?")
    }

    // Случай B
    fun main() = runBlocking {
        val deferred = async {
            throw RuntimeException("Boom")
        }
        delay(1000)
        println("This line?")
    }

    // Случай C
    fun main() = runBlocking {
        supervisorScope {
            launch {
                throw RuntimeException("Boom")
            }
            delay(1000)
            println("This line?")
        }
    }
    ```

4. Напишите программу, которая запускает 100_000 корутин через `launch`, каждая из которых делает `delay(1000)` и инкрементирует `AtomicInteger`. Убедитесь, что все 100_000 завершились. Попробуйте сделать то же самое с потоками (`thread { }`) — что произойдёт?

---

## Вопросы

1. Чем structured concurrency отличается от запуска корутин через `GlobalScope`? Какие проблемы решает structured concurrency?
2. В чём разница между `launch` и `async`? Когда стоит использовать каждый из них?
3. Почему отмена корутин называется «кооперативной»? Что произойдёт, если корутина выполняет CPU-интенсивную работу без проверки `isActive`?
4. Как выбрать правильный диспатчер? Что произойдёт, если запустить блокирующий I/O на `Dispatchers.Default`?
5. В чём разница между `Job` и `SupervisorJob`? Приведите пример, когда `SupervisorJob` необходим.
6. Почему `runBlocking` нельзя вызывать внутри корутины? Опишите сценарий deadlock-а.

---

## Итоги

1. **suspend-функции** компилируются в CPS: компилятор добавляет `Continuation` и строит конечный автомат по точкам приостановки.
2. **Приостановка ≠ блокировка**: при приостановке поток освобождается, корутина ждёт в памяти.
3. **CoroutineContext** — неизменяемая структура «ключ → элемент» (`Job`, `Dispatcher`, `CoroutineName`, `ExceptionHandler`), комбинируется через `+`.
4. **Диспатчеры** определяют, в каком потоке выполняется корутина: `Default` для CPU, `IO` для блокирующего I/O, `Main` для UI.
5. **CoroutineScope** — граница жизни корутин, обеспечивает structured concurrency: родитель ждёт детей, отмена каскадная.
6. **Job** — дескриптор корутины с жизненным циклом (New → Active → Completing → Completed/Cancelled), поддерживает иерархию parent-child.
7. **SupervisorJob** изолирует ошибки: падение одного ребёнка не отменяет остальных.
8. **launch** — fire-and-forget (возвращает `Job`), **async** — с результатом (возвращает `Deferred<T>`, получаем через `await()`).
9. **coroutineScope** — structured concurrency builder для параллельной декомпозиции внутри suspend-функций.
10. **runBlocking** — мост из блокирующего мира в корутины; используется в `main()` и тестах, **не** внутри корутин.
