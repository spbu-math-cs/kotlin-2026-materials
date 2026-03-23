# practice-06: generics, SOLID и паттерны проектирования

## План

1. Generics — параметры типов, type bounds, вариантность
2. Принципы проектирования — SOLID, KISS, DRY
3. Паттерны проектирования
4. Антипаттерны
5. Упражнение: разбор антипаттернов
6. Проектирование с нуля
7. Идеи для устного обсуждения

---

## Generics

### Зачем нужны generics

Без generics приходится использовать `Any` и кастовать:

```kotlin
class BoxAny(val value: Any)

val box = BoxAny(42)
val num: Int = box.value as Int // unsafe cast!
```

С generics — безопасно и выразительно:

```kotlin
class Box<T>(val value: T)

val box = Box(42)        // Box<Int>
val num: Int = box.value // без cast
```

### Type bounds (ограничения типов)

Позволяют ограничить допустимые типы-параметры:

```kotlin
// T должен быть Comparable
fun <T : Comparable<T>> maxOf(a: T, b: T): T {
    return if (a > b) a else b
}

fun main() {
    println(maxOf(3, 7))         // 7
    println(maxOf("abc", "xyz")) // xyz
}
```

Множественные ограничения — через `where`:

```kotlin
fun <T> ensureSorted(list: List<T>) where T : Comparable<T>, T : Any {
    for (i in 0 until list.size - 1) {
        require(list[i] <= list[i + 1]) {
            "List is not sorted at index $i: ${list[i]} > ${list[i + 1]}"
        }
    }
}
```

### Вариантность (variance)

Вопрос: если `Dog` наследует `Animal`, является ли `List<Dog>` подтипом `List<Animal>`?

#### Invariance (инвариантность) — по умолчанию

По умолчанию generic-типы в Kotlin **инвариантны**: `Box<Dog>` — **не** подтип `Box<Animal>`.

```kotlin
class Box<T>(var value: T)

fun main() {
    val dogBox: Box<Dog> = Box(Dog())
    // val animalBox: Box<Animal> = dogBox // CE: Type mismatch
}
```

Это безопасно, потому что `Box` позволяет и **читать**, и **писать**:

```kotlin
// Если бы это было разрешено:
val animalBox: Box<Animal> = dogBox
animalBox.value = Cat() // записали Cat в Box<Dog>!
val dog: Dog = dogBox.value // ClassCastException!
```

#### Covariance (ковариантность) — `out`

Если класс **только отдаёт** значения типа `T` (producer), можно пометить `T` как `out`:

```kotlin
class ReadOnlyBox<out T>(val value: T)

open class Animal
class Dog : Animal()

fun main() {
    val dogBox: ReadOnlyBox<Dog> = ReadOnlyBox(Dog())
    val animalBox: ReadOnlyBox<Animal> = dogBox // OK!
    println(animalBox.value)
}
```

Мнемоника: **`out` = Producer** — тип стоит только в «выходных» позициях (возвращаемые значения).

Примеры из stdlib:
- `List<out E>` — ковариантен, потому что `List` только отдаёт элементы;
- `Sequence<out T>` — аналогично.

#### Contravariance (контравариантность) — `in`

Если класс **только принимает** значения типа `T` (consumer), можно пометить `T` как `in`:

```kotlin
class Printer<in T> {
    fun print(value: T) {
        println(value)
    }
}

fun main() {
    val animalPrinter: Printer<Animal> = Printer()
    val dogPrinter: Printer<Dog> = animalPrinter // OK!
    dogPrinter.print(Dog())
}
```

Мнемоника: **`in` = Consumer** — тип стоит только во «входных» позициях (параметры функций).

Пример из stdlib:
- `Comparable<in T>` — `Comparable<Animal>` может сравнивать и `Dog`, и `Cat`.

#### Сводная таблица

