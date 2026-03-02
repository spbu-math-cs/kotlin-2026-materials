# practice-02

## null-safety

Одна из основных причин появления Kotlin.

### Как это устроено в Java и в чем проблемы

В Java `null` — это «обычное» значение ссылочного типа. Почти любая ссылка может быть `null`, но это не видно из типа.

Основные проблемы:

- **Нет информации в типах**: по сигнатуре метода нельзя понять, может ли он вернуть `null`.
- **NPE в рантайме**: ошибка возникает далеко от места, где `null` был создан/пропущен.
- **Защитный код размазывается**: постоянно приходится писать `if (x != null) ...`.
- **Аннотации помогают, но не гарантируют**: `@Nullable/@NotNull` не являются частью языка (и могут быть неверными или отсутствовать).

Пример типичной проблемы в Java:

```java
String findUserName(int id) {
    // может вернуть null
}

int len = findUserName(1).length(); // NPE в рантайме
```

### Nullable-тип в Kotlin

В Kotlin **nullability — часть системы типов**.

- `String` — не может быть `null`
- `String?` — может быть `null`

Пример:

```kotlin
val a: String = "hello"
// a = null // ошибка компиляции

val b: String? = null
```

### Где встречаются nullable-типы в Kotlin

- В ваших данных и моделях: поля, которые могут отсутствовать (`val middleName: String?`).
- В API, где «не найдено»/«не существует» выражается через `null` (например, `Map.get` возвращает `V?`).
- В результатах парсинга/поиска (`find(...)`/`firstOrNull()` и т.п.).
- При работе с Java-кодом (платформенные типы).

Примеры из stdlib:

```kotlin
val map = mapOf("a" to 1)
val x: Int? = map["missing"] // null

val y: String? = listOf("a", "b").firstOrNull { it.startsWith("z") }
```

### Вызов Java-функции из Kotlin: какие типы получаются

Если Kotlin вызывает Java-метод, Kotlin может получить:

1. **Явный nullable / not-null** — если в Java есть аннотации (`@Nullable`, `@NotNull`, JSR-305/JetBrains annotations) и они корректно настроены.
2. **Платформенный тип** — если аннотаций нет.

Платформенный тип записывают как `String!` (в IDE). Он означает:

- компилятор не знает, `null` это или нет;
- вы можете присвоить его и в `String`, и в `String?`;
- NPE может случиться в рантайме, если вы поверили, что там не `null`.

Пример:

```kotlin
// Java: String getName();  // без аннотаций
val nameFromJava = javaApi.name // тип в Kotlin: String!

val s1: String = nameFromJava      // компилятор разрешит
val s2: String? = nameFromJava     // тоже разрешит

println(s1.length) // может упасть NPE, если в Java вернули null
```

---

### ?.

Оператор безопасного вызова (safe-call-operator).

#### Семантика

`a?.b()` означает:

- если `a != null` → выполнить `a.b()`
- если `a == null` → результат всего выражения `null`

Тип результата становится nullable.

#### Пример

```kotlin
val s: String? = getNullableString()
val len: Int? = s?.length
```

#### Пример с многострочным оператором

```kotlin
val city: String? = user
    ?.address
    ?.city
    ?.trim()
    ?.takeIf { it.isNotEmpty() }
```

---

### !!

Оператор «мамой клянусь» (non-null assertion operator).

#### Семантика

`x!!`:

- если `x != null` → возвращает `x` как не-null тип
- если `x == null` → бросает исключение

#### Почему это часто плохой дизайн

`!!` — это отказ от преимуществ Kotlin null-safety. Если вы видите `!!` в бизнес-коде, обычно это признак:

- вы потеряли инвариант и не можете его выразить типами;
- API возвращает `T?`, хотя логически должен возвращать `T` или `Result`/`Either`/исключение;
- не хватает ранней проверки (`requireNotNull`) ближе к границе системы.

