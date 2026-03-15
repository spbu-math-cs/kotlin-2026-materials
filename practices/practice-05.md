# practice-05: исключения, продвинутые конструкции, generics, аннотации

## План

1. Исключения — `try`, `catch`, `finally`, иерархия, отличия от Java
2. Продвинутые конструкции языка — extension-функции, infix-функции, перегрузка операторов, delegate properties
3. Generics — параметры типов, type bounds, invariance, covariance, contravariance
4. Аннотации — стандартные, создание своих, use-site targets
5. Упражнения
6. Что важно вынести из практики
7. Идеи для устного обсуждения

---

## Исключения

### Иерархия исключений

В Kotlin (как и в Java) все исключения — наследники `Throwable`:

```
Throwable
├── Error              — критические ошибки JVM (OutOfMemoryError, StackOverflowError)
└── Exception
    ├── RuntimeException  — unchecked (IllegalArgumentException, NullPointerException, ...)
    └── IOException, ...  — checked (только в Java!)
```

Ключевое отличие от Java: **в Kotlin нет checked exceptions**.

Это значит:
- не нужно писать `throws` в сигнатуре;
- не нужен пустой `catch`, чтобы компилятор «замолчал»;
- вся ответственность за обработку — на разработчике.

### try / catch / finally

`try` — это **выражение** (expression), а не только statement. Оно возвращает значение.

```kotlin
val number: Int = try {
    "abc".toInt()
} catch (e: NumberFormatException) {
    -1
}
println(number) // -1
```

Полный синтаксис:

```kotlin
try {
    // опасный код
} catch (e: SomeException) {
    // обработка
} catch (e: AnotherException) {
    // обработка другого типа
} finally {
    // выполняется ВСЕГДА (и при нормальном завершении, и при исключении)
}
```

#### Правила

- `catch`-блоки проверяются **сверху вниз** — первый подходящий обрабатывает исключение.
- Более специфичный тип нужно ставить **выше** более общего.
- `finally` выполняется всегда — даже если в `catch` сделали `return`.

Пример с `finally`:

```kotlin
fun readConfig(): String {
    val reader = java.io.FileReader("config.txt")
    return try {
        reader.readText()
    } catch (e: java.io.FileNotFoundException) {
        "default"
    } finally {
        reader.close() // закрываем ресурс в любом случае
    }
}
```

### Функции-помощники из stdlib

Kotlin предоставляет несколько удобных функций для работы с проверками и ошибками:

```kotlin
// require — проверяет аргументы, бросает IllegalArgumentException
fun setAge(age: Int) {
    require(age >= 0) { "Age must be non-negative, got $age" }
}

// check — проверяет состояние, бросает IllegalStateException
fun start() {
    check(!isRunning) { "Already running" }
}

// error — бросает IllegalStateException с сообщением
fun unreachable(): Nothing {
    error("This should never happen")
}
```

### runCatching и Result

Вместо `try/catch` можно использовать функциональный стиль:

```kotlin
val result: Result<Int> = runCatching {
    "abc".toInt()
}

println(result.isSuccess)  // false
println(result.isFailure)  // true

val value = result.getOrDefault(-1)
println(value) // -1
```

`Result<T>` можно комбинировать:

```kotlin
val output = runCatching { riskyOperation() }
    .map { it * 2 }
    .getOrElse { e ->
        println("Error: ${e.message}")
        0
    }
```

Хороший вопрос студентам:

> Почему Kotlin отказался от checked exceptions? Какие плюсы и минусы это даёт?

---

## Продвинутые конструкции языка

### Extension-функции

Extension-функции позволяют «добавить» метод к существующему классу **без наследования и модификации исходного кода**.

```kotlin
fun String.isPalindrome(): Boolean {
    val cleaned = this.lowercase().filter { it.isLetterOrDigit() }
    return cleaned == cleaned.reversed()
}

fun main() {
    println("racecar".isPalindrome())     // true
    println("hello".isPalindrome())       // false
    println("A man a plan a canal Panama".isPalindrome()) // true
}
```

#### Как это работает под капотом

Extension-функции — это **обычные статические функции** с дополнительным первым параметром (receiver). Компилятор превращает:

```kotlin
fun String.isPalindrome(): Boolean = ...
```

в байткоде примерно в:

```java
public static boolean isPalindrome(String $this$isPalindrome) { ... }
```

Из этого следует:
- **нет доступа к `private`/`protected` членам** класса;
- **нет полиморфизма** — расширение привязано к **статическому** типу, а не к типу в рантайме;
- **нет переопределения** — нельзя «перекрыть» метод класса.

