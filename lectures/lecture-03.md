---

# lecture-03: ООП, классы и объекты

---

# Типы в Kotlin

---

## Тип vs Класс

- **Класс** — это описание структуры и поведения объекта
- **Тип** — это более широкое понятие: описывает множество допустимых значений

```kotlin
class Dog(val name: String)

val dog: Dog = Dog("Rex")   // Dog — и класс, и тип
val dog2: Dog? = null       // Dog? — тип, но не класс
```

- `Dog` и `Dog?` — **разные типы**, но один класс
- `List<Dog>` и `List<Cat>` — тоже разные типы, но один класс (**generics**)
- Тип — это контракт. Класс — это реализация.

> 📚 Подробнее: [Kotlin Spec — Type System](https://kotlinlang.org/spec/type-system.html)

---
layout: two-cols-header
---

# Иерархия типов в Kotlin

- `Any` — корень всех non-nullable типов (аналог `Object` в Java)
- `Unit` — тип функций, не возвращающих значение (аналог `void`)
- `Nothing` — тип, у которого нет значений вообще
- `Any?` — корень **всей** иерархии типов

---

<img src="https://files.speakerdeck.com/presentations/b3c6266ccdbe40cd936d5fc753f2f4dc/slide_19.jpg" width="750" />

---

## Свойства типов в языках программирования

- **Статическая / динамическая** типизация — когда проверяются типы
- **Строгая / слабая** — допускаются ли неявные преобразования
- **Nullable / non-nullable** — может ли значение быть `null`

Kotlin: **статическая + строгая + nullable/non-nullable разделены**

```kotlin
val x: Int = 42
val y: String = x          // CE: Type mismatch
val z: String = x.toString() // OK
```

---

## Примитивы vs Reference-типы на JVM

- В Kotlin **нет примитивов** на уровне языка — всё объекты
- Но компилятор умный: под капотом использует JVM-примитивы

```kotlin
val a: Int = 42      // → int на JVM (примитив)
val b: Int? = 42     // → Integer на JVM (boxing)
```

- Boxing = накладные расходы: лишняя аллокация + GC-давление
- Используйте `Int`, `Long`, `Double` (не `Int?`) там, где `null` не нужен

---

## Any vs Object

- `Any` в Kotlin ≈ `Object` в Java, но не то же самое
- Kotlin компилируется не только в JVM — `Object` может просто не существовать:
  - **Kotlin/JVM** → `Object`
  - **Kotlin/JS** → нет `Object`, корень иерархии не нужен
  - **Kotlin/Native** → нет JVM вообще
- `Any` — универсальная абстракция, не привязанная к платформе
- При вызове Java-кода тип может быть **платформенным**: `T!`

```kotlin
// Java:
// public String getName() { return name; }

val name: String = javaObject.name   // OK, но рискованно
val name2: String? = javaObject.name // безопаснее
```

- `T!` — платформенный тип: Kotlin не знает, nullable он или нет
- Это "дыра" в null-safety при работе с Java

---

## Unit

```kotlin
fun logMessage(msg: String): Unit {
    println("[LOG] $msg")
}

// Unit можно опустить:
fun logMessage2(msg: String) {
    println("[LOG] $msg")
}
```

- `Unit` — это настоящий тип, а не просто `void`
- Функция возвращает объект `Unit` (singleton)
- Полезно в generics: `Result<Unit>`, `() -> Unit`

---

## Nothing

```kotlin
fun fail(message: String): Nothing {
    throw IllegalStateException(message)
}

fun findUser(id: Int): User {
    return userRepository.find(id) ?: fail("User $id not found")
}
```

- `Nothing` — тип, у которого **нет значений**
- Функция с `Nothing` никогда не возвращает управление нормально
- Компилятор использует это для анализа достижимости кода

```kotlin
val x: Nothing? = null  // единственное значение Nothing? — это null
```

---

# ООП

---
layout: two-cols-header
---

# Зачем нужно ООП?

::left::

Без структуры код быстро превращается в это →

Проблемы:
- непонятно, кто за что отвечает
- данные и логика перемешаны
- изменение одного ломает другое
- невозможно тестировать по частям

::right::

<img src="https://i.redd.it/3uti49brbfd81.png" width="350" />

---

## class vs object (первое знакомство)

- `class` — **шаблон** для создания объектов
- `object` — **конкретный** объект (singleton), не шаблон

```kotlin
class Car(val brand: String)  // шаблон

val car1 = Car("Toyota")
val car2 = Car("BMW")         // два разных объекта

object Garage {               // один на всю программу
    val cars = mutableListOf<Car>()
}
```

---
layout: two-cols-header
---

# 4 главных слова

::left::

ООП строится на четырёх концепциях:

1. **Абстракция**
2. **Инкапсуляция**
3. **Наследование**
4. **Полиморфизм**

::right::

<img src="https://i.programmerhumor.io/2024/05/programmerhumor-io-programming-memes-f16e9c7abbea4b5.jpe" width="320" />

---

## Абстракция

> Скрываем детали реализации, показываем только то, что важно пользователю

```kotlin
interface PaymentGateway {
    fun charge(amount: Double, currency: String): PaymentResult
}

// Пользователю не важно, как именно происходит списание:
// через Stripe, через СБП или через крипту
class StripeGateway : PaymentGateway {
    override fun charge(amount: Double, currency: String): PaymentResult {
        // ... HTTP-запрос к Stripe API
    }
}
```

- Работаем с `PaymentGateway`, а не с конкретной реализацией
- Можно подменить реализацию, не меняя остальной код

---

## Инкапсуляция

> Данные и логика, которая с ними работает, живут вместе. Доступ снаружи — только через контролируемый интерфейс.

```kotlin
class BankAccount(initialBalance: Double) {
    private var balance: Double = initialBalance  // скрыто снаружи

    fun deposit(amount: Double) {
        require(amount > 0) { "Amount must be positive" }
        balance += amount
    }

    fun withdraw(amount: Double) {
        require(amount <= balance) { "Insufficient funds" }
        balance -= amount
    }

    fun getBalance(): Double = balance
}
```

- Никто снаружи не может сделать `account.balance = -1000.0`
- Класс **сам защищает свои инварианты**

---

## Модификаторы видимости

| Модификатор | Видимость |
|---|---|
| `public` | везде (по умолчанию) |
| `private` | только внутри класса / файла |
| `protected` | класс + подклассы |
| `internal` | внутри модуля |

```kotlin
class UserService {
    private val cache = mutableMapOf<Int, User>()  // детали реализации

    internal fun findCached(id: Int): User? = cache[id]  // для модуля

    fun getUser(id: Int): User {                          // публичный API
        return findCached(id) ?: loadFromDb(id)
    }

    private fun loadFromDb(id: Int): User { TODO() }
}
```

---

## Наследование

> Подкласс получает всё поведение родителя и может его расширить или изменить

```kotlin
open class Animal(val name: String) {
    open fun speak(): String = "..."
}

class Dog(name: String) : Animal(name) {
    override fun speak(): String = "Woof!"
}

class Cat(name: String) : Animal(name) {
    override fun speak(): String = "Meow!"
}

fun makeNoise(animal: Animal) {
    println("${animal.name} says: ${animal.speak()}")
}
```

- `open` — разрешает наследование / переопределение
- По умолчанию классы в Kotlin **закрыты** для наследования

---

## Полиморфизм

> Один и тот же код работает с объектами разных типов

```kotlin
val animals: List<Animal> = listOf(Dog("Rex"), Cat("Whiskers"), Dog("Buddy"))

for (animal in animals) {
    println(animal.speak())  // вызывается нужная реализация
}
// Woof!
// Meow!
// Woof!
```

- Код написан для `Animal`, но работает с `Dog` и `Cat`
- Конкретный метод выбирается **в runtime** (dynamic dispatch)

---

## Зачем всё это? Сохранение инвариантов

**Инвариант** — условие, которое всегда должно быть истинным

```kotlin
// ❌ Антипаттерн: данные открыты, инварианты не защищены
class Order {
    var items: MutableList<String> = mutableListOf()
    var total: Double = 0.0  // может рассинхронизироваться с items!
}

// ✅ Инварианты защищены
class Order {
    private val _items = mutableListOf<String>()
    val items: List<String> get() = _items

    var total: Double = 0.0
        private set  // изменяется только внутри класса

    fun addItem(item: String, price: Double) {
        _items.add(item)
        total += price  // total всегда соответствует items
    }
}
```

---

## ООП в Lisp

Интересный факт: ООП можно реализовать **без специального синтаксиса**

```lisp
; "Объект" — это просто функция, замыкающая состояние
(defun make-account (balance)
  (lambda (msg amount)
    (cond ((eq msg 'deposit)  (setq balance (+ balance amount)))
          ((eq msg 'withdraw) (setq balance (- balance amount)))
          ((eq msg 'balance)  balance))))

(setq acc (make-account 100))
(funcall acc 'deposit 50)   ; => 150
(funcall acc 'balance 0)    ; => 150
```

- Замыкание = инкапсуляция
- Диспетчеризация по `msg` = полиморфизм
- ООП — это **паттерн мышления**, а не синтаксис

---

<img src="https://i.redd.it/znyie912hyp01.jpg" width="500" />

---

## Принципы SOLID

| Принцип | Суть |
|---|---|
| **S** — Single Responsibility | У класса есть только одна зона ответственности |
| **O** — Open/Closed | Открыт для расширения, закрыт для изменения |
| **L** — Liskov Substitution | Подкласс можно использовать вместо родителя |
| **I** — Interface Segregation | Много маленьких интерфейсов лучше одного большого |
| **D** — Dependency Inversion | Зависеть от абстракций, не от реализаций |

---

## S — Single Responsibility

> У класса есть только одна зона ответственности

```kotlin
// ❌ Класс делает слишком много
class UserManager {
    fun createUser(name: String, email: String): User { TODO() }
    fun sendWelcomeEmail(user: User) { TODO() }  // это не его дело
    fun saveToDatabase(user: User) { TODO() }     // и это тоже
}

// ✅ Каждый класс — своя ответственность
class UserService {
    fun createUser(name: String, email: String): User { TODO() }
}
class EmailService {
    fun sendWelcomeEmail(user: User) { TODO() }
}
class UserRepository {
    fun save(user: User) { TODO() }
}
```

---

## O — Open/Closed

> Открыт для расширения, закрыт для изменения

```kotlin
// ❌ Каждый новый тип скидки — правка существующего класса
class DiscountService {
    fun calculate(order: Order, type: String): Double = when (type) {
        "VIP"      -> order.total * 0.2
        "SEASONAL" -> order.total * 0.1
        else       -> 0.0
        // нужна новая скидка? лезем сюда и правим
    }
}

// ✅ Новый тип скидки — новый класс, старый код не трогаем
fun interface Discount {
    fun calculate(order: Order): Double
}

val vipDiscount      = Discount { it.total * 0.2 }
val seasonalDiscount = Discount { it.total * 0.1 }
```

---

## L — Liskov Substitution

> Подкласс можно использовать везде, где ожидается родитель, без сюрпризов

```kotlin
open class Rectangle(open var width: Double, open var height: Double) {
    open fun area() = width * height
}

// ❌ Нарушение: Square меняет контракт Rectangle
class Square(side: Double) : Rectangle(side, side) {
    override var width  get() = super.width
        set(value) { super.width = value; super.height = value }
    override var height get() = super.height
        set(value) { super.width = value; super.height = value }
}

fun stretchWidth(r: Rectangle) {
    r.width = r.width * 2
    // ожидаем: area удвоилась. Для Square — area учетверилась!
}
```

- Если подкласс нарушает ожидания родителя — иерархия спроектирована неверно

---

## I — Interface Segregation

> Много маленьких интерфейсов лучше одного большого

```kotlin
// ❌ Один толстый интерфейс — классы вынуждены реализовывать лишнее
interface Worker {
    fun work()
    fun eat()
    fun sleep()
}

class Robot : Worker {
    override fun work() { println("working") }
    override fun eat()  { TODO("Robots don't eat") }  // бессмыслица
    override fun sleep(){ TODO("Robots don't sleep") }
}

// ✅ Разделяем по смыслу
fun interface Workable  { fun work() }
fun interface Feedable  { fun eat() }
fun interface Sleepable { fun sleep() }

class Robot  : Workable { override fun work() = println("working") }
class Human  : Workable, Feedable, Sleepable {
    override fun work()  = println("working")
    override fun eat()   = println("eating")
    override fun sleep() = println("sleeping")
}
```

---

## D — Dependency Inversion

> Зависеть от абстракций, не от реализаций

```kotlin
// ❌ Зависимость от конкретной реализации
class OrderService {
    private val db = PostgresDatabase()  // жёстко привязан к Postgres
    fun placeOrder(order: Order) = db.save(order)
}

// ✅ Зависимость от абстракции
interface OrderRepository {
    fun save(order: Order)
}

class OrderService(private val repository: OrderRepository) {
    fun placeOrder(order: Order) = repository.save(order)
}
```

- В продакшене: `PostgresOrderRepository`
- В тестах: `InMemoryOrderRepository`
- `OrderService` не знает и не должен знать разницы

---

## SOLID — не панацея

Код может следовать всем принципам SOLID и при этом быть **нечитаемым**:

```kotlin
// Всё по SOLID: SRP ✅ OCP ✅ LSP ✅ ISP ✅ DIP ✅
class UserRegistrationCommandHandlerFactoryProviderImpl(
    private val userCreationServiceDelegate: UserCreationServiceDelegate,
    private val emailNotificationDispatcherAdapter: EmailNotificationDispatcherAdapter,
    private val userPersistenceRepositoryGateway: UserPersistenceRepositoryGateway,
) : UserRegistrationCommandHandlerFactoryProvider {
    override fun handle(cmd: RegisterUserCommand) {
        userCreationServiceDelegate.delegate(cmd.toDto())
    }
}
```

- SOLID — это **инструмент**, а не цель
- Здравый смысл и читаемость важнее формального следования принципам
- Чрезмерное дробление = **over-engineering**

---
layout: two-cols-header
---

## DDD: Domain-Driven Design

::left::

Подход к проектированию, где **бизнес-логика** — центр системы

**Единый язык (Ubiquitous Language)** — разработчики и бизнес говорят на одном языке.
Названия в коде = названия в техзаданиях = названия в разговоре с заказчиком

**Bounded Context** — граница, внутри которой модель имеет чёткий смысл.
Одно слово может значить разное в разных контекстах: `User` в аутентификации ≠ `User` в биллинге

::right::

<img src="https://martinfowler.com/bliki/images/boundedContext/sketch.png" width="380" />

---

## DDD: ключевые понятия

- **Entity** — объект с идентичностью (`User`, `Order`)
- **Value Object** — объект без идентичности, определяется значениями (`Money`, `Address`)
- **Repository** — абстракция доступа к данным
- **Service** — логика, не принадлежащая конкретной сущности
- **Aggregate** — группа объектов с единым корнем

```kotlin
// Value Object: два Money с одинаковыми полями — одинаковы
data class Money(val amount: Double, val currency: String)

// Entity: два User с одинаковым именем — разные объекты
class User(val id: UUID, var name: String)
```

---
layout: two-cols-header
---

# Как это выглядит в продакшене

::left::

В реальных проектах:
- Сотни классов, интерфейсов, модулей
- Слои: controller → service → repository
- Зависимости между слоями
- Без архитектуры — хаос

Хорошая архитектура = **читаемость + тестируемость + изменяемость**

::right::

![Реальный граф зависимостей в большом проекте](https://gateway240.com/blog/what-is-a-dependency-graph/opensim-code-deps.png)

---

# Классы

---

## Синтаксис класса

```kotlin
class Person(val name: String, var age: Int) {
    fun greet(): String = "Hi, I'm $name"
}

val person = Person("Alice", 30)
println(person.greet())  // Hi, I'm Alice
println(person.name)     // Alice
person.age = 31          // OK: var
// person.name = "Bob"   // CE: val
```

- `class` — ключевое слово
- Параметры в скобках — **primary constructor**
- `val`/`var` в конструкторе автоматически создают свойства

---

## Свойства: getters и setters

```kotlin
class Temperature(celsius: Double) {
    var celsius: Double = celsius
        set(value) {
            require(value >= -273.15) { "Below absolute zero!" }
            field = value  // field — backing field
        }

    val fahrenheit: Double
        get() = celsius * 9 / 5 + 32  // вычисляется каждый раз
}

val t = Temperature(100.0)
println(t.fahrenheit)  // 212.0
t.celsius = 232.78     // 451°F — температура воспламенения бумаги 📚
println(t.fahrenheit)  // 451.0
t.celsius = -300.0     // IllegalArgumentException
```

- `field` — ссылка на backing field внутри getter/setter
- Computed property (`fahrenheit`) — нет backing field

---

## Свойства: декомпилированный Java-код

Kotlin:
```kotlin
class User(val name: String, var age: Int)
```

Эквивалентный Java-код:
```java
public final class User {
    private final String name;
    private int age;

    public User(String name, int age) {
        this.name = name;
        this.age = age;
    }

    public String getName() { return name; }
    public int getAge() { return age; }
    public void setAge(int age) { this.age = age; }
}
```

- Kotlin генерирует getters/setters автоматически
- `val` → только getter, `var` → getter + setter

---

## Конструкторы: primary constructor

```kotlin
class User(
    val id: Int,
    val name: String,
    val email: String = "",  // default-значение
)
```

- Primary constructor — прямо в заголовке класса
- Параметры с `val`/`var` становятся свойствами
- Поддерживает default-значения и named arguments

```kotlin
val u1 = User(1, "Alice", "alice@example.com")
val u2 = User(2, "Bob")               // email = ""
val u3 = User(id = 3, name = "Carol") // named arguments
```

---

## Конструкторы: параметр без `val`/`var`

Если не написать `val`/`var` — параметр **не становится свойством**, он доступен только внутри `init` и других инициализаторов:

```kotlin
class User(val name: String, age: Int) {  // age — просто параметр
    val isAdult: Boolean = age >= 18      // OK: используем в init-контексте

    fun getAge(): Int = age               // CE: Unresolved reference: age
}
```

Это полезно, когда параметр нужен только для вычисления свойств и хранить его не нужно.

---

## Конструкторы: init-блок

```kotlin
class User(val name: String, val age: Int) {
    init {
        require(name.isNotBlank()) { "Name cannot be blank" }
        require(age >= 0) { "Age cannot be negative" }
    }
}
```

- `init` выполняется сразу после primary constructor
- Используйте для **валидации** входных данных
- Можно иметь несколько `init`-блоков (выполняются по порядку)

> ⚠️ Антипаттерн: сложная логика в `init` — лучше вынести в фабричный метод

---

## Конструкторы: secondary constructors

```kotlin
class Connection(val host: String, val port: Int) {
    constructor(host: String) : this(host, 443)  // вызывает primary
    constructor() : this("localhost")             // вызывает secondary выше
}
```

> ⚠️ Антипаттерн: secondary constructors — почти всегда лучше использовать **default-значения**

```kotlin
// ✅ Лучше так:
class Connection(
    val host: String = "localhost",
    val port: Int = 443,
)
```

Secondary constructors оправданы только при наследовании от Java-классов.

---

## lateinit var

```kotlin
class UserController {
    lateinit var userService: UserService  // инициализируется позже (DI)

    fun getUser(id: Int): User {
        if (!::userService.isInitialized) error("Service not injected")
        return userService.findById(id)
    }
}
```

- `lateinit` — обещание: "я инициализирую это до первого использования"
- Только для `var`, только для non-nullable reference-типов
- При обращении до инициализации — `UninitializedPropertyAccessException`

> ⚠️ Антипаттерн: `lateinit` вместо нормального конструктора. Используйте только для DI-фреймворков (Spring, Koin).

---

## data class

```kotlin
data class Point(val x: Double, val y: Double)

val p1 = Point(1.0, 2.0)
val p2 = Point(1.0, 2.0)
val p3 = p1.copy(y = 5.0)

println(p1 == p2)   // true  (структурное равенство)
println(p1)         // Point(x=1.0, y=2.0)
println(p3)         // Point(x=1.0, y=5.0)
```

Компилятор автоматически генерирует:
- `equals()` / `hashCode()` — по всем свойствам из конструктора
- `toString()` — читаемое представление
- `copy()` — создание копии с изменёнными полями
- `componentN()` — для деструктуризации

---

<img src="https://miro.medium.com/1*Vn1UGKL1KOQP_xPSe5JgdA.png" width="450" />

---

## data class: декомпилированный Java-код

Kotlin:
```kotlin
data class Point(val x: Double, val y: Double)
```

Эквивалентный Java-код (упрощённо):
```java
public final class Point {
    private final double x, y;
    public Point(double x, double y) { this.x = x; this.y = y; }
    public double getX() { return x; }
    public double getY() { return y; }

    @Override public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Point)) return false;
        Point p = (Point) o;
        return Double.compare(x, p.x) == 0 && Double.compare(y, p.y) == 0;
    }
    @Override public int hashCode() { return Objects.hash(x, y); }
    @Override public String toString() { return "Point(x=" + x + ", y=" + y + ")"; }
    public Point copy(double x, double y) { return new Point(x, y); }
}
```

---

## data class: Pair и Triple

Встроенные data classes в стандартной библиотеке:

```kotlin
val pair = Pair("key", 42)
val (key, value) = pair  // деструктуризация

// Синтаксический сахар:
val pair2 = "key" to 42

val triple = Triple(1, "hello", true)
val (a, b, c) = triple
```

> ⚠️ Антипаттерн: использовать `Pair`/`Triple` вместо именованного data class

```kotlin
// ❌ Непонятно, что есть что
fun getCoords(): Pair<Double, Double> = Pair(55.75, 37.61)

// ✅ Читаемо и самодокументируемо
data class Coordinates(val lat: Double, val lon: Double)
fun getCoords(): Coordinates = Coordinates(55.75, 37.61)
```

---

## data class: антипаттерн с методами

```kotlin
// ⚠️ Антипаттерн: бизнес-логика в data class
data class Order(val items: List<String>, val total: Double) {
    fun applyDiscount(percent: Double): Order { TODO() }
    fun sendConfirmationEmail() { TODO() }  // точно не сюда
}

// ✅ data class — только данные, логика — в сервисе
data class Order(val items: List<String>, val total: Double)

class OrderService {
    fun applyDiscount(order: Order, percent: Double): Order {
        return order.copy(total = order.total * (1 - percent / 100))
    }
}
```

---

# Наследование

---

## Базовый синтаксис

```kotlin
open class Shape(val color: String) {
    open fun area(): Double = 0.0
    open fun describe(): String = "Shape($color)"
}

class Circle(color: String, val radius: Double) : Shape(color) {
    override fun area(): Double = Math.PI * radius * radius
    override fun describe(): String = "Circle(r=$radius, ${super.describe()})"
}

class Rectangle(color: String, val w: Double, val h: Double) : Shape(color) {
    override fun area(): Double = w * h
}
```

- `open` — разрешает наследование / переопределение
- `override` — обязателен (в отличие от Java)
- `super` — доступ к родительской реализации

---

<img src="https://miro.medium.com/0*F0lVBkMufX-MxRYf.jpg" width="500" />

---

## open — антипаттерн

> По умолчанию классы в Kotlin **final** — это правильно

```kotlin
// ⚠️ Антипаттерн: открывать всё подряд
open class UserService {
    open fun findUser(id: Int): User { TODO() }
}

// Проблема: наследник может сломать инварианты родителя
class HackedUserService : UserService() {
    override fun findUser(id: Int): User = User(id, "hacker")
}
```

**Правило:** используйте `open` осознанно. Предпочитайте **композицию** наследованию.

---

## interface vs abstract class

```kotlin
interface Drawable {
    fun draw()                          // абстрактный метод
    fun hide() { println("hidden") }   // метод с реализацией по умолчанию
    // val state = "active"            // ❌ нельзя хранить состояние
}

abstract class Widget(val id: String) {
    protected var visible: Boolean = true  // состояние — можно
    abstract fun render()
    fun show() { visible = true }
}
```

| | `interface` | `abstract class` |
|---|---|---|
| Состояние | ❌ | ✅ |
| `protected` | ❌ | ✅ |
| Множественное наследование | ✅ | ❌ |

**Правило:** если нужно состояние — `abstract class`. Иначе — `interface`.

---

## Делегирование: by

```kotlin
interface Logger {
    fun log(message: String)
}

class ConsoleLogger : Logger {
    override fun log(message: String) = println("[LOG] $message")
}

// Делегируем реализацию Logger объекту consoleLogger
class UserService(private val logger: Logger) : Logger by logger {
    fun createUser(name: String) {
        log("Creating user: $name")  // вызывается logger.log()
        // ...
    }
}
```

- `by` — делегирование реализации интерфейса другому объекту
- Альтернатива наследованию: **"предпочитайте композицию"**
- Нет boilerplate с ручным делегированием каждого метода

---

## SAM (Single Abstract Method)

```kotlin
// Java-интерфейс с одним методом
// interface Runnable { void run(); }

val runnable = Runnable { println("Running!") }  // SAM-конверсия

// Kotlin-интерфейс: нужен fun interface
fun interface Validator<T> {
  fun validate(value: T): Boolean
}

val emailValidator = Validator<String> { it.contains("@") }
val emailValidatorLong = object : Validator<String> {
  override fun validate(value: String): Boolean {
    it.contains("@")
  }
}

println(emailValidator.validate("user@example.com"))  // true
```

- SAM-конверсия: лямбда вместо анонимного класса
- Для Kotlin-интерфейсов нужно явно указать `fun interface`

---

## sealed class и sealed interface

```kotlin
sealed class Result<out T> {
    data class Success<T>(val data: T) : Result<T>()
    data class Error(val message: String, val cause: Throwable? = null) : Result<Nothing>()
    object Loading : Result<Nothing>()
}

fun handleResult(result: Result<User>) {
    when (result) {
        is Result.Success -> println("Got user: ${result.data.name}")
        is Result.Error   -> println("Error: ${result.message}")
        Result.Loading    -> println("Loading...")
        // else не нужен — компилятор знает все варианты
    }
}
```

- `sealed` — все подклассы известны на этапе компиляции
- `when` без `else` — компилятор проверяет полноту
- Идеально для моделирования состояний и результатов операций

---

# enum

---

## Базовый синтаксис

```kotlin
enum class Direction {
    NORTH, SOUTH, EAST, WEST
}

val dir = Direction.NORTH
println(dir.name)    // "NORTH"
println(dir.ordinal) // 0

when (dir) {
    Direction.NORTH -> println("Going up")
    Direction.SOUTH -> println("Going down")
    Direction.EAST  -> println("Going right")
    Direction.WEST  -> println("Going left")
}
```

- `enum class` — фиксированный набор значений
- Каждое значение — singleton-объект
- `name` и `ordinal` — встроенные свойства

---

## enum с конструктором и интерфейсом

```kotlin
enum class HttpStatus(val code: Int, val description: String) {
    OK(200, "OK"),
    NOT_FOUND(404, "Not Found"),
    INTERNAL_ERROR(500, "Internal Server Error");

    val isSuccess: Boolean get() = code in 200..299
}

println(HttpStatus.OK.isSuccess)          // true
println(HttpStatus.NOT_FOUND.description) // "Not Found"

// enum может реализовывать интерфейс
interface Describable { fun describe(): String }

enum class Planet(val mass: Double) : Describable {
    EARTH(5.97e24),
    MARS(6.39e23);

    override fun describe() = "$name: mass=$mass kg"
}
```

---

# object

---
layout: two-cols-header
---

## Singleton: object

::left::

```kotlin
object AppConfig {
    val apiUrl: String = "https://api.example.com"
    val timeout: Int = 30

    fun buildUrl(path: String) = "$apiUrl/$path"
}

println(AppConfig.apiUrl)
println(AppConfig.buildUrl("users"))
```

- `object` — singleton, создаётся при первом обращении
- Thread-safe: JVM гарантирует атомарность инициализации класса
- Не нужен double-checked locking как в Java

::right::

```java
// Java: нужно писать вручную
public class AppConfig {
    private static volatile AppConfig instance;
    private AppConfig() {}
    public static AppConfig getInstance() {
        if (instance == null) {
            synchronized (AppConfig.class) {
                if (instance == null) instance = new AppConfig();
            }
        }
        return instance;
    }
}
```

---

## data object

```kotlin
sealed class Command {
    data object Quit : Command()
    data class Move(val x: Int, val y: Int) : Command()
    data class Print(val message: String) : Command()
}

val cmd: Command = Command.Quit
println(cmd)  // "Quit" (а не "Command$Quit@1a2b3c")
```

- `data object` — как `object`, но с нормальным `toString()` и `equals()`
- Полезно в `sealed`-иерархиях рядом с `data class`

---

## companion object

```kotlin
class User private constructor(val id: Int, val name: String) {

    companion object {
        private var nextId = 1

        fun create(name: String): User {
            return User(nextId++, name)
        }

        fun fromJson(json: String): User {
            // парсинг JSON
            TODO()
        }
    }
}

val user = User.create("Alice")   // фабричный метод
// val user2 = User(1, "Bob")    // CE: конструктор private
```

- `companion object` — объект, привязанный к классу (аналог `static` в Java)
- Используется для фабричных методов, констант класса

---

# Вложенные классы

---

## nested vs inner

```kotlin
class Outer(val value: Int) {

    // nested: НЕ имеет доступа к Outer
    class Nested {
        fun hello() = "I'm nested"
        // fun bad() = value  // CE: нет доступа к Outer
    }

    // inner: имеет доступ к Outer через this@Outer
    inner class Inner {
        fun hello() = "Outer value is ${this@Outer.value}"
    }
}

val nested = Outer.Nested()   // без экземпляра Outer
val inner = Outer(42).Inner() // нужен экземпляр Outer
println(inner.hello())        // "Outer value is 42"
```

- По умолчанию вложенный класс — `nested` (нет ссылки на внешний)
- `inner` — хранит ссылку на внешний объект (осторожно с утечками памяти!)

---

# Области видимости

---

## Пакеты и организация кода

```kotlin
package com.example.users

class User(val name: String)
class UserService
interface UserRepository

// Kotlin не требует одного класса на файл!
// Можно группировать связанные классы в одном файле
```

- Kotlin не навязывает правило "один класс = один файл"
- Группируйте **связанные** классы вместе
- Но следите за длиной файла (> 300–400 строк — повод разбить)

---

## Модификаторы видимости: итог

```kotlin
// public (по умолчанию): видно везде
class PublicClass

// internal: видно только внутри модуля
internal class InternalHelper

// private (на уровне файла): видно только в этом файле
private fun helperFunction() { }

class MyClass {
    public val publicProp = 1       // везде
    internal val internalProp = 2   // в модуле
    protected val protectedProp = 3 // класс + подклассы
    private val privateProp = 4     // только здесь
}
```

---

## const val: глобальные константы

```kotlin
// Файл: Constants.kt
package com.example

const val MAX_RETRY_COUNT = 3
const val DEFAULT_TIMEOUT_MS = 5000L
const val API_BASE_URL = "https://api.example.com"
```

- `const val` — compile-time константа (вычисляется при компиляции)
- Только примитивные типы + `String`
- Используйте `UPPER_SNAKE_CASE`
- Вычисляется в compile-time → встраивается в байткод напрямую

---

# Примеры плохо спроектированного кода

---

<img src="https://i.imgflip.com/1ln7vm.jpg" width="400" />

---

## God Object

```kotlin
// ❌ Антипаттерн: класс знает и делает всё
class ApplicationManager {
    fun createUser(name: String) { TODO() }
    fun sendEmail(to: String, body: String) { TODO() }
    fun saveToDatabase(obj: Any) { TODO() }
    fun parseJson(json: String): Any { TODO() }
    fun generateReport(): String { TODO() }
    fun handleHttpRequest(path: String): String { TODO() }
    fun validateInput(input: String): Boolean { TODO() }
    // ... ещё 50 методов
}
```

**Проблема:** любое изменение в системе затрагивает этот класс. Невозможно тестировать по частям.

---

## Анемичная модель

```kotlin
// ❌ Антипаттерн: данные отдельно, логика отдельно
data class Order(var status: String, var total: Double, var items: MutableList<String>)

class OrderUtils {
    fun addItem(order: Order, item: String, price: Double) {
        order.items.add(item)
        order.total += price
    }
    fun cancel(order: Order) { order.status = "CANCELLED" }
}

// ✅ Логика принадлежит данным
class Order(initialItems: List<String> = emptyList()) {
    private val _items = initialItems.toMutableList()
    val items: List<String> get() = _items
    var status: String = "PENDING"
        private set
    var total: Double = 0.0
        private set

    fun addItem(item: String, price: Double) { _items.add(item); total += price }
    fun cancel() { status = "CANCELLED" }
}
```

---

## Нарушение инкапсуляции через открытые коллекции

```kotlin
// ❌ Антипаттерн: внутреннее состояние утекает наружу
class Classroom {
    val students: MutableList<String> = mutableListOf()
}

val classroom = Classroom()
classroom.students.add("Alice")   // кто угодно меняет список снаружи

// ✅ Класс контролирует доступ к списку
class Classroom {
    private val _students = mutableListOf<String>()
    val students: List<String> get() = _students  // рид-онли вид наружу

    fun enroll(name: String) {
        require(name.isNotBlank()) { "Student name cannot be blank" }
        check(name !in _students) { "Student already enrolled" }
        _students.add(name)
    }
}
```

---

## Наследование ради повторного использования кода

```kotlin
// ❌ Антипаттерн: наследование для повторного использования кода
open class JsonParser {
    fun parse(json: String): Map<String, Any> { TODO() }
}

class UserParser : JsonParser() {  // наследуем, чтобы использовать parse()
    fun parseUser(json: String): User {
        val map = parse(json)
        return User(map["name"] as String)
    }
}

// ✅ Композиция: возьми парсер как зависимость
class UserParser(private val jsonParser: JsonParser) {
    fun parseUser(json: String): User {
        val map = jsonParser.parse(json)
        return User(map["name"] as String)
    }
}
```

- `UserParser` не является `JsonParser` по смыслу — он его **использует**
- Правило: наследуй только тогда, когда объект **является** родителем («собака является животным», а не «собака использует животное»)

---

## Магические числа и строки

```kotlin
// ❌ что значит 86400000? И почему "ADMIN"?
fun isExpired(createdAt: Long) =
    System.currentTimeMillis() - createdAt > 86400000
fun canEdit(user: User) = user.role == "ADMIN"

// ✅ названные константы и enum
enum class Role { ADMIN, VIEWER, EDITOR }
const val SESSION_TTL_MS = 24 * 60 * 60 * 1000L  // 24 часа

fun isExpired(createdAt: Long) =
    System.currentTimeMillis() - createdAt > SESSION_TTL_MS
fun canEdit(user: User) = user.role == Role.ADMIN
```

---

## Утечка `this` в `init`

```kotlin
abstract class Base {
    init {
        println("Base init: value = ${getValue()}")
    }
    abstract fun getValue(): Int
}

class Derived : Base() {
    val value: Int = 42  // ещё не инициализировано, когда вызывается init родителя!
    override fun getValue(): Int = value
}

Derived()  // выведет: "Base init: value = 0" — а не 42!
```

- Порядок инициализации: сначала `init` родителя, потом свойства подкласса
- Вызов `open`/`abstract` метода из `init` — почти всегда баг
- Kotlin даже предупреждает об этом инспекцией ⚠️

---

## Хрупкий базовый класс (Fragile Base Class)

```kotlin
open class Collection {
    private var count = 0

    open fun add(element: Int) {
        count++  // считаем элементы
    }

    open fun addAll(elements: List<Int>) {
        elements.forEach { add(it) }  // вызываем add()
    }
}

class CountingCollection : Collection() {
    var addedCount = 0

    override fun add(element: Int) {
        addedCount++
        super.add(element)
    }
}

val c = CountingCollection()
c.addAll(listOf(1, 2, 3))
println(c.addedCount)  // 3 — ок?
// Разработчик Base решил оптимизировать addAll() и убрал вызов add() → addedCount = 0!
```

- Изменение внутренней реализации родителя незаметно ломает подкласс
- Поэтому классы в Kotlin `final` по умолчанию — это защита от этой проблемы

---

## Конструктор родителя с side effects

```kotlin
object Registry {
    val instances = mutableListOf<Service>()
}

abstract class Service(val name: String) {
    init {
        Registry.instances.add(this)  // регистрируемся в глобальный реестр
        println("Registered: $name")
    }
}

class PaymentService : Service("payment")
class EmailService : Service("email")

// Проблемы:
// 1. Подкласс не знает, что происходит side effect при создании
// 2. Тесты загрязняют глобальный Registry и влияют друг на друга
// 3. Невозможно создать объект без регистрации
```

- `init` должен только валидировать данные, но не взаимодействовать с внешним миром
- Регистрацию лучше делегировать в DI-фреймворк (Spring, Koin)

---

## Итог лекции

- **Типы** в Kotlin: `Any`, `Unit`, `Nothing`, nullable/non-nullable, платформенные типы
- **ООП**: абстракция, инкапсуляция, наследование, полиморфизм — не просто слова, а инструменты
- **Классы**: свойства, конструкторы, `data class` — Kotlin генерирует много кода за вас
- **Наследование**: `open`/`override`, `interface` vs `abstract class`, `sealed`, делегирование
- **enum** и **object**: для фиксированных наборов и singleton'ов
- **Архитектура**: SOLID и DDD — как думать о коде в продакшене

> Хороший код — это код, который легко **читать**, **тестировать** и **изменять**.