`!!` допустим, например, в тестах/прототипах, либо рядом с местом, где **железно** гарантирован не-null (но тогда лучше выразить это иначе).

#### Что кидается, если там на самом деле null

Бросается `kotlin.KotlinNullPointerException` (на JVM это наследник `NullPointerException`).

Пример:

```kotlin
val s: String? = null
val len = s!!.length // KotlinNullPointerException
```

---

### ?:

Elvis-оператор.

![Elvis operator (?:)](https://i.sstatic.net/bVG64.png)

Идея: «если слева null — возьми справа». 

```kotlin
val x: String? = getNullableString()
val y: String = x ?: "default"
```

#### Пример с safe-cast, когда компилятор позволяет не использовать `?.`

```kotlin
val any: Any = "hello"
val s: String = (any as? String) ?: error("Expected String")
// Здесь после elvis s уже не-null и безопасно использовать обычный вызов:
println(s.length)
```

#### Многострочный elvis

Правая часть — это обычное выражение, часто там делают ранний выход:

```kotlin
fun parsePort(value: String?): Int {
    val v = value
        ?.trim()
        ?.takeIf { it.isNotEmpty() }
        ?: return 8080

    return v.toInt()
}
```

---

### bytecode

Задача этого блока: увидеть, что null-safety — это не «магия», а набор проверок/ветвлений.

#### Пример кода

```kotlin
fun lenOrZero(s: String?): Int {
    return s?.length ?: 0
}

fun forceLen(s: String?): Int {
    return s!!.length
}
```

#### Что смотреть в байткоде

1) Для `s?.length ?: 0` вы увидите проверку `ifnull` и ветвление.

2) Для `s!!` вы увидите вызов проверки, которая бросает исключение при `null`.

Как получить байткод в IntelliJ IDEA:

- Tools → Kotlin → Show Kotlin Bytecode
- затем Decompile / View bytecode

Ожидаемые маркеры:

- `IFNULL` / `IFNONNULL` для safe-call
- `kotlin/jvm/internal/Intrinsics.checkNotNull(...)` или аналогичная проверка для `!!`

---

### Паттерны

#### Ранняя проверка через elvis + error

Паттерн (удобен, когда отсутствие значения — это ошибка программирования):

```kotlin
val x = nullableVar
    ?.calculateSomething()
    ?: error("can't be nullable")
```

Смысл:

- слева мы делаем «нормальную» работу,
- если `null` — падаем **быстро** и с понятным сообщением.

#### requireNotNull / checkNotNull: когда что использовать

Обе функции проверяют, что значение не `null`, и возвращают не-null тип.

- `requireNotNull(value)` — для **проверки аргументов** публичных функций. Бросает `IllegalArgumentException`.
- `checkNotNull(value)` — для **проверки состояния** (инвариантов) внутри объекта/процесса. Бросает `IllegalStateException`.

Примеры:

```kotlin
fun saveUser(name: String?) {
    val n = requireNotNull(name) { "name must not be null" }
    // ...
}

class Session {
    private var token: String? = null

    fun authorizedAction() {
        val t = checkNotNull(token) { "Session is not authorized" }
        println(t.length)
    }
}
```

---

## стандартная библиотека

### Зачем она нужна (мотивация)

Стандартная библиотека Kotlin (stdlib) решает несколько задач:

- **Единый набор базовых типов/расширений**: удобные функции на `String`, коллекции, `Result`, и т.д.
- **Сахар**: scope-функции (`let/run/also/apply/with`) и расширения упрощают читаемость.
- **Безопасность и контракты**: `require`, `check`, `requireNotNull`, `checkNotNull`.
- **Кроссплатформенность**: общий API для JVM/JS/Native (с нюансами реализации).

---

### сахар

Ниже — scope-функции. У них две оси различия:

1) как передается объект: `this` (receiver) или `it` (argument)
2) что возвращается: объект (`T`) или результат блока (`R`)