Пример отсутствия полиморфизма:

```kotlin
open class Shape
class Circle : Shape()

fun Shape.name() = "Shape"
fun Circle.name() = "Circle"

fun printName(s: Shape) {
    println(s.name())
}

fun main() {
    printName(Circle()) // "Shape" — не "Circle"!
}
```

#### Extension properties

Можно также определять extension-свойства (без backing field):

```kotlin
val String.wordCount: Int
    get() = this.split("\\s+".toRegex()).size

fun main() {
    println("Hello beautiful world".wordCount) // 3
}
```

### Infix-функции

Функция с ключевым словом `infix` вызывается **без точки и скобок**.

Требования:
- должна быть **методом класса** или **extension-функцией**;
- должна иметь **ровно один параметр**;
- параметр **не может быть vararg** и **не может иметь значение по умолчанию**.

```kotlin
infix fun Int.pow(exponent: Int): Long {
    var result = 1L
    repeat(exponent) { result *= this }
    return result
}

fun main() {
    println(2 pow 10)  // 1024
    println(2.pow(10)) // 1024 — можно вызвать и обычным способом
}
```

Примеры из stdlib:

```kotlin
val pair = "key" to "value"   // infix fun <A, B> A.to(that: B): Pair<A, B>
val range = 1 until 10        // infix fun Int.until(to: Int): IntRange

val list = listOf(1, 2, 3, 4)
println(3 in list)             // operator, но синтаксис похож
```

### Перегрузка операторов

Kotlin позволяет определять поведение стандартных операторов (`+`, `-`, `*`, `[]`, `()` и др.) для своих классов через функции с модификатором `operator`.

```kotlin
data class Vector(val x: Double, val y: Double) {
    operator fun plus(other: Vector) = Vector(x + other.x, y + other.y)
    operator fun minus(other: Vector) = Vector(x - other.x, y - other.y)
    operator fun times(scalar: Double) = Vector(x * scalar, y * scalar)
    operator fun unaryMinus() = Vector(-x, -y)
}

fun main() {
    val a = Vector(1.0, 2.0)
    val b = Vector(3.0, 4.0)

    println(a + b)       // Vector(x=4.0, y=6.0)
    println(a - b)       // Vector(x=-2.0, y=-2.0)
    println(a * 2.0)     // Vector(x=2.0, y=4.0)
    println(-a)          // Vector(x=-1.0, y=-2.0)
}
```

Таблица основных операторов:

| Выражение | Функция |
|---|---|
| `a + b` | `a.plus(b)` |
| `a - b` | `a.minus(b)` |
| `a * b` | `a.times(b)` |
| `a / b` | `a.div(b)` |
| `a % b` | `a.rem(b)` |
| `-a` | `a.unaryMinus()` |
| `a[i]` | `a.get(i)` |
| `a[i] = v` | `a.set(i, v)` |
| `a()` | `a.invoke()` |
| `a in b` | `b.contains(a)` |
| `a == b` | `a.equals(b)` |
| `a > b` | `a.compareTo(b) > 0` |

Оператор можно определить и как **extension-функцию**:

```kotlin
operator fun Vector.get(index: Int): Double = when (index) {
    0 -> x
    1 -> y
    else -> throw IndexOutOfBoundsException("Vector has only 2 components")
}

fun main() {
    val v = Vector(3.0, 4.0)
    println(v[0]) // 3.0
    println(v[1]) // 4.0
}
```

### Delegate properties

Kotlin позволяет **делегировать** реализацию свойства внешнему объекту. Это реализуется через `by`.

#### Стандартные делегаты

**`lazy`** — значение вычисляется при первом обращении:

```kotlin
val heavyData: List<String> by lazy {
    println("Computing...")
    (1..1_000_000).map { "item-$it" }
}

fun main() {
    println("Before access")
    println(heavyData.size) // "Computing..." выводится только здесь
    println(heavyData.size) // повторного вычисления нет
}
```

**`observable`** — вызывает callback при каждом изменении:

```kotlin
import kotlin.properties.Delegates

var name: String by Delegates.observable("Unknown") { property, oldValue, newValue ->
    println("${property.name}: $oldValue -> $newValue")
}

fun main() {
    name = "Alice" // name: Unknown -> Alice
    name = "Bob"   // name: Alice -> Bob
}
```

**`vetoable`** — позволяет отклонить изменение:

```kotlin
import kotlin.properties.Delegates

var age: Int by Delegates.vetoable(0) { _, _, newValue ->
    newValue >= 0 // отклоняем отрицательные значения
}

fun main() {
    age = 25
    println(age) // 25
    age = -1
    println(age) // 25 — изменение отклонено
}
```