| Модификатор | Название | Отношение подтипов | Позиции `T` | Мнемоника |
|---|---|---|---|---|
| _(нет)_ | Invariant | `Box<Dog>` ≠ `Box<Animal>` | in + out | — |
| `out` | Covariant | `Box<Dog>` ⊆ `Box<Animal>` | только out | Producer |
| `in` | Contravariant | `Box<Animal>` ⊆ `Box<Dog>` | только in | Consumer |

#### Проекции на месте использования (use-site variance)

Иногда нельзя изменить объявление класса. Тогда вариантность задаётся на месте использования:

```kotlin
// Копируем из «только читаемого» источника в «только записываемый» приёмник
fun <T> copy(from: Array<out T>, to: Array<in T>) {
    for (i in from.indices) {
        to[i] = from[i]
    }
}

fun main() {
    val strings: Array<String> = arrayOf("a", "b", "c")
    val anys: Array<Any> = arrayOf("", "", "")
    copy(strings, anys)
    println(anys.toList()) // [a, b, c]
}
```

#### Star projection (`*`)

Когда конкретный тип не важен или неизвестен:

```kotlin
fun printAll(list: List<*>) {
    for (item in list) {
        println(item) // тип элемента — Any?
    }
}
```

`List<*>` эквивалентен `List<out Any?>` — можно **читать**, но нельзя **писать**.

### Упражнения: Generics

1. Напишите generic-функцию `<T : Comparable<T>> List<T>.isSorted(): Boolean`, которая проверяет, отсортирован ли список по возрастанию.

2. Напишите generic-класс `Stack<T>` с методами `push(item: T)`, `pop(): T`, `peek(): T`, `isEmpty(): Boolean`. `pop()` и `peek()` должны бросать `NoSuchElementException` на пустом стеке.

3. Напишите функцию `filterByType<reified R>(list: List<Any>): List<R>`, которая возвращает только элементы заданного типа (используйте `reified`).

4. Ответьте письменно:
    - почему `Array<String>` нельзя присвоить переменной типа `Array<Any>`, а `List<String>` можно присвоить `List<Any>`?
    - в каком случае нужен `where` для generic-функции?

---

## Принципы проектирования

### SOLID

| Принцип | Суть | Нарушение | Решение |
|---|---|---|---|
| **S** — Single Responsibility | У класса одна зона ответственности | `UserManager` создаёт пользователя, шлёт email и пишет в БД | Разделить: `UserService`, `EmailService`, `UserRepository` |
| **O** — Open/Closed | Открыт для расширения, закрыт для изменения | Новый тип скидки требует правки `when` внутри `DiscountService` | `fun interface Discount { fun calculate(order: Order): Double }` — новый тип = новый объект |
| **L** — Liskov Substitution | Подкласс можно использовать вместо родителя без сюрпризов | `Square : Rectangle` — `stretchWidth` ломает инвариант площади | Не наследовать, если подкласс не выполняет контракт родителя |
| **I** — Interface Segregation | Много маленьких интерфейсов лучше одного большого | `Robot : Worker` вынужден реализовывать `eat()` и `sleep()` | Разбить: `Workable`, `Feedable`, `Sleepable` |
| **D** — Dependency Inversion | Зависеть от абстракций, не от реализаций | `OrderService` создаёт `PostgresDatabase()` внутри себя | `OrderService(val repo: OrderRepository)` — в тестах подменяется `InMemoryOrderRepository` |

### KISS и DRY

| Принцип | Расшифровка | Суть | Нарушение | Решение |
|---|---|---|---|---|
| **KISS** | Keep It Simple, Stupid | Простое решение лучше умного | Универсальный `EventProcessor` с 15 параметрами и флагами для каждого кейса | Разделить на несколько простых классов с понятными именами |
| **DRY** | Don't Repeat Yourself | Каждый кусок знания — в одном месте | Одна и та же валидация email продублирована в `UserService`, `AdminService` и `InviteService` | Вынести в `EmailValidator` или extension-функцию `String.isValidEmail()` |

---

## Паттерны проектирования