#### let

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T, R> T.let(block: (T) -> R): R
```

**Зачем**:

- удобно делать преобразование/маппинг значения
- удобно работать с nullable: `x?.let { ... }`

**Пример 1 (nullable)**:

```kotlin
val email: String? = user.email
email?.let { sendEmail(it) }
```

**Пример 2 (цепочка преобразований)**:

```kotlin
val len = "  hello  ".let { it.trim() }.let { it.length }
```

#### with

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T, R> with(receiver: T, block: T.() -> R): R
```

**Зачем**:

- удобен, когда есть объект и хочется много раз обратиться к нему как к `this`
- часто используют для конфигурации/вычисления на объекте без создания промежуточных переменных

**Пример 1**:

```kotlin
val fullName = with(user) {
    "$firstName $lastName".trim()
}
```

**Пример 2 (много обращений к одному объекту)**:

```kotlin
val result = with(StringBuilder()) {
    append("a")
    append("b")
    toString()
}
```

#### run

**Сигнатуры** (две формы):

```kotlin
public inline fun <T, R> T.run(block: T.() -> R): R
public inline fun <R> run(block: () -> R): R
```

**Зачем**:

- как `with`, но вызывается как extension (удобно в цепочках)
- вторая форма — для ограниченной области видимости переменных

**Пример 1 (extension run)**:

```kotlin
val normalized = "  hi  ".run { trim().uppercase() }
```

**Пример 2 (scoped variables)**:

```kotlin
val port = run {
    val env = System.getenv("PORT")
    env?.toIntOrNull() ?: 8080
}
```

#### also

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T> T.also(block: (T) -> Unit): T
```

**Зачем**:

- делать побочные эффекты (логирование, метрики) в цепочке, не меняя значение

**Пример 1 (логирование в цепочке)**:

```kotlin
val user = loadUser(id)
    .also { println("Loaded user: $it") }
    .also { metrics.count("user.loaded") }
```

**Пример 2 (инициализация + сайд-эффект)**:

```kotlin
val list = mutableListOf<Int>()
    .also { it.add(1) }
    .also { it.add(2) }
```

#### apply

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T> T.apply(block: T.() -> Unit): T
```

**Зачем**:

- конфигурирование объекта, возвращая сам объект

**Пример 1 (конфигурация)**:

```kotlin
val sb = StringBuilder().apply {
    append("Hello")
    append(", ")
    append("world")
}
println(sb.toString())
```

**Пример 2 (builder-like)**:

```kotlin
data class User(var name: String = "", var age: Int = 0)

val u = User().apply {
    name = "Alice"
    age = 20
}
```

#### use

Это расширение для `Closeable` (и `AutoCloseable`), гарантирует закрытие ресурса.

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T : java.io.Closeable?, R> T.use(block: (T) -> R): R
```

**Зачем**:

- безопасно закрывать файлы/потоки/сокеты, даже если в блоке выбросили исключение

**Пример 1 (чтение файла)**:

```kotlin
val text = java.io.File("input.txt").bufferedReader().use { it.readText() }
```

**Пример 2 (работа с output stream)**:

```kotlin
val bytes = java.io.ByteArrayOutputStream().use { out ->
    out.write("hello".toByteArray())
    out.toByteArray()
}
```

---

### iterable

`Iterable<T>` — базовый интерфейс для "по нему можно пройтись циклом".

Упрощенно:

```kotlin
interface Iterable<out T> {
    operator fun iterator(): Iterator<T>
}
```

#### Своя реализация (пример: дни недели)

```kotlin
enum class WeekDay { MON, TUE, WED, THU, FRI, SAT, SUN }

class WeekDays : Iterable<WeekDay> {
    private val values = WeekDay.entries

    override fun iterator(): Iterator<WeekDay> = values.iterator()
}