#### Делегирование в Map

Удобно для работы с динамическими данными (JSON, конфиги):

```kotlin
class Config(map: Map<String, Any?>) {
    val host: String by map
    val port: Int by map
    val debug: Boolean by map
}

fun main() {
    val config = Config(mapOf(
        "host" to "localhost",
        "port" to 8080,
        "debug" to true
    ))
    println("${config.host}:${config.port}, debug=${config.debug}")
    // localhost:8080, debug=true
}
```

#### Собственный делегат

Чтобы создать свой делегат, нужно реализовать операторы `getValue` (и `setValue` для `var`):

```kotlin
import kotlin.reflect.KProperty

class TrimmedString {
    private var value: String = ""

    operator fun getValue(thisRef: Any?, property: KProperty<*>): String = value

    operator fun setValue(thisRef: Any?, property: KProperty<*>, newValue: String) {
        value = newValue.trim()
    }
}

var greeting: String by TrimmedString()

fun main() {
    greeting = "   Hello, World!   "
    println("'$greeting'") // 'Hello, World!'
}
```

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

val box = Box(42)       // Box<Int>
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

---

## Аннотации

### Что такое аннотации

Аннотации — это **метаданные**, которые можно прикрепить к классам, функциям, свойствам, параметрам и т.д. Сами по себе они **не меняют поведение кода**, но могут быть прочитаны:

- **компилятором** (например, `@Deprecated`, `@Suppress`);
- **процессором аннотаций** на этапе компиляции;
- **через reflection** в рантайме.

### Стандартные аннотации

```kotlin
@Deprecated("Use newMethod() instead", ReplaceWith("newMethod()"))
fun oldMethod() { }

@Suppress("UNCHECKED_CAST")
fun unsafeCast(x: Any): List<String> = x as List<String>

@JvmStatic      // генерирует статический метод в байткоде
@JvmOverloads   // генерирует перегрузки для параметров со значениями по умолчанию
@JvmField       // убирает getter/setter, делает поле публичным
@Throws(IOException::class) // добавляет checked exception в сигнатуру для Java
```

