---

# lecture-04: Многопоточность и корутины

---

<img src="https://nicholasgribanov.name/wp-content/uploads/2020/12/1428558658_521718796.jpg" width="650" />

---

# Зачем нужна многопоточность?

Современные процессоры имеют несколько ядер. Один поток использует только одно ядро.

**Задачи, где важна многопоточность:**
- Обработка HTTP-запросов (тысячи одновременно)
- Параллельные вычисления (обработка данных, ML)
- UI-приложения (не блокировать интерфейс)
- I/O-операции (сеть, диск, БД)

> Без многопоточности мы тратим дорогое железо впустую

---

# Многопоточность vs Конкурентность vs Асинхронность

---

## Три разных понятия

| Понятие | Суть |
|---|---|
| **Параллелизм (Parallelism)** | Задачи выполняются **одновременно** на разных ядрах |
| **Конкурентность (Concurrency)** | Задачи **пересекаются во времени**, но не обязательно одновременно |
| **Асинхронность (Asynchrony)** | Задача **не блокирует** вызывающий поток, результат придёт позже |

---

## Аналогия: кофейня

- **Параллелизм** — два бариста делают два кофе одновременно
- **Конкурентность** — один бариста жонглирует тремя заказами: пока один варится, готовит молоко для другого
- **Асинхронность** — вы сделали заказ, сели за столик и ждёте, пока вас позовут. Вы не стоите у стойки

> Конкурентность — про **структуру** программы. Параллелизм — про **исполнение**.

---

## Конкурентность ≠ Параллелизм

```
Конкурентность (1 ядро):           Параллелизм (2 ядра):

Поток A: ██░░██░░██               Поток A: ██████████
Поток B: ░░██░░██░░               Поток B: ██████████
         ──────────► время                  ──────────► время

Задачи чередуются                  Задачи выполняются одновременно
```

- Конкурентность возможна на **одном** ядре (переключение контекста)
- Параллелизм требует **нескольких** ядер
- Конкурентная программа **может** исполняться параллельно, но не обязана

---

## Синхронность vs Асинхронность

```kotlin
// Синхронный вызов — поток заблокирован, пока не придёт ответ
fun getUser(id: Int): User {
    val response = httpClient.get("https://api.example.com/users/$id")  // ждём...
    return parseUser(response)
}

// Асинхронный вызов — поток свободен, результат придёт в callback
fun getUserAsync(id: Int, callback: (User) -> Unit) {
    httpClient.getAsync("https://api.example.com/users/$id") { response ->
        callback(parseUser(response))
    }
}
```

- Синхронный вызов: поток **стоит и ждёт**
- Асинхронный вызов: поток **отправил запрос и пошёл дальше**

---

# JVM-потоки

---

## Что такое поток?

- **Поток (Thread)** — единица исполнения внутри процесса
- Все потоки одного процесса **разделяют** общую память (heap)
- У каждого потока свой **стек вызовов** (call stack)

```
Процесс (JVM)
┌─────────────────────────────────────┐
│           Общая память (Heap)       │
│                                     │
│  ┌─────────┐ ┌─────────┐ ┌───────┐  │
│  │ Поток 1 │ │ Поток 2 │ │Поток 3│  │
│  │ (стек)  │ │ (стек)  │ │(стек) │  │
│  └─────────┘ └─────────┘ └───────┘  │
└─────────────────────────────────────┘
```

- Общая память — и благо (легко обмениваться данными), и проблема (гонки данных)

---

## Runnable и Thread

```kotlin
// Способ 1: через Runnable
val task = Runnable {
    println("Hello from ${Thread.currentThread().name}")
}
val thread = Thread(task)
thread.start()

// Способ 2: наследование от Thread (антипаттерн)
val thread2 = object : Thread() {
    override fun run() {
        println("Hello from $name")
    }
}
thread2.start()

// Способ 3: Kotlin-хелпер
val thread3 = thread(name = "my-thread") {
    println("Hello from ${Thread.currentThread().name}")
}
```

- `Runnable` — задача (что делать)
- `Thread` — поток (кто делает)
- Разделяй **задачу** и **исполнителя**

---

## Жизненный цикл потока

```
          start()
NEW ──────────────► RUNNABLE ◄───────────┐
                       │                 │
                       │ планировщик ОС  │
                       ▼                 │
                    RUNNING              │
                    /    \               │
           завершение    блокировка      │
               │         (sleep, join,   │
               ▼          wait, I/O)     │
          TERMINATED         │           │
                             ▼           │
                          BLOCKED ───────┘
                           (пробуждение)
```

- `start()` — запускает поток (вызывать `run()` напрямую — ошибка!)
- `join()` — ждать завершения другого потока
- `sleep()` — приостановить поток на время
- `interrupt()` — отправить сигнал прерывания

---

## start() vs run() — частая ошибка