fun main() {
    for (d in WeekDays()) {
        println(d)
    }
}
```

---

### коллекции

#### принципы проектирования коллекций в Kotlin

Контракты задаются на уровне типов, а не модификаторов переменных.

![image](https://miro.medium.com/v2/resize:fit:1400/format:webp/1*i9uiJqXFcK27t-qpFJttuA.png)

##### immutable (read-only)

Только чтение (на уровне API):

- `List`
- `Set`
- `Map` (с оговорками)

Важно: `List` — «read-only view», а не обязательно «абсолютно неизменяемая структура». Если вы где-то держите ссылку на `MutableList` и отдаете ее как `List`, вы можете все равно изменить данные через мутабельную ссылку.

##### mutable

Чтение и запись:

- `MutableList`
- `MutableSet`
- `MutableMap` (с оговорками)

Почему публичный API с `MutableList` часто плохой дизайн:

- вы отдаете право модифицировать внутреннее состояние наружу;
- сложнее поддерживать инварианты (например, отсортированность);
- усложняется потокобезопасность;
- увеличивается количество неявных эффектов.

---

### List

**Пример 1: создание**

```kotlin
val xs = listOf(1, 2, 3)
val ys = mutableListOf(1, 2, 3)
ys += 4
```

**Пример 2: доступ по индексу и безопасные функции**

```kotlin
val names = listOf("a", "b")
println(names[0])
println(names.getOrNull(10)) // null
```

**Пример 3: копирование и защита от мутаций**

```kotlin
val mutable = mutableListOf(1, 2, 3)
val readOnly: List<Int> = mutable

mutable += 4
println(readOnly) // [1, 2, 3, 4] — read-only не значит immutable!

val snapshot = mutable.toList()
mutable += 5
println(snapshot) // [1, 2, 3, 4]
```

#### Map

**Пример 1: базовые операции**

```kotlin
val ages = mapOf("Alice" to 20, "Bob" to 30)
println(ages["Alice"]) // Int?
println(ages.getValue("Alice")) // Int, но бросит если нет ключа
```

**Пример 2: mutability**

```kotlin
val m = mutableMapOf<String, Int>()
m["x"] = 1
m["x"] = 2
```

**Пример 3: дефолтные значения**

```kotlin
val counts = mutableMapOf<String, Int>()
val key = "k"
counts[key] = (counts[key] ?: 0) + 1
```

#### Set

**Пример 1: уникальность**

```kotlin
val s = setOf(1, 1, 2, 2)
println(s) // [1, 2]
```

**Пример 2: проверка принадлежности**

```kotlin
val allowed = setOf("GET", "POST")
println("PUT" in allowed) // false
```

**Пример 3: мутабельный set как накопитель**

```kotlin
val unique = mutableSetOf<Int>()
listOf(1, 2, 2, 3).forEach { unique += it }
println(unique)
```

#### HashSet vs LinkedHashSet

**В чем разница?**

- `HashSet`:
  - не гарантирует порядок итерации;
  - обычно немного меньше памяти/чуть быстрее вставка.
- `LinkedHashSet`:
  - сохраняет порядок добавления (iteration order);
  - чуть больше памяти (хранит связи).

Сравнение скорости работы (идея для мини-эксперимента):

- вставить N элементов
- сделать M проверок `contains`
- сравнить время

В реальности различия зависят от JVM/данных; главное — **порядок итерации**.

---

#### Полезные методы коллекций (частые в реальном коде)

Ниже — набор операций, которые часто заменяют ручные циклы и `if`.

##### first / firstOrNull / find

- `first { ... }` — возвращает первый подходящий элемент или бросает `NoSuchElementException`.
- `firstOrNull { ... }` — возвращает `T?`.
- `find { ... }` — синоним `firstOrNull` (по смыслу «найти»).

```kotlin
val xs = listOf(1, 2, 3)