### Создание своей аннотации

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class LogExecution(val level: String = "INFO")
```

- `@Target` — **где** можно использовать аннотацию (класс, функция, свойство, параметр и т.д.);
- `@Retention` — **как долго** аннотация живёт:
    - `SOURCE` — только в исходном коде, не попадает в байткод;
    - `BINARY` — в байткоде, но не видна через reflection;
    - `RUNTIME` — доступна через reflection в рантайме.

Использование:

```kotlin
@LogExecution(level = "DEBUG")
fun processData() {
    println("Processing...")
}
```

### Use-site targets

В Kotlin одно свойство может генерировать несколько элементов в байткоде (поле, getter, setter, параметр конструктора). Use-site target уточняет, **к чему именно** относится аннотация:

```kotlin
class User(
    @field:NotNull       // к Java-полю
    @get:JsonProperty("user_name")  // к getter-у
    @param:Valid         // к параметру конструктора
    val name: String
)
```

Доступные targets: `field`, `get`, `set`, `param`, `property`, `receiver`, `delegate`, `file`, `setparam`.

Хороший вопрос студентам:

> Зачем нужны use-site targets, если аннотация и так привязана к свойству?

---

## Упражнения

### Упражнение 1: обработка исключений

1. Напишите функцию `safeParseInt(s: String): Int?`, которая:
    - пытается преобразовать строку в `Int`;
    - при ошибке возвращает `null` (без `toIntOrNull`).

2. Напишите функцию `readIntFromConsole(): Int`, которая:
    - в цикле запрашивает ввод, пока пользователь не введёт корректное число;
    - при некорректном вводе выводит `"Некорректное число, попробуйте снова"`.

3. Перепишите `safeParseInt` с использованием `runCatching` и `Result`.

4. Ответьте письменно:
    - в чём отличие `runCatching` от `try/catch`?
    - когда `finally` полезнее, чем `use`?

### Упражнение 2: extension-функции

1. Напишите extension-функцию `String.countVowels(): Int`, которая считает количество гласных букв (латинских: a, e, i, o, u, без учёта регистра).

2. Напишите extension-свойство `List<Int>.median: Double`, которое возвращает медиану списка (бросает `IllegalArgumentException` для пустого списка).

3. Напишите extension-функцию `MutableList<T>.swap(i: Int, j: Int)`, которая меняет местами элементы по индексам.

4. Ответьте письменно:
    - можно ли в extension-функции обратиться к `private`-полю класса?
    - что произойдёт, если у класса уже есть метод с такой же сигнатурой?

### Упражнение 3: операторы и infix

1. Создайте data class `Fraction(val num: Int, val den: Int)` и определите:
    - `operator fun plus(other: Fraction): Fraction` — сложение дробей с сокращением;
    - `operator fun times(other: Fraction): Fraction` — умножение;
    - `operator fun compareTo(other: Fraction): Int` — сравнение;
    - `operator fun unaryMinus(): Fraction` — смена знака.

2. Добавьте infix-функцию `Fraction.over(den: Int): Fraction` для удобного создания: `3 over 4`.

3. Проверьте: `1 over 2 + 1 over 3` — что получится? Объясните приоритет операторов.

### Упражнение 4: delegate properties

1. Реализуйте делегат `NonNegative`, который:
    - хранит `Int`;
    - при попытке записать отрицательное значение бросает `IllegalArgumentException`.

2. Реализуйте делегат `CachedProperty<T>`, который:
    - принимает `compute: () -> T` и `ttlMs: Long`;
    - при первом чтении вычисляет значение;
    - при повторном чтении возвращает кешированное, если не прошло `ttlMs` миллисекунд;
    - иначе — перевычисляет.

3. Используйте `Delegates.observable` для реализации `var logEntries: List<String>`, который при каждом изменении выводит старое и новое значение.

### Упражнение 5: generics

1. Напишите generic-функцию `<T : Comparable<T>> List<T>.isSorted(): Boolean`, которая проверяет, отсортирован ли список по возрастанию.

2. Напишите generic-класс `Stack<T>` с методами `push(item: T)`, `pop(): T`, `peek(): T`, `isEmpty(): Boolean`. `pop()` и `peek()` должны бросать `NoSuchElementException` на пустом стеке.

3. Напишите функцию `filterByType<reified R>(list: List<Any>): List<R>`, которая возвращает только элементы заданного типа (используйте `reified`).

4. Ответьте письменно:
    - почему `Array<String>` нельзя присвоить переменной типа `Array<Any>`, а `List<String>` можно присвоить `List<Any>`?
    - в каком случае нужен `where` для generic-функции?

### Упражнение 6: аннотации

1. Создайте аннотацию `@Description(val text: String)` с `@Retention(RUNTIME)` и `@Target(CLASS, FUNCTION)`.

2. Пометьте несколько классов и функций этой аннотацией.

3. Напишите функцию `printDescriptions(vararg classes: KClass<*>)`, которая через reflection находит и выводит все `@Description` аннотации (подсказка: используйте `findAnnotation`).

4. Ответьте письменно:
    - чем `RUNTIME` retention отличается от `SOURCE`?
    - приведите пример, когда `SOURCE` retention достаточно.

---

## Что важно вынести из практики

После этой практики студент должен понимать:

1. В Kotlin нет checked exceptions — все исключения unchecked. `try` — это выражение.
2. `require`, `check`, `error` — идиоматичные способы проверок и выбрасывания исключений.
3. `runCatching` и `Result` — функциональная альтернатива `try/catch`.
4. Extension-функции — синтаксический сахар, компилируемый в статические методы. Нет полиморфизма.
5. Infix-функции — удобный DSL-подобный синтаксис для функций с одним параметром.
6. Операторы — ограниченный набор, перегружается через `operator fun`.
7. Delegate properties (`by`) — переиспользуемая логика для свойств. `lazy`, `observable`, `vetoable`, делегирование в `Map`.
8. Generics обеспечивают type safety без потери гибкости. Type bounds ограничивают допустимые типы.
9. Вариантность: `out` (ковариантность, Producer), `in` (контравариантность, Consumer), без модификатора — инвариантность.
10. Аннотации — метаданные для компилятора, процессоров и reflection. Use-site targets уточняют привязку.

---

## Идеи для устного обсуждения

- Почему Kotlin отказался от checked exceptions? Как это влияет на качество обработки ошибок?
- В чём разница между extension-функцией и обычным методом класса? Когда предпочтительнее одно, а когда другое?
- Почему extension-функции не являются полиморфными? К каким ошибкам это может привести?
- Зачем нужен `operator` — почему нельзя просто дать функции имя `plus`?
- Что произойдёт, если сделать `data class Box<T>(var value: T)` и пометить `T` как `out`?
- Когда `lazy` лучше, чем `lateinit`? А когда наоборот?
- Зачем нужны use-site targets для аннотаций? Приведите пример, когда без них аннотация попадёт не туда.
- Как вариантность влияет на API коллекций Kotlin?