```kotlin
val thread = Thread {
    println("Работаю в потоке: ${Thread.currentThread().name}")
}

// ❌ Вызывает run() в текущем потоке — никакой многопоточности!
thread.run()    // Работаю в потоке: main

// ✅ Запускает новый поток
thread.start()  // Работаю в потоке: Thread-0
```

- `run()` — обычный вызов метода, код выполняется в **текущем** потоке
- `start()` — создаёт новый поток ОС и вызывает `run()` в **нём**

---

## join() — ожидание завершения

```kotlin
fun main() {
    var result = 0

    val thread = thread {
        Thread.sleep(1000)
        result = 42
    }

    println("Результат до join: $result")   // 0 — поток ещё работает

    thread.join()  // ждём завершения

    println("Результат после join: $result") // 42
}
```

- Без `join()` — главный поток может завершиться раньше дочернего
- `join()` **блокирует** текущий поток до завершения целевого

---

## interrupt() — кооперативная отмена

```kotlin
val worker = thread {
    while (!Thread.currentThread().isInterrupted) {
        println("Работаю...")
        try {
            Thread.sleep(500)
        } catch (e: InterruptedException) {
            println("Получил сигнал прерывания, завершаюсь")
            break
        }
    }
}

Thread.sleep(2000)
worker.interrupt()  // отправляем сигнал
worker.join()
```

- `interrupt()` — **не убивает** поток, а устанавливает флаг
- Поток должен **сам** проверять флаг и корректно завершаться
- `sleep()`, `join()`, `wait()` бросают `InterruptedException` при прерывании

---

## Демон-потоки

```kotlin
val daemon = thread(isDaemon = true) {
    while (true) {
        println("Фоновая задача...")
        Thread.sleep(500)
    }
}

// Когда main завершится, daemon-поток будет убит автоматически
Thread.sleep(2000)
println("Main завершён")
```

- **Daemon-поток** — фоновый поток, не мешающий завершению JVM
- JVM завершается, когда завершились все **не-daemon** потоки
- Используйте для фоновых задач: мониторинг, метрики, heartbeat, периодическая очистка кэша

---

## Стоимость потоков

```kotlin
// ⚠️ Каждый поток — это ресурсы ОС
fun main() {
    val threads = (1..10_000).map {
        thread {
            Thread.sleep(10_000)
        }
    }
    println("Создано ${threads.size} потоков")
    threads.forEach { it.join() }
}
```

- Один поток ≈ **512 KB – 1 MB** стека
- 10 000 потоков ≈ **5–10 GB** только на стеки
- Создание потока — системный вызов (дорого)
- Переключение контекста — тоже дорого

> Потоки — ценный ресурс. Нельзя создавать по потоку на каждую задачу.

---

# Пулы потоков

---

## Идея: переиспользование потоков

Вместо того чтобы создавать поток на каждую задачу — создаём **пул** заранее и отправляем задачи в очередь.

```
                        ┌──────────┐
  Задача 1 ──►          │ Поток 1  │ ──► выполняет задачу 1, потом берёт задачу 4
  Задача 2 ──► Очередь  │ Поток 2  │ ──► выполняет задачу 2, потом берёт задачу 5
  Задача 3 ──►          │ Поток 3  │ ──► выполняет задачу 3, потом берёт задачу 6
  Задача 4 ──►          └──────────┘
  Задача 5 ──►
  Задача 6 ──►
```

- Потоки **не уничтожаются** после выполнения задачи — берут следующую
- Экономим на создании/уничтожении потоков
- Контролируем **максимальную степень параллелизма**

---

## ExecutorService

```kotlin
// imports:
//   java.util.concurrent.Executors
//   java.util.concurrent.TimeUnit

// Пул из 4 потоков
val pool = java.util.concurrent.Executors.newFixedThreadPool(4)

repeat(10) { i ->
    pool.submit {
        println("Задача $i в потоке ${Thread.currentThread().name}")
    }
}

pool.shutdown() // запрещаем новые задачи, пул завершится после выполнения очереди
pool.awaitTermination(1, java.util.concurrent.TimeUnit.MINUTES) // ✅ ждём завершения (или timeout)
```

- `submit()` — отправить задачу в пул
- `shutdown()` — завершить пул после выполнения всех задач
- `shutdownNow()` — попытаться прервать все задачи немедленно

---

## Виды пулов

| Пул | Описание | Когда использовать |
|---|---|---|
| `newFixedThreadPool(n)` | Фиксированное число потоков | Известная нагрузка, CPU-задачи |
| `newCachedThreadPool()` | Потоки создаются по необходимости, переиспользуются | Много коротких I/O-задач |
| `newSingleThreadExecutor()` | Один поток, гарантия порядка | Последовательная обработка |
| `newScheduledThreadPool(n)` | Отложенные и периодические задачи | Таймеры, cron-задачи |

```kotlin
// Периодическое выполнение
val scheduler = Executors.newScheduledThreadPool(1)
scheduler.scheduleAtFixedRate(
    { println("Heartbeat: ${System.currentTimeMillis()}") },
    0, 1, TimeUnit.SECONDS  // начать сразу, повторять каждую секунду
)
```