| Паттерн | Категория | Проблема | Решение |
|---|---|---|---|
| **Strategy** | Поведенческий | Алгоритм меняется в зависимости от контекста, но логика смешана с основным кодом | `fun interface PricingRule`; контекст применяет цепочку через `fold` |
| **Template Method** | Поведенческий | Несколько классов повторяют одну последовательность шагов с разными деталями | Скелет в `abstract fun generate()`, шаги — `abstract fun header()`, `formatRow()` |
| **Visitor** | Поведенческий | Нужно добавить операцию над иерархией классов, не меняя сами классы | Каждый узел реализует `accept(visitor)`; новая операция — новый `ExprVisitor<R>` |
| **Factory Method** | Порождающий | Код создаёт объекты напрямую — сложно добавить новый тип без правки клиента | `NotificationFactory.create("email", target)` возвращает нужную реализацию |
| **Builder** | Порождающий | Конструктор с десятком параметров — легко перепутать порядок, нет валидации при построении | Цепочка методов, объект создаётся в `build()` с проверками |
| **Facade** | Структурный | Клиент вынужден координировать несколько сложных подсистем | `VideoConverterFacade.convert(...)` скрывает `Decoder`, `Encoder`, `Saver` |
| **Proxy** | Структурный | Нужно добавить кеширование, логирование или контроль доступа, не меняя реальный объект | `CachingProxy(real: UserService) : UserService by real` |

---

## Антипаттерны

| Антипаттерн | Суть | Симптомы | Нарушает |
|---|---|---|---|
| **God Object** | Один класс знает и делает всё | Класс на 1000+ строк, десятки методов из разных доменов | SRP, OCP |
| **Shotgun Surgery** | Одно изменение требует правок в десятках мест | Добавление поля — правка 10 файлов | DRY, SRP |
| **Feature Envy** | Метод больше работает с данными чужого класса, чем своего | `OrderService.calculateTotal(cart)` лезет во внутренности `Cart` | SRP, инкапсуляция |
| **Primitive Obsession** | Доменные понятия выражены примитивами вместо типов | `fun createUser(role: String, status: String, type: String)` — легко перепутать порядок | LSP, читаемость |
| **Magic Numbers / Strings** | Числа и строки без имени разбросаны по коду | `if (status == "A" && retries > 3)` | KISS, читаемость |
| **Anemic Model** | Классы-данные без поведения + сервисы со всей логикой | `data class Order(...)` + `OrderService` с 20 методами | SRP, инкапсуляция |
| **Lava Flow** | Мёртвый код, который никто не решается удалить | Закомментированные блоки, `// TODO: remove`, неиспользуемые классы | KISS, DRY |

---

## Упражнение: разбор антипаттернов

Для каждого примера:
1. Определите антипаттерн(ы) и какие принципы нарушены.
2. Зарефакторьте код.
3. Укажите, какие паттерны проектирования помогли.

### Пример 1