val a: Int = xs.first { it > 1 }          // 2
val b: Int? = xs.firstOrNull { it > 10 }  // null
val c: Int? = xs.find { it % 2 == 0 }     // 2
```

Типичный паттерн с elvis:

```kotlin
val user = users.firstOrNull { it.id == id } ?: return
```

##### single / singleOrNull

- `single { ... }` — ожидаем ровно один элемент, иначе исключение.
- `singleOrNull { ... }` — если 0 или >1 элементов → `null`.

```kotlin
val xs = listOf(1, 2, 3)
val one: Int = xs.single { it == 2 }          // 2
val none: Int? = xs.singleOrNull { it > 10 }  // null
```

##### take / drop / takeWhile / dropWhile

Операции «отрезать часть»:

```kotlin
val xs = listOf(1, 2, 3, 4, 5)

println(xs.take(2))            // [1, 2]
println(xs.drop(2))            // [3, 4, 5]
println(xs.takeWhile { it < 4 }) // [1, 2, 3]
println(xs.dropWhile { it < 4 }) // [4, 5]
```

С `Sequence` это становится ленивым и может рано остановиться.

##### any / all / none

Проверки предикатов:

```kotlin
val xs = listOf(1, 2, 3)

println(xs.any { it > 2 })  // true
println(xs.all { it > 0 })  // true
println(xs.none { it < 0 }) // true
```

##### count / sumOf

Частые агрегаты:

```kotlin
val words = listOf("a", "bb", "ccc")

println(words.count { it.length >= 2 }) // 2
println(words.sumOf { it.length })      // 6
```

##### distinct / distinctBy

Убрать дубликаты:

```kotlin
data class Person(val id: Int, val name: String)

val ps = listOf(Person(1, "A"), Person(1, "A2"), Person(2, "B"))

println(ps.distinctBy { it.id }) // оставит по одному на id
```

##### sortedBy / sortedWith

Сортировки:

```kotlin
val xs = listOf(3, 1, 2)
println(xs.sorted())

val ps = listOf(Person(2, "B"), Person(1, "A"))
println(ps.sortedBy { it.id })
```

##### chunked / windowed

Разбиение на куски и «скользящее окно»:

```kotlin
val xs = (1..7).toList()
println(xs.chunked(3))          // [[1, 2, 3], [4, 5, 6], [7]]
println(xs.windowed(3, step = 1)) // [[1, 2, 3], [2, 3, 4], ...]
```

---

#### персистентные коллекции

Персистентные (persistent / immutable) коллекции — структуры данных, где операции добавления/удаления возвращают **новую** коллекцию, но эффективно переиспользуют внутренние узлы (structural sharing).

##### Подключение зависимости

Gradle Kotlin DSL (пример):

```kotlin
dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-collections-immutable:0.3.8")
}
```

##### Интерфейсы

- `PersistentList<T>`
- `PersistentSet<T>`
- `PersistentMap<K, V>`

##### persistentListOf, persistentSetOf

```kotlin
val pList = kotlinx.collections.immutable.persistentListOf(1, 2, 3)
val pList2 = pList.add(4) // pList не изменился

val pSet = kotlinx.collections.immutable.persistentSetOf("a", "b")
val pSet2 = pSet.add("c")
```

---

### функциональная парадигма

**В чем смысл функциональной парадигмы?**

- меньше мутаций → проще рассуждать о коде;
- функции как значения → удобно строить переиспользуемые трансформации;
- композиция операций над коллекциями → читаемый «конвейер» данных;
- проще тестировать (меньше скрытых зависимостей и состояния).

#### функции для работы с коллекциями

Вопрос: что такое trailing lambda?

Если последний параметр функции — лямбда, ее можно вынести за скобки:

```kotlin
val xs = listOf(1, 2, 3)
val ys = xs.map({ it * 2 })
val zs = xs.map { it * 2 } // trailing lambda
```

##### filter

**Что делает**: оставляет элементы, для которых предикат вернул `true`.

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T> Iterable<T>.filter(predicate: (T) -> Boolean): List<T>
```