---

# Future — отложенный результат

---

## Идея Future

```kotlin
val pool = Executors.newFixedThreadPool(2)

// submit возвращает Future — "обещание" результата
val future: Future<Int> = pool.submit(Callable {
    Thread.sleep(1000)  // долгое вычисление
    42
})

println("Задача запущена, делаем что-то ещё...")

val result = future.get()  // ⚠️ блокирует поток до получения результата
println("Результат: $result")

pool.shutdown()
```

- `Future<T>` — контейнер для результата, который **будет** получен
- `get()` — **блокирует** текущий поток до готовности результата
- `get(timeout, unit)` — ждать не дольше указанного времени

---

## Проблемы Future

```kotlin
// ❌ Проблема: цепочка зависимых вызовов блокирует поток на каждом шаге
val userFuture = pool.submit(Callable { fetchUser(id) })
val user = userFuture.get()  // блокировка

val ordersFuture = pool.submit(Callable { fetchOrders(user) })
val orders = ordersFuture.get()  // ещё блокировка

val reportFuture = pool.submit(Callable { buildReport(orders) })
val report = reportFuture.get()  // и ещё...
```

**Проблемы:**
- Каждый `get()` — **блокировка** потока
- Нельзя скомпоновать: "когда A завершится — запусти B"
- Нет обработки ошибок в цепочке
- Нет отмены зависимых задач

---

## CompletableFuture — композиция

```kotlin
// import: java.util.concurrent.CompletableFuture

// ✅ Цепочка без блокировок
java.util.concurrent.CompletableFuture
    .supplyAsync { fetchUser(id) }              // по умолчанию использует ForkJoinPool.commonPool()
    .thenApply { user -> fetchOrders(user) }    // когда готово — преобразовать
    .thenApply { orders -> buildReport(orders) }
    .thenAccept { report -> println(report) }   // потребить результат
    .exceptionally { error ->                   // обработать ошибку
        println("Ошибка: ${error.message}")
        null
    }
```

- Результат одного шага **автоматически** передаётся в следующий
- Поток не блокируется между шагами
- Есть комбинаторы: `thenCombine`, `allOf`, `anyOf`

> CompletableFuture — шаг к асинхронности, но код быстро становится громоздким. Корутины решат это элегантнее.

---

# Классические проблемы многопоточности

---

## Race Condition — гонка данных

```kotlin
var counter = 0

fun main() {
    val threads = (1..100).map {
        thread {
            repeat(1000) {
                counter++  // ⚠️ не атомарная операция!
            }
        }
    }
    threads.forEach { it.join() }
    println("Ожидали: 100000, получили: $counter")  // почти наверняка < 100000
}
```

- `counter++` — это **три** операции: прочитать, прибавить, записать
- Два потока могут прочитать **одно и то же** значение и записать одинаковый результат
- Результат зависит от порядка исполнения — **недетерминизм**

---

## Race Condition — что происходит

```
Поток A                  Поток B               counter
─────────────────────────────────────────────────────────
read counter (= 5)                                5
                         read counter (= 5)       5
add 1 → 6                                         5
                         add 1 → 6                5
write 6                                           6
                         write 6                  6  ← потеряли инкремент!
```

- Два инкремента выполнены, но `counter` увеличился только на 1
- Это **потерянное обновление (lost update)** — классический race condition

---

<img src="https://encrypted-tbn0.gstatic.com/images?q=tbn:ANd9GcSZrMCH1TrpCxmmQ9RI74_CO7lUyRnHqYO3PQ&s" width="300" />

---

## Deadlock — взаимная блокировка

```kotlin
val lockA = ReentrantLock()
val lockB = ReentrantLock()

// Поток 1: захватывает A, потом B
val t1 = thread {
    lockA.lock()
    Thread.sleep(100)  // даём время потоку 2 захватить lockB
    lockB.lock()       // ⚠️ ждём lockB, который держит поток 2
    // ... никогда не дойдём сюда
}

// Поток 2: захватывает B, потом A
val t2 = thread {
    lockB.lock()
    Thread.sleep(100)
    lockA.lock()       // ⚠️ ждём lockA, который держит поток 1
    // ... никогда не дойдём сюда
}
```

- Поток 1 ждёт Поток 2, а Поток 2 ждёт Поток 1 — **оба стоят вечно**
- Программа зависает, потоки не завершатся

---

## Deadlock — условия возникновения

Дедлок возникает, когда **одновременно** выполнены все 4 условия (условия Коффмана):

1. **Взаимное исключение** — ресурс может быть захвачен только одним потоком
2. **Удержание и ожидание** — поток держит один ресурс и ждёт другой
3. **Невозможность отнять** — ресурс нельзя отобрать принудительно
4. **Циклическое ожидание** — цепочка потоков, каждый ждёт ресурс следующего