```kotlin
class ApplicationManager {
    private val users = mutableMapOf<String, Triple<String, String, Boolean>>()
    private val orders = mutableMapOf<String, List<Pair<String, Double>>>()
    private val discounts = mapOf("VIP" to 0.2, "REGULAR" to 0.0, "EMPLOYEE" to 0.5)

    fun registerUser(id: String, email: String, role: String, active: Boolean) {
        if (!email.contains("@") || !email.contains(".")) {
            throw IllegalArgumentException("Bad email")
        }
        if (role != "VIP" && role != "REGULAR" && role != "EMPLOYEE") {
            throw IllegalArgumentException("Bad role")
        }
        users[id] = Triple(email, role, active)
        println("INSERT INTO users VALUES ('$id', '$email', '$role', $active)")
        println("Sending welcome email to $email")
    }

    fun placeOrder(userId: String, items: List<Pair<String, Double>>) {
        val user = users[userId] ?: throw IllegalStateException("User not found")
        if (!user.third) throw IllegalStateException("User is not active")
        val orderId = "ORD-${System.currentTimeMillis()}"
        orders[orderId] = items
        println("INSERT INTO orders VALUES ('$orderId', '$userId')")
        for (item in items) {
            println("INSERT INTO order_items VALUES ('$orderId', '${item.first}', ${item.second})")
        }
        val total = items.sumOf { it.second }
        val discount = discounts[user.second] ?: 0.0
        val finalPrice = total * (1 - discount)
        println("Sending order confirmation to ${user.first}: total=$finalPrice")
        println("Sending SMS to ${user.first}: your order $orderId is placed")
    }

    fun cancelOrder(orderId: String, userId: String, reason: String) {
        val order = orders[orderId] ?: throw IllegalStateException("Order not found")
        val user = users[userId] ?: throw IllegalStateException("User not found")
        orders.remove(orderId)
        println("UPDATE orders SET status='CANCELLED' WHERE id='$orderId'")
        println("Sending cancellation email to ${user.first}: reason=$reason")
        println("Sending SMS to ${user.first}: order $orderId cancelled")
        val total = order.sumOf { it.second }
        val discount = discounts[user.second] ?: 0.0
        val refund = total * (1 - discount)
        println("Processing refund of $refund to ${user.first}")
    }

    fun generateReport(type: String): String {
        return if (type == "users") {
            "Users: ${users.size}, active: ${users.values.count { it.third }}"
        } else if (type == "orders") {
            "Orders: ${orders.size}, total revenue: ${orders.values.flatten().sumOf { it.second }}"
        } else {
            throw IllegalArgumentException("Unknown report type: $type")
        }
    }
}
```

### Пример 2

```kotlin
class PaymentProcessor {
    fun process(amount: Double, currency: String, method: String,
                cardNumber: String?, cardExpiry: String?, cardCvv: String?,
                walletId: String?, walletProvider: String?,
                bankAccount: String?, bankRouting: String?,
                saveMethod: Boolean, userId: String, orderId: String,
                notifyEmail: Boolean, notifySms: Boolean, retryOnFail: Boolean) {

        if (amount <= 0) throw IllegalArgumentException("Amount must be positive")
        if (currency != "RUB" && currency != "USD" && currency != "EUR") {
            throw IllegalArgumentException("Unsupported currency")
        }

        when (method) {
            "CARD" -> {
                if (cardNumber == null || cardExpiry == null || cardCvv == null)
                    throw IllegalArgumentException("Card details required")
                if (cardNumber.length != 16) throw IllegalArgumentException("Invalid card number")
                if (cardCvv.length != 3) throw IllegalArgumentException("Invalid CVV")
                println("Charging card $cardNumber for $amount $currency")
                if (saveMethod) println("Saving card for user $userId")
            }
            "WALLET" -> {
                if (walletId == null || walletProvider == null)
                    throw IllegalArgumentException("Wallet details required")
                if (walletProvider != "YOOKASSA" && walletProvider != "QIWI")
                    throw IllegalArgumentException("Unsupported wallet")
                println("Charging wallet $walletId via $walletProvider for $amount $currency")
            }
            "BANK" -> {
                if (bankAccount == null || bankRouting == null)
                    throw IllegalArgumentException("Bank details required")
                println("Bank transfer from $bankAccount ($bankRouting) for $amount $currency")
            }
            else -> throw IllegalArgumentException("Unknown payment method: $method")
        }

        println("Recording payment for order $orderId")
        if (notifyEmail) println("Sending payment confirmation email to user $userId")
        if (notifySms) println("Sending payment SMS to user $userId")
        if (retryOnFail) println("Retry policy enabled for order $orderId")
    }
}
```

### Пример 3