##### map

**Что делает**: преобразует каждый элемент.

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T, R> Iterable<T>.map(transform: (T) -> R): List<R>
```

##### groupBy

**Что делает**: группирует элементы по ключу.

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T, K> Iterable<T>.groupBy(keySelector: (T) -> K): Map<K, List<T>>
```

##### associate

**Что делает**: строит `Map<K, V>` из элементов, когда вы возвращаете `Pair<K, V>`.

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T, K, V> Iterable<T>.associate(transform: (T) -> Pair<K, V>): Map<K, V>
```

##### associateBy

**Что делает**: строит `Map<K, T>` по ключу.

**Сигнатура** (упрощенно):

```kotlin
public inline fun <T, K> Iterable<T>.associateBy(keySelector: (T) -> K): Map<K, T>
```

##### filterTo, mapTo, associateTo, etc.

**Отличие от обычных версий**:

- обычные `filter/map/...` создают **новую** коллекцию (обычно `List`/`Map`).
- версии `...To` пишут результат в **указанную** `MutableCollection`/`MutableMap`.

**Когда использовать**:

- когда хотите конкретный тип результата (например, сразу `MutableSet`),
- когда хотите переиспользовать уже созданную коллекцию (снижение аллокаций),
- когда накапливаете в существующий контейнер.

Пример «сразу в сет»:

```kotlin
val words = listOf("a", "b", "b", "ccc")

val lengths: MutableSet<Int> = mutableSetOf()
words.mapTo(lengths) { it.length }

println(lengths) // [1, 3]
```

#### pipeline

Осмысленный пример многострочного пайплайна:

Задача: из списка пользователей получить `Map<Country, List<Email>>` только для активных, у кого есть email.

```kotlin
data class User(
    val id: Long,
    val active: Boolean,
    val country: String,
    val email: String?
)

val users: List<User> = loadUsers()

val emailsByCountry: Map<String, List<String>> = users
    .asSequence()
    .filter { it.active }
    .mapNotNull { u -> u.email?.let { email -> u.country to email } }
    .groupBy(keySelector = { it.first }, valueTransform = { it.second })

println(emailsByCountry)
```

Что происходит?

- `filter` убирает неактивных
- `mapNotNull` убирает тех, у кого `email == null`, и одновременно превращает в пару `(country, email)`
- `groupBy` собирает `Map<country, List<email>>`
- `asSequence()` делает вычисления ленивыми (см. ниже)

#### asSequence

**Когда использовать**:

- большие коллекции и длинные цепочки операций,
- когда важно сократить промежуточные списки,
- когда можно выиграть на ленивости (например, есть `take(n)`/`first()` и можно рано остановиться).

**За счет чего оптимизация**:

- у `Iterable` каждая стадия (`map`, потом `filter`, ...) обычно создает промежуточную коллекцию
- `Sequence` выполняет операции **лениво**: элементы проходят через весь конвейер по одному

**Внутренняя реализация (идея на уровне концепции)**:

- `Sequence<T>` — это обертка над источником + цепочка трансформаций
- `map/filter` возвращают новые `Sequence`, которые при итерации «прокидывают» элементы дальше

Мини-пример, который показывает раннюю остановку:

```kotlin
val xs = (1..1_000_000).asSequence()
    .map { it * 2 }
    .first { it > 10 }

println(xs) // 12, и не надо вычислять весь диапазон
```

#### Преобразования коллекций

##### toList

Создает `List` (обычно копию элементов):

```kotlin
val seq = sequenceOf(1, 2, 3)
val list = seq.toList()
```

Полезно:

- чтобы «материализовать» `Sequence`
- чтобы сделать snapshot изменяемой коллекции

##### toSet

Создает `Set` (убирает дубликаты):

```kotlin
val list = listOf(1, 1, 2, 3)
val set = list.toSet()
println(set) // [1, 2, 3]
```

---

## Сложные примеры

### Пример 1 — safe-call + elvis + require/check + scope функции

Вопросы:

- что вернет функция при `null`/пустом `raw`?
- когда будет исключение и какое?
- почему тут `also`, а не `let`?

```kotlin
data class Config(val host: String, val port: Int)