> **Решение:** нарушить хотя бы одно условие. Чаще всего — **упорядочить** захват ресурсов (всегда сначала A, потом B).

---

## Livelock и Starvation

**Livelock** — потоки **активны**, но не делают полезной работы:

```
Поток A: "Я уступлю B" → отпускает ресурс → ждёт
Поток B: "Я уступлю A" → отпускает ресурс → ждёт
Поток A: "О, свободно!" → захватывает → "Я уступлю B" → ...
```

- Как два человека в коридоре, которые вечно уступают друг другу дорогу

**Starvation (голодание)** — поток **никогда не получает** ресурс:

- Высокоприоритетные потоки постоянно «перебивают»
- Поток вечно стоит в очереди, но его обгоняют

> Решение starvation: **честные (fair)** блокировки — `ReentrantLock(fair = true)`

---

<img src="https://www.meme-arsenal.com/memes/867228557fb0724bf49579d77c29bae2.jpg" width="650" />

---

# Механизмы синхронизации

---

## Проблема: доступ к общей памяти

Когда несколько потоков читают и пишут в одни и те же данные, нужно **координировать** доступ.

```kotlin
// ❌ Небезопасно: много потоков модифицируют один список
val list = mutableListOf<Int>()

val threads = (1..10).map { i ->
    thread {
        repeat(1000) { list.add(i) }  // гонки: потеря данных/повреждение структуры списка
    }
}
threads.forEach { it.join() }
println(list.size)  // ожидали 10000, получили... кто знает
```

**Решение:** ограничить доступ к критической области — участку кода, где в каждый момент времени может находиться только **один** поток.

---

## Mutex и критическая область

**Mutex (Mutual Exclusion)** — примитив, который гарантирует: в критической области находится не более одного потока.

```
Поток A: захватил → [работает с данными] → освободил
Поток B:            ждёт...               → захватил → [работает] → освободил
```

- Если mutex захвачен — другие потоки **блокируются** до освобождения
- Простейший и самый распространённый механизм синхронизации

---

## synchronized — JVM vs Kotlin

```kotlin
// Java-стиль: synchronized на мониторе объекта
class Counter {
    private var count = 0
    private val lock = Any()

    fun increment() {
        synchronized(lock) {  // захват монитора
            count++           // критическая область
        }
    }
}
```

- В JVM каждый объект имеет **монитор** — встроенный mutex
- `synchronized` захватывает монитор объекта, выполняет блок, освобождает
- В Java: `synchronized` — ключевое слово. В Kotlin: `synchronized` — **функция**
- Kotlin не поддерживает `synchronized` на методе, но есть `@Synchronized`

---

## Lock и ReentrantLock

```kotlin
// import: java.util.concurrent.locks.ReentrantLock

class SafeCounter {
    private var count = 0
    private val lock = java.util.concurrent.locks.ReentrantLock()

    fun increment() {
        lock.lock()
        try {
            count++
        } finally {
            lock.unlock()  // всегда в finally!
        }
    }

    // Kotlin-идиома: используйте withLock
    fun decrement() {
        lock.withLock {
            count--
        }
    }
}
```

---

| | `synchronized` | `ReentrantLock` |
|---|---|---|
| Гибкость | Минимальная | Высокая |
| Fairness | Нет | Да (`fair = true`) |
| `tryLock` | Нет | Да |
| Condition | `wait`/`notify` | `newCondition()` |
| Освобождение | Автоматическое | Ручное (finally!) |

---

## Condition — ожидание по условию

```kotlin
class BoundedBuffer<T>(private val capacity: Int) {
    private val items = mutableListOf<T>()
    private val lock = ReentrantLock()
    private val notFull = lock.newCondition()   // можно добавлять
    private val notEmpty = lock.newCondition()  // можно забирать

    fun put(item: T) = lock.withLock {
        while (items.size == capacity) notFull.await()  // ждём места
        items.add(item)
        notEmpty.signal()  // сообщаем: появился элемент
    }

    fun take(): T = lock.withLock {
        while (items.isEmpty()) notEmpty.await()  // ждём элемент
        val item = items.removeFirst()
        notFull.signal()  // сообщаем: появилось место
        return item
    }
}
```

- `await()` — отпустить lock и уснуть, пока не разбудят
- `signal()` — разбудить один ожидающий поток
- Всегда проверяйте условие в **`while`**, не в `if` (spurious wakeup!)

---

## ReadWriteLock

```kotlin
// imports:
//   java.util.concurrent.locks.ReentrantReadWriteLock
//   kotlin.concurrent.read
//   kotlin.concurrent.write

class Cache<K, V> {
    private val map = mutableMapOf<K, V>()
    private val rwLock = java.util.concurrent.locks.ReentrantReadWriteLock()

    fun get(key: K): V? = rwLock.read {   // много читателей одновременно
        map[key]
    }

    fun put(key: K, value: V) = rwLock.write {  // только один писатель
        map[key] = value
    }
}
```