```kotlin
class ReportService {
    fun generate(reportType: String, format: String, data: List<Map<String, Any>>): String {
        // Фильтрация
        val filtered = if (reportType == "sales") {
            data.filter { (it["amount"] as Double) > 0 }
        } else if (reportType == "users") {
            data.filter { it["active"] == true }
        } else if (reportType == "inventory") {
            data.filter { (it["stock"] as Int) >= 0 }
        } else {
            throw IllegalArgumentException("Unknown report type")
        }

        // Агрегация
        val summary = if (reportType == "sales") {
            "Total: ${filtered.sumOf { it["amount"] as Double }}"
        } else if (reportType == "users") {
            "Active users: ${filtered.size}"
        } else {
            "Items in stock: ${filtered.sumOf { it["stock"] as Int }}"
        }

        // Форматирование
        return if (format == "CSV") {
            val header = filtered.firstOrNull()?.keys?.joinToString(",") ?: ""
            val rows = filtered.joinToString("\n") { row ->
                row.values.joinToString(",")
            }
            "$header\n$rows\n$summary"
        } else if (format == "JSON") {
            val rows = filtered.joinToString(",\n") { row ->
                "{" + row.entries.joinToString(", ") { "\"${it.key}\": \"${it.value}\"" } + "}"
            }
            "[\n$rows\n]\n// $summary"
        } else if (format == "HTML") {
            val header = filtered.firstOrNull()?.keys?.joinToString("") { "<th>$it</th>" } ?: ""
            val rows = filtered.joinToString("\n") { row ->
                "<tr>" + row.values.joinToString("") { "<td>$it</td>" } + "</tr>"
            }
            "<table><tr>$header</tr>\n$rows</table>\n<!-- $summary -->"
        } else {
            throw IllegalArgumentException("Unknown format")
        }
    }
}
```

### Пример 4

```kotlin
// Файл 1: UserService.kt
class UserService {
    fun getDiscount(userId: String): Double {
        val db = java.sql.DriverManager.getConnection("jdbc:postgresql://localhost/shop")
        val rs = db.prepareStatement("SELECT role FROM users WHERE id = '$userId'").executeQuery()
        return if (rs.next()) {
            when (rs.getString("role")) {
                "VIP"      -> 0.2
                "EMPLOYEE" -> 0.5
                else       -> 0.0
            }
        } else 0.0
    }

    fun getUserEmail(userId: String): String {
        val db = java.sql.DriverManager.getConnection("jdbc:postgresql://localhost/shop")
        val rs = db.prepareStatement("SELECT email FROM users WHERE id = '$userId'").executeQuery()
        return if (rs.next()) rs.getString("email") else ""
    }
}

// Файл 2: OrderService.kt
class OrderService {
    private val userService = UserService()

    fun calculateTotal(userId: String, items: List<Pair<String, Double>>): Double {
        val subtotal = items.sumOf { it.second }
        val discount = userService.getDiscount(userId)
        return subtotal * (1 - discount)
    }

    fun placeOrder(userId: String, items: List<Pair<String, Double>>) {
        val total = calculateTotal(userId, items)
        val db = java.sql.DriverManager.getConnection("jdbc:postgresql://localhost/shop")
        db.prepareStatement(
            "INSERT INTO orders (user_id, total) VALUES ('$userId', $total)"
        ).execute()

        // Дублируем логику получения email из UserService
        val emailDb = java.sql.DriverManager.getConnection("jdbc:postgresql://localhost/shop")
        val rs = emailDb.prepareStatement(
            "SELECT email FROM users WHERE id = '$userId'"
        ).executeQuery()
        val email = if (rs.next()) rs.getString("email") else ""
        println("Sending confirmation to $email: total=$total")
    }
}

// Файл 3: NotificationService.kt
class NotificationService {
    private val userService = UserService()

    fun notifyOrderShipped(userId: String, orderId: String) {
        // Снова дублируем логику получения email
        val email = userService.getUserEmail(userId)
        println("Sending shipping notification to $email: order $orderId shipped")

        // Снова открываем соединение с БД напрямую
        val db = java.sql.DriverManager.getConnection("jdbc:postgresql://localhost/shop")
        db.prepareStatement(
            "INSERT INTO notifications (user_id, order_id, type) VALUES ('$userId', '$orderId', 'SHIPPED')"
        ).execute()
    }

    fun notifyPaymentFailed(userId: String, orderId: String) {
        val email = userService.getUserEmail(userId)
        println("Sending payment failure to $email: order $orderId")

        val db = java.sql.DriverManager.getConnection("jdbc:postgresql://localhost/shop")
        db.prepareStatement(
            "INSERT INTO notifications (user_id, order_id, type) VALUES ('$userId', '$orderId', 'PAYMENT_FAILED')"
        ).execute()
    }
}
```