fun parseConfig(raw: String?): Config {
    val normalized = raw
        ?.trim()
        ?.takeIf { it.isNotEmpty() }
        ?: return Config(host = "localhost", port = 8080)

    return normalized
        .split(':', limit = 2)
        .also { parts -> require(parts.size == 2) { "Expected host:port" } }
        .let { (host, portStr) ->
            val port = portStr.toIntOrNull() ?: error("Port is not a number: $portStr")
            Config(host = host, port = port)
        }
}
```

### Пример 2 — platform type (Java) + риск NPE

Контекст: представьте, что `legacyApi.name()` — Java-метод **без аннотаций**.

Вопросы:

- какие типы у `n1` и `n2`?
- где может произойти NPE?
- как переписать безопаснее?

```kotlin
fun demoPlatformType(legacyApi: Any): Int {
    // Псевдо-вызов Java: val nameFromJava: String! = legacyApi.name()
    val nameFromJava = legacyApi.toString() // представим, что это String!

    val n1: String = nameFromJava
    val n2: String? = nameFromJava

    return n1.length + (n2?.length ?: 0)
}
```

### Пример 3 — pipeline + groupBy + associateBy + mapTo

Вопросы:

- что получится на выходе и почему тип именно такой?
- что будет, если два пользователя с одинаковым `id`?
- где создаются коллекции? как оптимизировать?

```kotlin
data class User(val id: Long, val name: String, val email: String?, val active: Boolean)

data class EmailMessage(val to: String, val text: String)

fun buildEmails(users: List<User>): Pair<Map<Long, User>, Set<String>> {
    val activeById: Map<Long, User> = users
        .asSequence()
        .filter { it.active }
        .associateBy { it.id }

    val domains: MutableSet<String> = linkedSetOf()

    users
        .asSequence()
        .filter { it.active }
        .mapNotNull { it.email }
        .mapNotNull { email -> email.substringAfter('@', missingDelimiterValue = "").takeIf { it.isNotEmpty() } }
        .mapTo(domains) { it.lowercase() }

    return activeById to domains
}
```

### Пример 4 — Sequence и ранняя остановка + «скрытые» аллокации

Вопросы:

- сколько элементов реально будет обработано?
- чем отличается поведение, если убрать `asSequence()`?
- где здесь есть потенциальный `NumberFormatException`?

```kotlin
fun firstBigEven(raw: List<String?>): Int {
    return raw
        .asSequence()
        .mapNotNull { it?.trim() }
        .filter { it.isNotEmpty() }
        .map { it.toInt() }
        .filter { it % 2 == 0 }
        .first { it > 1_000 }
}
```

### Пример 5 — use + also/apply + nullable-логика

Вопросы:

- в каком порядке выполняются операции?
- что гарантирует `use`?
- что будет, если файл пустой?

```kotlin
fun readFirstLineOrNull(path: String): String? {
    return java.io.File(path)
        .takeIf { it.exists() }
        ?.bufferedReader()
        ?.use { br ->
            br.readLine()
                ?.trim()
                ?.takeIf { it.isNotEmpty() }
        }
        .also { line ->
            if (line == null) println("No non-empty first line in $path")
        }
}
```

### Пример 6 — «плохой» пример с !! (на поиск альтернатив)

Вопросы:

- какие входы приводят к падению?
- какое исключение и где?
- как переписать через `requireNotNull` или через elvis + `return`?

```kotlin
data class Profile(val nickname: String?)

data class Account(val profile: Profile?)