- **Много читателей или один писатель** — оптимизация для сценариев "читают часто, пишут редко"
- Read lock не мешает другим читателям, но блокирует писателей
- Write lock блокирует всех

---

## Semaphore — ограничение параллелизма

```kotlin
// import: java.util.concurrent.Semaphore

// Максимум 3 одновременных соединения к БД
val connectionPool = java.util.concurrent.Semaphore(3)

fun queryDatabase(sql: String): String {
    connectionPool.acquire()  // занять слот (или ждать)
    try {
        // работа с БД
        return executeQuery(sql)
    } finally {
        connectionPool.release()  // освободить слот
    }
}
```

- **Mutex** = Semaphore(1) — частный случай
- Semaphore(N) — пускает до N потоков одновременно
- Идеально для ограничения доступа к ограниченным ресурсам

---

## ThreadLocal — данные потока

```kotlin
// Каждый поток получает свою копию переменной
val requestId = ThreadLocal.withInitial { "none" }

fun handleRequest(id: String) {
    requestId.set(id)
    processStep1()  // все функции в цепочке
    processStep2()  // видят свой requestId
}

fun processStep1() {
    println("Шаг 1, request: ${requestId.get()}")  // без передачи параметра!
}
```

- ThreadLocal — переменная с **отдельным значением для каждого потока**
- Типичные сценарии: request ID, текущий пользователь, форматеры дат

> ⚠️ ThreadLocal плохо сочетается с корутинами: корутина может продолжиться в другом потоке!

---

# Блокирующие коллекции

---

## Идея: очередь между потоками

Одни потоки **кладут** элементы, другие **забирают**. Если очередь пуста/полна — поток **блокируется** и ждёт.

```
Производитель A ──►┌──────────────────┐──► Потребитель X
Производитель B ──►│ BlockingQueue    │──► Потребитель Y
                   └──────────────────┘
```

- Паттерн **Производитель–Потребитель (Producer-Consumer)**
- Очередь сама заботится о синхронизации — никаких lock-ов вручную

---

## Виды блокирующих очередей

| Очередь | Особенность | Когда использовать |
|---|---|---|
| `SynchronousQueue` | Емкость 0: передача из рук в руки | Handoff между потоками |
| `ArrayBlockingQueue` | Фиксированная емкость, массив | Ограниченный буфер |
| `LinkedBlockingQueue` | Неограниченная (или ограниченная), связный список | Большинство случаев |
| `PriorityBlockingQueue` | Элементы извлекаются по приоритету | Задачи с приоритетами |

---

```kotlin
val queue = ArrayBlockingQueue<String>(10)

// Производитель
thread {
    repeat(20) { i ->
        queue.put("задача-$i")  // заблокируется, если очередь полна
    }
}

// Потребитель
thread {
    repeat(20) {
        val task = queue.take()  // заблокируется, если очередь пуста
        println("Обработано: $task")
    }
}
```

---

# Неблокирующие коллекции

---

## Concurrent-коллекции

Не блокируют потоки, а используют **алгоритмы без блокировок** (CAS, lock-free).

| Коллекция | Описание |
|---|---|
| `ConcurrentHashMap` | Потокобезопасный HashMap (CAS + тонкие блокировки) |
| `ConcurrentLinkedQueue` | Неблокирующая очередь (lock-free) |
| `ConcurrentLinkedDeque` | Неблокирующая двусторонняя очередь |
| `ConcurrentSkipListMap` | Потокобезопасный сортированный Map |

---

## ConcurrentHashMap vs synchronized Map

```kotlin
// ❌ Медленно: блокирует всю Map целиком
val syncMap = Collections.synchronizedMap(mutableMapOf<String, Int>())

// ✅ Обычно быстрее: операции используют CAS и при необходимости блокировки на уровне отдельных участков структуры
val concurrentMap = ConcurrentHashMap<String, Int>()

// Атомарные операции:
concurrentMap.putIfAbsent("key", 1)
concurrentMap.compute("key") { _, v -> (v ?: 0) + 1 }
concurrentMap.merge("key", 1) { old, new -> old + new }
```

- `synchronizedMap` — один lock на всю карту → один поток работает, остальные ждут
- `ConcurrentHashMap` — допускает параллельный доступ, обычно без одного глобального lock-а

---

# Модель памяти JVM

---

## Почему нужна JMM?

Потоки не всегда видят **актуальные** значения переменных:

```
ОПЕРАТИВНАЯ ПАМЯТЬ (ОБЩАЯ)
┌───────────────────────────────┐
│         x = 0, flag = false   │
└───────────────────────────────┘
        ↑                      ↑
        │ (может быть          │ (может быть
        │  не синхронизирован) │  устаревшим)
┌─────────────┐     ┌─────────────┐
│ Кэш потока A│     │ Кэш потока B│
│ x = 42      │     │ x = 0       │
│ flag = true │     │ flag = false│
└─────────────┘     └─────────────┘
```