---

## Проектирование с нуля

### Контекст

Вы разрабатываете бэкенд интернет-магазина. Платёжные провайдеры (Stripe, YooKassa, SBP) присылают **webhook-уведомления** о событиях оплаты. Ваша система должна их принять, обработать и уведомить пользователя.

### Требования

**Приём и парсинг событий**
- Система принимает события от трёх провайдеров: Stripe, YooKassa, SBP
- Каждый провайдер присылает данные в своём формате (разные поля, разные названия)
- Типы событий: `PAYMENT_SUCCESS`, `PAYMENT_FAILED`, `PAYMENT_REFUNDED`
- Если провайдер неизвестен — событие отклоняется с понятной ошибкой

**Обработка событий**
- `PAYMENT_SUCCESS` → статус заказа становится `PAID`
- `PAYMENT_FAILED` → статус заказа становится `FAILED`, фиксируется причина
- `PAYMENT_REFUNDED` → статус заказа становится `REFUNDED`, сумма возвращается на баланс пользователя
- Одно и то же событие не должно обработаться дважды

**Уведомления пользователя**
- При каждом событии пользователь получает уведомление
- Каналы: email, SMS, push
- У каждого пользователя свои настройки: какие каналы включены
- Текст уведомления зависит от типа события
- Если один канал недоступен — остальные всё равно отрабатывают

**Логирование**
- Каждое входящее событие логируется
- Каждая ошибка обработки логируется с деталями

**Инварианты**
- Сумма платежа не может быть отрицательной или нулевой
- Нельзя перевести заказ из `REFUNDED` обратно в `PAID`

---

### Задача

Спроектируйте систему **с нуля**. Напишите код на Kotlin.

**Шаг 1 — доменная модель.** Определите типы данных: что такое событие, заказ, пользователь, настройки уведомлений. Подумайте, какие типы Kotlin лучше всего подходят для каждого понятия.

**Шаг 2 — интерфейсы и зависимости.** Определите границы между компонентами. Что должно быть интерфейсом? Какие зависимости нужны каждому компоненту?

**Шаг 3 — реализация.** Реализуйте компоненты. Начните с самого простого и двигайтесь к сложному.

**Шаг 4 — проверка.** Напишите `main`, который демонстрирует работу системы для всех трёх типов событий от разных провайдеров.

---

### Вопросы для обсуждения в процессе

- Как представить тип события и статус заказа? `String`? `enum`? `sealed interface`? Почему?
- Где должна жить логика парсинга сырых данных провайдера? В каком классе?
- Как добавить четвёртый провайдер, не трогая существующий код?
- Как гарантировать идемпотентность? Где хранить информацию об уже обработанных событиях?
- Как обеспечить, что ошибка в одном канале уведомлений не сломает остальные?
- Как протестировать `PaymentEventProcessor`, не поднимая базу данных и не отправляя реальные письма?
- Где и как проверять инварианты (сумма > 0, недопустимые переходы статусов)?

---

## Идеи для устного обсуждения

- Почему `Array<String>` инвариантен, а `List<String>` ковариантен? Что это означает на практике?
- Что такое `reified` и зачем он нужен? Почему обычный generic-параметр не подходит для `is`-проверок?
- Какой из принципов SOLID нарушается чаще всего? Почему?
- Можно ли нарушить LSP, не нарушая компилятор? Приведите пример.
- Чем KISS и DRY противоречат друг другу? Когда дублирование оправдано?
- Чем Visitor отличается от `when (expr) { is Add -> ... }`? Когда каждый подход лучше?
- Почему Strategy в Kotlin часто реализуется как лямбда, а не как класс?
- В чём разница между Facade и Adapter? Между Proxy и Decorator?
- Как DIP связан с Dependency Injection? Это одно и то же?
- Чем Anemic Model отличается от нормальной доменной модели? Когда она оправдана?