fun nicknameLength(acc: Account?): Int {
    return acc!!.profile!!.nickname!!.length
}
```

### Пример 7 — firstOrNull + elvis + ранний return (поиск ресурса)

Вопросы:

- какие ветки возможны?
- почему `firstOrNull`, а не `first`?
- какой тип у `candidate`?

```kotlin
data class Feature(val name: String, val enabled: Boolean)

fun isFeatureEnabled(features: List<Feature>?, name: String): Boolean {
    val candidate = features
        ?.firstOrNull { it.name == name }
        ?: return false

    return candidate.enabled
}
```

### Пример 8 — singleOrNull (строгая уникальность) + require

Вопросы:

- почему `singleOrNull`, а не `firstOrNull`?
- когда вернется `null`, а когда упадем на `require`?

```kotlin
data class Order(val id: String, val status: String)

fun shippedOrderOrThrow(orders: List<Order>, id: String): Order {
    val order = orders.singleOrNull { it.id == id }
        ?: error("Order not found or not unique: $id")

    require(order.status == "SHIPPED") { "Order $id is not shipped (status=${order.status})" }
    return order
}
```

### Пример 9 — take/drop + Sequence (пагинация/ограничение) + материализация

Вопросы:

- где именно происходит вычисление?
- что поменяется, если убрать `.asSequence()`?

```kotlin
data class LogLine(val level: String, val text: String)

fun firstErrors(lines: List<LogLine>, skip: Int, limit: Int): List<String> {
    return lines
        .asSequence()
        .drop(skip)
        .filter { it.level == "ERROR" }
        .take(limit)
        .map { it.text }
        .toList()
}
```

### Пример 10 — distinctBy + sortedBy + take (топ-N уникальных)

Вопросы:

- почему `distinctBy` стоит до сортировки/после — что изменится?
- что попадет в результат, если есть несколько событий с одинаковым `userId`?

```kotlin
data class Event(val userId: Long, val timestamp: Long, val type: String)

fun topRecentUsers(events: List<Event>, n: Int): List<Long> {
    return events
        .sortedByDescending { it.timestamp }
        .distinctBy { it.userId }
        .take(n)
        .map { it.userId }
}
```

### Пример 11 — any/all/none для бизнес-правил

Вопросы:

- чем отличается `any` от `all`?
- что будет на пустом списке?

```kotlin
data class Item(val name: String, val qty: Int)

data class Cart(val items: List<Item>)

fun validateCart(cart: Cart): Boolean {
    val hasNegative = cart.items.any { it.qty < 0 }
    val allNonEmptyNames = cart.items.all { it.name.isNotBlank() }
    val noneZero = cart.items.none { it.qty == 0 }

    return !hasNegative && allNonEmptyNames && noneZero
}
```

### Пример 12 — sumOf + groupBy (агрегация) + Map.getValue

Вопросы:

- какие типы у `totalsByUser` и `totalFor42`?
- чем `getValue` отличается от `map[key]`?

```kotlin
data class Payment(val userId: Long, val amount: Long)

fun totals(payments: List<Payment>): Long {
    val totalsByUser: Map<Long, Long> = payments
        .groupBy { it.userId }
        .mapValues { (_, ps) -> ps.sumOf { it.amount } }

    val totalFor42: Long = totalsByUser.getValue(42L)
    return totalFor42
}
```

### Пример 13 — chunked и windowed (батчи и скользящее окно)

Вопросы:

- что вернет `chunked(3)` для 7 элементов?
- чем `windowed(3, 1)` отличается от `chunked(3)`?

```kotlin
fun batchAndTrend(xs: List<Int>): Pair<List<List<Int>>, List<Int>> {
    val batches: List<List<Int>> = xs.chunked(3)

    // сумма каждого окна длиной 3
    val trend: List<Int> = xs
        .windowed(size = 3, step = 1, partialWindows = false)
        .map { w -> w.sum() }

    return batches to trend
}
```