- Каждый поток может работать с **локальным кэшем** (регистры, L1/L2 кэш)
- Изменения могут быть **не видны** другим потокам
- Компилятор и процессор могут **переупорядочивать** инструкции

---

## Happens-Before

**JMM (Java Memory Model)** определяет правила **happens-before** — гарантии видимости изменений.

Если действие A **happens-before** действие B, то B гарантированно видит все изменения A.

**Основные правила:**
- `synchronized`-блок: unlock → следующий лок
- `volatile`-запись → следующее чтение той же переменной
- `thread.start()` → весь код внутри потока
- `thread.join()` → код после join

> Без happens-before — **никаких гарантий** о видимости изменений между потоками.

---

## volatile переменные

```kotlin
// ❌ Без volatile — поток B может никогда не увидеть изменение
var running = true

val worker = thread {
    while (running) {  // может крутиться вечно!
        doWork()
    }
}

Thread.sleep(1000)
running = false  // поток может не заметить
```

```kotlin
// ✅ С volatile — изменение гарантированно видно всем потокам
@Volatile
var running = true
```

- `@Volatile` в Kotlin = `volatile` в Java
- Гарантирует: чтение всегда видит **последнюю** запись
- **Не** гарантирует атомарность! `volatile counter++` — всё ещё гонка

---

## volatile: пример с несколькими вариантами вывода

```kotlin
var x = 0; var y = 0
var seenX = -1; var seenY = -1
val t1 = thread { x = 1; y = 1 }
val t2 = thread {
    // Важно: не println внутри потока — он сам по себе синхронизирован и может искажать пример
    seenY = y; seenX = x
}
t1.join(); t2.join()
println("y=$seenY, x=$seenX")
```

Возможные результаты (все легальны!):

| Вывод | Почему |
|---|---|
| `y=0, x=0` | t2 выполнился до t1 |
| `y=1, x=1` | t2 выполнился после t1 |
| `y=0, x=1` | t2 выполнился между x=1 и y=1 |
| `y=1, x=0` | Переупорядочивание инструкций! Без volatile/synchronized это возможно |

> Без синхронизации JVM имеет право переупорядочить операции для оптимизации.

---

# Атомарные операции

---

## Проблема volatile + инкремент

```kotlin
// ❌ volatile не спасает от гонки на инкременте
@Volatile
var counter = 0

fun main() {
    val threads = (1..100).map {
        thread { repeat(1000) { counter++ } }  // всё ещё гонка!
    }
    threads.forEach { it.join() }
    println(counter)  // < 100000
}
```

- `volatile` гарантирует **видимость**, но не **атомарность**
- `counter++` = read + add + write — между ними может вклиниться другой поток

---

## Atomic-классы

```kotlin
// import: java.util.concurrent.atomic.AtomicInteger

val counter = java.util.concurrent.atomic.AtomicInteger(0)

fun main() {
    val threads = (1..100).map {
        thread { repeat(1000) { counter.incrementAndGet() } }  // ✅ атомарно!
    }
    threads.forEach { it.join() }
    println(counter.get())  // всегда 100000
}
```

- Используют **CAS (Compare-And-Swap)** — аппаратную инструкцию процессора
- Нет блокировок — быстрее `synchronized` для простых операций
- `AtomicInteger`, `AtomicLong`, `AtomicBoolean`, `AtomicReference`

---

## CAS — идея

```
CAS(address, expected, new):
    атомарно {
        если *address == expected:
            *address = new
            вернуть true
        иначе:
            вернуть false  // кто-то успел раньше, попробуй ещё раз
    }
```

```kotlin
// Под капотом incrementAndGet:
fun incrementAndGet(): Int {
    while (true) {
        val current = get()
        val next = current + 1
        if (compareAndSet(current, next)) return next
        // не получилось — повторяем (spin)
    }
}
```

- Без блокировок, но при высокой конкуренции много повторов (contention)

---

## kotlinx-atomicfu

```kotlin
// Java атомики — громоздко + boxing для не-примитивов
val jCounter = java.util.concurrent.atomic.AtomicInteger(0)
jCounter.incrementAndGet()

// kotlinx-atomicfu — идиоматичный Kotlin
// import: kotlinx.atomicfu.atomic
val kCounter = kotlinx.atomicfu.atomic(0)
kCounter.incrementAndGet()
// Под капотом: компилятор преобразует доступ к полям в атомарные операции
```

- **Kotlin Multiplatform**: работает на JVM, JS, Native
- Компиляторный плагин убирает обёртки — нулевой overhead
- Ссылка: [github.com/Kotlin/kotlinx-atomicfu](https://github.com/Kotlin/kotlinx-atomicfu)

---

# Корутины в Kotlin: внутреннее устройство

---

<img src="https://preview.redd.it/coroutines-v0-wp614y0xjqqb1.jpg?auto=webp&s=78e34d7492b62130e812d987862e5443357dacde" width="400" />

---

## Что такое корутина?

- **Корутина** — легковесная единица исполнения, которая может **приостановиться** и **возобновиться** без блокировки потока
- Это **не** поток. Корутины работают **поверх** потоков

```
Поток                     Корутина
─────────────────────   ─────────────────────
• Ресурс ОС               • Объект в памяти
• ~1 MB стека              • ~сотни байт
• Вытесняющая             • Кооперативная
   многозадачность         многозадачность
• Переключает ОС        • Приостанавливается
                              сама в явных точках
```

---

## Кооперативная vs Вытесняющая многозадачность

**Вытесняющая (потоки):** ОС сама решает, когда переключить

```
Поток A: █████|переключение|███████  (ОС прерывает в любой момент)
Поток B: ░░░░░|переключение|░░░░░░░
```

**Кооперативная (корутины):** корутина сама говорит "я приостановлюсь"

```
Корутина A: █████|suspend|██████  (сама уступает место)
Корутина B: ░░░░░░░░░░░|suspend|░░░
```

- Корутина приостанавливается только в **явных точках** (suspension points)
- Между точками — непрерывное выполнение, никто не прервёт

---

## suspend-функции и точки приостановки

```kotlin
suspend fun fetchUser(id: Int): User {
    val response = httpClient.get("/users/$id")  // точка приостановки
    val avatar = downloadAvatar(response.avatarUrl)  // ещё одна
    return User(response.name, avatar)
}
```

- `suspend` — маркер: эта функция **может** приостановить выполнение
- **Точка приостановки** — место, где корутина может уступить поток
- Вызывать `suspend`-функцию можно только из другой `suspend`-функции или из корутины

---

## Приостановка ≠ блокировка

```
Блокировка (Thread.sleep, future.get):

Поток: ████[занят, но ничего не делает]████  ← поток простаивает


Приостановка (suspend):

Поток: ████░░░░░░░░░░░░░░░░░░░░░████  ← поток свободен!
              ↑                    ↑
        корутина уступила    корутина вернулась
        (поток делает        (может быть
         другую работу)        в другом потоке!)
```

- **Блокировка** — поток занят, но простаивает. Ресурс тратится впустую.
- **Приостановка** — поток освобождается, корутина ждёт в памяти.

---

## CPS-трансформация (Continuation Passing Style)

Компилятор превращает каждую `suspend`-функцию, добавляя скрытый параметр:

```kotlin
// Что мы пишем:
suspend fun fetchUser(id: Int): User

// Что видит JVM после компиляции:
fun fetchUser(id: Int, continuation: Continuation<User>): Any?
//                     ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
//                     скрытый параметр: "что делать дальше"
```

- **Continuation** = "что делать, когда результат будет готов"
- Возвращает `Any?` — либо **результат**, либо специальный **маркер приостановки**
- Это похоже на callback, но компилятор делает всю работу за нас

---

## CPS — идея на пальцах

CPS — стиль программирования, где функция **не возвращает** результат, а **передаёт** его в продолжение:

```kotlin
// Обычный стиль: возвращаем результат
fun add(a: Int, b: Int): Int = a + b
val result = add(1, 2)
println(result)

// CPS: передаём результат в "продолжение"
fun add(a: Int, b: Int, cont: (Int) -> Unit) {
    cont(a + b)  // не return, а вызов continuation
}
add(1, 2) { result -> println(result) }
```

- Kotlin-компилятор автоматически преобразует `suspend`-функции в CPS
- Мы пишем **линейный** код, компилятор делает из него **callback-цепочку**

---

## Continuation — объект продолжения

**Continuation** — это объект, который хранит:

1. **Где мы остановились** — номер состояния (label)
2. **Локальные переменные** — сохраняются между приостановками
3. **Контекст корутины** — диспатчер, Job и т.д.
4. **Как возобновить** — метод `resumeWith(result)`

```
По сути Continuation — это колбэк с памятью:

┌─────────────────────────┐
│      Continuation       │
├─────────────────────────┤
│ label = 2               │  ← где остановились
│ localVar1 = "Alice"     │  ← сохранённые переменные
│ localVar2 = 42          │
│ context = ...           │  ← контекст
│ resumeWith(result) {...}│  ← что делать дальше
└─────────────────────────┘
```

- При приостановке: локальные переменные со стека **переезжают в поля объекта**
- При возобновлении: **восстанавливаются** обратно

---

## Конечный автомат (State Machine)

Компилятор разбивает `suspend`-функцию **по точкам приостановки** на состояния:

```kotlin
// Исходный код:
suspend fun loadData(): String {
    val user = fetchUser()       // точка приостановки ①
    val orders = fetchOrders(user) // точка приостановки ②
    return "$user: $orders"
}
```

```
Компилятор превращает в автомат с 3 состояниями:

┌───────────┐    resume ┌───────────┐    resume ┌───────────┐
│ label = 0 │ ─────────►│ label = 1 │ ─────────►│ label = 2 │
│ fetchUser │           │fetchOrders│           │  return   │
└───────────┘           └───────────┘           └───────────┘
```

- Каждое возобновление — вход в функцию с переходом на нужный label
- Одна `suspend`-функция = один объект continuation = один автомат

---

## Конечный автомат — псевдокод

```kotlin
// Примерно то, во что превращается loadData():
fun loadData(cont: Continuation<String>): Any? {
    // состояние хранится в объекте continuation
    when (cont.label) {
        0 -> {
            cont.label = 1
            val result = fetchUser(cont)  // может приостановиться
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
            // иначе сразу переходим к label 1
        }
        1 -> {
            val user = cont.result       // результат предыдущего шага
            cont.label = 2
            cont.user = user             // сохраняем локальную переменную
            val result = fetchOrders(user, cont)
            if (result == COROUTINE_SUSPENDED) return COROUTINE_SUSPENDED
        }
        2 -> {
            val orders = cont.result
            val user = cont.user         // восстанавливаем
            return "$user: $orders"      // финальный результат
        }
    }
}
```

- **Один вызов функции** — одно состояние автомата
- Локальные переменные становятся **полями** объекта continuation
- `COROUTINE_SUSPENDED` — специальный маркер "корутина приостановилась"

---

## Контекст корутины

Контекст — это ассоциативная структура **"ключ → элемент"**, привязанная к корутине:

```
┌────────────────────────────────────┐
│         CoroutineContext           │
├────────────────────────────────────┤
│ Job → управление жизненным циклом  │
│ Dispatcher → в каком потоке        │
│ CoroutineName → имя для отладки    │
│ ExceptionHandler → обработка ошибок│
└────────────────────────────────────┘
```

- Контекст передаётся через continuation и **наследуется** от родителя
- Можно комбинировать: `Dispatchers.IO + CoroutineName("loader")`
- Это неизменяемая структура — комбинация создаёт новый контекст

---

## Диспатчинг — как корутина попадает в поток

Когда корутина возобновляется, нужно решить: **в каком потоке** выполнять код?

```
Корутина приостановилась       Корутина возобновляется
         │                              │
         └─► Dispatcher решает ────► Поток X
```

- **Dispatcher** — элемент контекста, который **перехватывает** continuation
- При возобновлении диспатчер оборачивает `resumeWith()` и отправляет в нужный поток
- По сути диспатчер = пул потоков + логика отправки

---

## Маркер приостановки

Как рантайм отличает **"функция приостановилась"** от **"функция вернула результат сразу"**?

```kotlin
val result = fetchUser(continuation)

if (result == COROUTINE_SUSPENDED) {
    // Функция реально приостановилась.
    // Освобождаем поток. Когда-нибудь continuation.resumeWith() вернёт нас.
    return COROUTINE_SUSPENDED
} else {
    // Функция выполнилась мгновенно (например, результат был в кэше).
    // Продолжаем без приостановки.
    val user = result as User
}
```

- `COROUTINE_SUSPENDED` — специальный синглтон-маркер
- Если возвращён — корутина действительно приостановлена
- Иначе — можно продолжить синхронно (оптимизация!)

---

## Мост между callback-миром и корутинами

Идея: обернуть любой асинхронный callback в `suspend`-функцию:

```kotlin
// Было: callback-стиль
fun fetchUserAsync(id: Int, callback: (User) -> Unit) {
    // ... асинхронный запрос
}

// Стало: suspend-функция
suspend fun fetchUser(id: Int): User =
    suspendCoroutine { continuation ->   // получаем continuation
        fetchUserAsync(id) { user ->
            continuation.resume(user)    // возобновляем корутину
        }
    }
```

- `suspendCoroutine` — приостанавливает корутину и даёт доступ к continuation
- Когда callback сработает — вызываем `resume()`, корутина продолжается
- Это мост между **любым** асинхронным API и корутинами

---

## Полная картина

```
1. Мы пишем:    suspend fun loadData() { ... }
                              │
2. Компилятор:   CPS-трансформация + конечный автомат
                              │
3. На JVM:       fun loadData(cont: Continuation): Any?
                   с when(label) и полями для локальных переменных
                              │
4. Рантайм:     Dispatcher отправляет continuation в поток
                              │
5. Результат:   Линейный код работает как асинхронный,
                   но без callback-ада и без блокировки потоков
```

---

## Итоги лекции

| Тема | Главная мысль |
|---|---|
| Потоки | Ресурс ОС, тяжёлые, общая память |
| Пулы | Переиспользование потоков, контроль параллелизма |
| Синхронизация | Защита общих данных: synchronized, Lock, Semaphore |
| JMM | Без happens-before нет гарантий видимости |
| Atomic | CAS — безблокировочная альтернатива lock-ам |
| Корутины | CPS + автомат + continuation + диспатчер |

> На практиках будем применять корутины на деле: builders, structured concurrency, Flow.

---
