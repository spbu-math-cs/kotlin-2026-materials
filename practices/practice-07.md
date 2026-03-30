# practice-07: аннотации и reflection

## План

1. Аннотации — что такое, отличия от Java, стандартные аннотации Kotlin, создание своих, use-site targets, применение аннотаций
2. Reflection — зачем нужна, kotlin-reflect, KClass, KFunction, KProperty, работа с аннотациями через reflection
3. Применения reflection — сериализация, DI, тестовые фреймворки, ORM, валидация
4. Вопросы
5. Итоги

---

## Аннотации

### Что такое аннотации

Аннотации — это **метаданные**, прикреплённые к элементам кода (классам, функциям, свойствам, параметрам и т.д.). Сами по себе они **не меняют поведение программы**, но могут быть прочитаны:

- **компилятором** — `@Deprecated`, `@Suppress`, `@OptIn`;
- **расширениями компилятора** (compiler plugins) — `@Serializable` (kotlinx.serialization), `@Composable` (Jetpack Compose);
- **процессорами аннотаций** на этапе компиляции (kapt, KSP) — генерация кода;
- **через reflection** в рантайме — фреймворки (Spring, JUnit, Jackson).

### Отличия от Java

| Аспект | Java | Kotlin |
|---|---|---|
| Объявление | `@interface MyAnnotation {}` | `annotation class MyAnnotation` |
| Параметры | Методы: `String value()` | Свойства конструктора: `val value: String` |
| Значение по умолчанию | `String value() default "x"` | `val value: String = "x"` |
| Допустимые типы параметров | Примитивы, `String`, `Class`, enum, другие аннотации, массивы из них | То же + `KClass` вместо `Class` |
| Массив в параметре | `String[] tags()`, передаётся `{"a", "b"}` | `vararg val tags: String`, передаётся `["a", "b"]` |
| Checked exceptions | `@throws` не обязателен | `@Throws` нужен для Java-interop |
| Use-site targets | Не нужны (однозначная привязка) | Нужны: `@field:`, `@get:`, `@param:` и т.д. |
| `@Repeatable` | Требует контейнерную аннотацию вручную | Встроенная поддержка через `@Repeatable` |

Пример различий в синтаксисе:

```java
// Java
@Retention(RetentionPolicy.RUNTIME)
@Target(ElementType.METHOD)
public @interface CacheResult {
    long ttlSeconds() default 60;
    String key() default "";
}
```

```kotlin
// Kotlin
@Retention(AnnotationRetention.RUNTIME)
@Target(AnnotationTarget.FUNCTION)
annotation class CacheResult(
    val ttlSeconds: Long = 60,
    val key: String = "",
)
```

### Стандартные аннотации Kotlin

#### Аннотации для разработчика

```kotlin
// Помечает устаревший API с подсказкой замены
@Deprecated(
    message = "Use newMethod() instead",
    replaceWith = ReplaceWith("newMethod()"),
    level = DeprecationLevel.WARNING, // WARNING → ERROR → HIDDEN
)
fun oldMethod() {}

// Подавление предупреждений компилятора
@Suppress("UNCHECKED_CAST")
fun unsafeCast(x: Any): List<String> = x as List<String>

// Пометка экспериментального API
@RequiresOptIn(message = "This API is experimental", level = RequiresOptIn.Level.WARNING)
annotation class ExperimentalMyApi

@ExperimentalMyApi
fun experimentalFeature() {}

// Использование экспериментального API
@OptIn(ExperimentalMyApi::class)
fun useExperimental() {
    experimentalFeature()
}
```

#### Аннотации для JVM-interop

```kotlin
class Api {
    companion object {
        @JvmStatic // генерирует статический метод — вызывается из Java как Api.create()
        fun create(): Api = Api()

        @JvmField  // делает поле публичным в Java без getter/setter
        val VERSION: String = "1.0"
    }

    @JvmOverloads // генерирует перегрузки для параметров со значениями по умолчанию
    fun fetch(url: String, timeout: Int = 5000, retries: Int = 3) {}

    @Throws(java.io.IOException::class) // добавляет checked exception для Java
    fun load() { /* ... */ }
}
```

Почему это важно: Kotlin-код часто вызывается из Java. Без `@JvmStatic` в Java придётся писать `Api.Companion.create()`, без `@JvmOverloads` — передавать все параметры.

#### Другие полезные аннотации

```kotlin
@Volatile   // аналог volatile в Java — гарантия видимости между потоками
var flag: Boolean = false

@Synchronized  // аналог synchronized метода в Java
fun criticalSection() { /* ... */ }

@Transient  // поле не сериализуется (JVM-аннотация kotlin.jvm.Transient)
val cache: MutableMap<String, Any> = mutableMapOf()

@PublishedApi  // internal-функция доступна из inline-функций
internal fun helperForInline() {}
```

### Создание своей аннотации

```kotlin
@Target(AnnotationTarget.FUNCTION, AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class LogExecution(val level: String = "INFO")
```

- **`@Target`** — **где** можно использовать:
  `CLASS`, `FUNCTION`, `PROPERTY`, `FIELD`, `VALUE_PARAMETER`, `CONSTRUCTOR`, `EXPRESSION`, `FILE`, `TYPE`, `TYPEALIAS`, `LOCAL_VARIABLE`, `ANNOTATION_CLASS`

- **`@Retention`** — **как долго** живёт аннотация:
    - `SOURCE` — только в исходном коде, не попадает в байткод;
    - `BINARY` — попадает в байткод, но **не видна** через reflection;
    - `RUNTIME` — доступна через reflection в рантайме (самая частая для фреймворков).

Использование:

```kotlin
@LogExecution(level = "DEBUG")
fun processData() {
    println("Processing...")
}
```

### Use-site targets

В Kotlin одно свойство в конструкторе генерирует **несколько элементов** в байткоде: поле, getter, setter, параметр конструктора. Use-site target уточняет, **к чему именно** относится аннотация:

> Примечание: `NotNull`, `JsonProperty`, `Valid` в примере ниже — аннотации из сторонних библиотек (валидация / Jackson) и приведены только для иллюстрации.

```kotlin
class User(
    @field:NotNull                  // к Java-полю
    @get:JsonProperty("user_name") // к getter-у
    @param:Valid                   // к параметру конструктора
    val name: String,
)
```

Доступные targets:

| Target | К чему привязывается |
|---|---|
| `field` | Backing field свойства |
| `get` | Getter свойства |
| `set` | Setter свойства |
| `param` | Параметр конструктора |
| `property` | Kotlin property (не видно из Java) |
| `receiver` | Receiver extension-функции |
| `delegate` | Поле, хранящее делегат |
| `file` | Весь файл (ставится перед `package`) |
| `setparam` | Параметр setter-а |

Пример с `file`:

```kotlin
@file:JvmName("StringUtils") // меняет имя файла-класса в байткоде
// package com.example.utils

fun String.capitalizeFirst(): String =
    replaceFirstChar { ch -> ch.titlecase() }
```

> Примечание: `@file:JvmName` применяется ко *всему файлу* и в реальном коде должен стоять в самом начале файла, перед `package`.

Без `@file:JvmName` из Java пришлось бы писать `StringUtilsKt.capitalizeFirst(...)`, а с ней — `StringUtils.capitalizeFirst(...)`.

### Как аннотации используются на практике

#### 1. Compiler plugins (расширения компилятора)

Некоторые аннотации обрабатываются **расширениями компилятора** — плагинами, которые встраиваются в процесс компиляции и **трансформируют код или генерируют дополнительный байткод**.

**kotlinx.serialization** — аннотация `@Serializable` обрабатывается compiler plugin, который генерирует `serializer()` прямо в байткоде:

```kotlin
@Serializable
data class User(val name: String, val age: Int)

// Сгенерированный код позволяет:
val json = Json.encodeToString(User("Alice", 25))
val user = Json.decodeFromString<User>(json)
```

**Jetpack Compose** — `@Composable` трансформирует функцию через compiler plugin, добавляя механизм рекомпозиции:

```kotlin
@Composable
fun Greeting(name: String) {
    Text("Hello, $name!")
}
```

Без compiler plugin эта аннотация была бы просто маркером без эффекта.

#### 2. Annotation processing (kapt / KSP)

Процессоры аннотаций **генерируют новые файлы с кодом** на этапе компиляции.

- **kapt** (Kotlin Annotation Processing Tool) — оборачивает Java-процессоры для Kotlin. Медленный, устаревший подход.
- **KSP** (Kotlin Symbol Processing) — нативный для Kotlin, работает быстрее kapt.

Примеры: Dagger/Hilt (`@Inject`, `@Module`), Room (`@Entity`, `@Dao`).

##### Зачем нужен annotation processor, если можно всё сделать через reflection в рантайме?

Короткий ответ: **можно**, но это будет другой набор компромиссов. Annotation processing (и KSP/kapt) переносит «магию по аннотациям» из рантайма в **compile-time**, и за счёт этого даёт преимущества.

Сравнение подходов:

| Критерий | Annotation processing (KSP/kapt) | Runtime reflection |
|---|---|---|
| Когда происходит анализ | На этапе компиляции | На старте/в рантайме |
| Производительность | Быстрее в рантайме (обычный сгенерированный код) | Обычно медленнее (сканирование + `invoke/call`) |
| Ошибки | Ловятся компилятором (раньше) | Часто проявляются в рантайме |
| Type safety | Выше: генерируется типобезопасный код | Ниже: много `Any?`, кастов и ошибок выполнения |
| Размер/платформа | Нет `kotlin-reflect` в рантайме; лучше для Android | `kotlin-reflect` может быть тяжёлым; R8/ProGuard + keep rules |
| Обфускация | Обычно устойчивее (работа на этапе компиляции) | Может ломаться из-за переименований |
| Мультиплатформа | KSP часто применим шире | Reflection может быть ограничен/дорог на не-JVM |

Жизненные примеры:
- На **Android** DI через генерацию кода (Dagger/Hilt, Koin KSP) уменьшает стоимость старта и избегает многих проблем reflection.
- В **серверных JVM** проектах runtime reflection часто допустим (Spring), но платится временем старта и «магией».

Правило-памятка:
- Нужна **скорость старта/типобезопасность/ранние ошибки** → KSP/kapt.
- Нужна **динамика** (плагины, неизвестные типы) → runtime reflection.

#### 3. Runtime reflection

Фреймворки читают аннотации в рантайме через reflection:

- **Spring**: `@Controller`, `@Service`, `@Autowired`
- **JUnit**: `@Test`, `@BeforeEach`, `@ParameterizedTest`
- **Jackson**: `@JsonProperty`, `@JsonIgnore`

Этот подход мы подробно разберём в следующем разделе.

### Упражнения: Аннотации

1. Создайте аннотацию `@Description(val text: String)` с `@Retention(RUNTIME)` и `@Target(CLASS, FUNCTION)`. Пометьте ей 2–3 класса и функции.

2. Создайте аннотацию `@Validate` с `@Target(VALUE_PARAMETER)` и `@Retention(RUNTIME)`:
    - параметр `min: Int = Int.MIN_VALUE`
    - параметр `max: Int = Int.MAX_VALUE`
    - параметр `notBlank: Boolean = false`

   Пометьте ей параметры какой-нибудь data class.

3. Объясните письменно:
    - к какому элементу в байткоде попадёт `@NotNull` в `data class User(@NotNull val name: String)` без use-site target?
    - как это изменить, чтобы аннотация попала именно на поле?

4. Ответьте письменно:
    - почему DI/ORM библиотеки на Android часто предпочитают KSP/kapt, а не runtime reflection?
    - приведите один пример задачи, где runtime reflection удобнее/неизбежнее (почему генерация кода не подходит).

---

## Reflection

> Для примеров ниже обычно нужны импорты:
>
> ```kotlin
> // import kotlin.reflect.KClass
> // import kotlin.reflect.full.*
> ```

### Что такое reflection

Reflection — это способность программы **исследовать и изменять свою структуру** во время выполнения. Вместо того чтобы обращаться к классам и функциям напрямую (статически), мы получаем их описание как **объекты** и работаем с ними программно.

Зачем это нужно:
- код **не знает заранее**, с какими классами будет работать (фреймворки, плагины);
- нужно **читать метаданные** (аннотации) в рантайме;
- нужно **динамически вызывать** функции или создавать объекты.

### Зависимость kotlin-reflect

Kotlin reflection — **отдельная библиотека**, не входит в stdlib:

```kotlin
// build.gradle.kts
dependencies {
    implementation(kotlin("reflect"))
}
```

Без неё `KClass` доступен, но большинство его методов бросят `KotlinReflectionNotSupportedError`.

### Java Reflection vs Kotlin Reflection

Важно не путать:

- **JVM/Java reflection как универсальный механизм** (часть JDK): `java.lang.Class`, `java.lang.reflect.Method/Field/Constructor`.
  Он работает с байткодом и применим к классам из **любого** JVM-языка (Java/Kotlin/Scala/и т.д.).

- **Kotlin reflection как языко-специфичная библиотека** (`kotlin-reflect`): `kotlin.reflect.KClass`, `KFunction`, `KProperty`, `KType`.
  Это более высокоуровневое API, которое поверх Java reflection восстанавливает Kotlin-модель (свойства, nullability, `data/sealed/object`, имена параметров) на основе **Kotlin metadata** (`@kotlin.Metadata`).

Следствие: для классов, скомпилированных не из Kotlin (например, Scala), Kotlin reflection обычно даёт только то, что можно вывести из байткода (часто через `.java`-view), без «кotlin-специфичных» деталей.

#### 1) Базовые типы и соответствия

| Что хотим описать | Java | Kotlin |
|---|---|---|
| Класс | `Class<T>` | `KClass<T>` |
| Метод/функция | `Method` | `KFunction<*>` |
| Поле | `Field` | чаще всего `KProperty<*>` (а backing field может отсутствовать) |
| Конструктор | `Constructor<T>` | `KFunction<T>` из `constructors` / `primaryConstructor` |
| Тип (с generics) | `Type` (`ParameterizedType`, ...) | `KType` (`returnType`, `starProjectedType`, ...) |

Переходы туда-обратно:

```kotlin
val kClass = String::class
val jClass = String::class.java     // Class<String>
val kAgain = String::class.java.kotlin // KClass<String>
```

> Важно: `::class` — это Kotlin (`KClass`), `.java` — мост к Java (`Class`).

#### 2) Свойства vs поля/геттеры

В Kotlin `val/var` — это **свойство** (property), которое может компилироваться в:
- поле + getter (и setter для `var`)
- только getter (без поля)
- делегат (дополнительные синтетические поля)

Java reflection видит то, что реально лежит в байткоде: поля и методы.

Пример:

```kotlin
data class User(val name: String) {
    val upper: String get() = name.uppercase() // поля НЕ будет
}
```

- Kotlin reflection: увидит `KProperty upper`
- Java reflection: увидит метод `getUpper()`, но **не увидит Field**

Из-за этого для фреймворков на Java иногда приходится помогать аннотациями (`@field:...`, `@get:...`).

#### 3) Kotlin-специфичная информация (metadata)

Kotlin compiler кладёт в классы аннотацию `@kotlin.Metadata`, по которой Kotlin reflection восстанавливает:
- имена параметров;
- nullability (`String` vs `String?`) на уровне `KType`;
- информацию о `data class`, `sealed`, `object`, `value class` и т.д.

Java reflection:
- не знает про nullability как про типовую информацию (максимум — аннотации);
- видит только то, что выражено в байткоде.

#### 4) Производительность и размер

- Java reflection — часть JDK, доступна всегда, обычно легче по размеру.
- Kotlin reflection требует `kotlin-reflect` (добавляет заметный вес) и часто даёт более «высокоуровневое» API.

#### 5) Interop: как добраться до Java Method/Field из Kotlin reflection

Иногда нужно стартовать с Kotlin (`KFunction`/`KProperty`), а потом получить Java-элемент:

```kotlin
// import kotlin.reflect.jvm.javaField
// import kotlin.reflect.jvm.javaGetter
// import kotlin.reflect.jvm.javaMethod

data class Person(val name: String)

fun Person.greet(): String = "Hi, $name"

fun main() {
    val nameProp = Person::name
    println(nameProp.javaField)   // Field? (может быть null, если поля нет)
    println(nameProp.javaGetter)  // Method? getter

    val greetFn = Person::greet
    println(greetFn.javaMethod)   // Method?
}
```

> Здесь используются расширения из пакета `kotlin.reflect.jvm.*`.

#### 6) Доступ к private

И Java, и Kotlin reflection позволяют обходить ограничения видимости, но это плохая практика вне тестов/инструментов:

- Java: `method.isAccessible = true` (или `setAccessible(true)`)
- Kotlin: `callable.isAccessible = true` (нужно `kotlin.reflect.jvm.isAccessible`)

### Упражнения: Java vs Kotlin Reflection

1. Напишите `fun printJavaAndKotlinNames(obj: Any)`, которая выводит:
   - `obj::class.qualifiedName`
   - `obj.javaClass.name`
   - а также проверьте (письменно), совпадают ли они всегда.

2. Возьмите класс с вычисляемым свойством без backing field (как `upper` выше) и:
   - найдите это свойство через Kotlin reflection;
   - попытайтесь найти одноимённое поле через Java reflection;
   - объясните результат.

---

### KClass — описание класса

`KClass<T>` — ключевой тип Kotlin reflection. Получить его можно двумя способами:

```kotlin
// import kotlin.reflect.KClass

// 1. Через литерал типа
val kClass: KClass<String> = String::class

// 2. Через экземпляр объекта
val str = "hello"
val kClass2: KClass<out String> = str::class
```

Что можно узнать о классе:

```kotlin
data class User(val name: String, val age: Int) {
    fun greet() = "Hi, I'm $name"
}

fun main() {
    val kClass = User::class

    println(kClass.simpleName)       // "User"
    println(kClass.qualifiedName)    // "com.example.User"
    println(kClass.isData)           // true
    println(kClass.isAbstract)       // false
    println(kClass.isSealed)         // false
    println(kClass.visibility)       // PUBLIC

    // Все члены: свойства + функции + наследованные
    kClass.members.forEach { println("  ${it.name}") }

    // Только объявленные свойства
    kClass.declaredMemberProperties.forEach {
        println("  property: ${it.name} : ${it.returnType}")
    }

    // Конструкторы
    kClass.constructors.forEach {
        println("  constructor: ${it.parameters.map { p -> "${p.name}: ${p.type}" }}")
    }
}
```

### KFunction и KProperty — описание функций и свойств

```kotlin
// import kotlin.reflect.full.declaredFunctions
// import kotlin.reflect.full.declaredMemberProperties

fun main() {
    val kClass = User::class

    // Функции
    val greetFn = kClass.declaredFunctions.first { it.name == "greet" }
    println(greetFn.name)            // "greet"
    println(greetFn.returnType)      // kotlin.String
    println(greetFn.parameters)      // [instance parameter]

    // Вызов через reflection
    val user = User("Alice", 25)
    val result = greetFn.call(user)  // "Hi, I'm Alice"
    println(result)

    // Свойства
    val nameProp = kClass.declaredMemberProperties.first { it.name == "name" }
    println(nameProp.get(user))      // "Alice"
}
```

### Ссылки на функции и свойства (callable references)

Kotlin позволяет получить reflection-объект через оператор `::`:

```kotlin
fun double(x: Int) = x * 2

fun main() {
    val ref = ::double          // KFunction1<Int, Int>
    println(ref.name)           // "double"
    println(ref(21))            // 42
    println(ref.call(21))       // 42

    // Ссылка на свойство
    val nameProp = User::name   // KProperty1<User, String>
    val user = User("Bob", 30)
    println(nameProp.get(user)) // "Bob"

    // Ссылка на конструктор
    val ctor = ::User           // KFunction2<String, Int, User>
    val newUser = ctor("Eve", 22)
    println(newUser)            // User(name=Eve, age=22)
}
```

### Чтение аннотаций через reflection

```kotlin
// import kotlin.reflect.full.findAnnotation
// import kotlin.reflect.full.hasAnnotation

@Target(AnnotationTarget.CLASS, AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Description(val text: String)

@Description("Represents a product in the catalog")
data class Product(val id: Int, val title: String, val price: Double)

@Description("Formats price for display")
fun Product.formatPrice(): String = "$$price"

fun main() {
    val kClass = Product::class

    // Чтение аннотации с класса
    val desc = kClass.findAnnotation<Description>()
    println(desc?.text)  // "Represents a product in the catalog"

    // Проверка наличия аннотации
    println(kClass.hasAnnotation<Description>())  // true

    // Чтение аннотаций со всех функций
    Product::class.members
        .filter { it.annotations.any { a -> a is Description } }
        .forEach {
            val annotation = it.findAnnotation<Description>()
            println("${it.name}: ${annotation?.text}")
        }
}
```

### Упражнения: Reflection — основы

1. Напишите функцию `inspectClass(kClass: KClass<*>)`, которая выводит:
    - имя класса;
    - является ли `data class`;
    - список свойств с типами;
    - список функций (исключая стандартные `equals`, `hashCode`, `toString`).

2. Напишите функцию `printDescriptions(vararg classes: KClass<*>)`, которая через reflection находит и выводит все `@Description` аннотации на классах и их функциях (подсказка: `findAnnotation`).

3. Напишите функцию `createInstance(kClass: KClass<*>, args: Map<String, Any?>): Any?`, которая:
    - находит primary constructor;
    - сопоставляет `args` по именам параметров;
    - создаёт экземпляр через `callBy`.

   Проверьте на `data class Config(val host: String, val port: Int, val debug: Boolean)`.

---

## Применения reflection

Reflection — основа многих фреймворков и инструментов. Рассмотрим ключевые сценарии.

### 1. Сериализация / десериализация

Одно из самых частых применений: превращение объекта в строку (JSON, XML) и обратно **без ручного маппинга**.

Идея: через reflection обходим свойства класса и собираем их значения.

```kotlin
// import kotlin.reflect.full.declaredMemberProperties

fun toJsonSimple(obj: Any): String {
    val kClass = obj::class
    val props = kClass.declaredMemberProperties
    val entries = props.joinToString(", ") { prop ->
        val value = prop.getter.call(obj)
        val formatted = if (value is String) "\"$value\"" else "$value"
        "\"${prop.name}\": $formatted"
    }
    return "{ $entries }"
}

fun main() {
    data class User(val name: String, val age: Int)
    println(toJsonSimple(User("Alice", 25)))
    // { "name": "Alice", "age": 25 }
}
```

Реальные библиотеки (Jackson, Gson, Moshi) делают то же самое, но обрабатывают вложенные объекты, коллекции, null-ы, кастомные аннотации (`@JsonProperty`, `@JsonIgnore`) и т.д.

### 2. Dependency Injection (DI)

Фреймворки вроде Spring / Koin / Dagger используют reflection для автоматической «сборки» зависимостей.

Упрощённая идея:

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class Injectable

@Injectable
class UserRepository {
    fun findAll() = listOf("Alice", "Bob")
}

@Injectable
class UserService(val repo: UserRepository) {
    fun listUsers() = repo.findAll()
}

// Простейший DI-контейнер
class Container {
    private val instances = mutableMapOf<KClass<*>, Any>()

    inline fun <reified T : Any> resolve(): T = resolve(T::class) as T

    fun resolve(kClass: KClass<*>): Any {
        // Возвращаем кешированный экземпляр
        instances[kClass]?.let { return it }

        // Находим primary constructor
        val ctor = kClass.primaryConstructor
            ?: error("No primary constructor for ${kClass.simpleName}")

        // Рекурсивно резолвим зависимости
        val args = ctor.parameters.map { param ->
            resolve(param.type.classifier as KClass<*>)
        }.toTypedArray()

        val instance = ctor.call(*args)
        instances[kClass] = instance
        return instance
    }
}

fun main() {
    val container = Container()
    val service = container.resolve<UserService>()
    println(service.listUsers())  // [Alice, Bob]
}
```

### 3. Тестовые фреймворки

JUnit использует reflection для:
- **обнаружения** тестовых методов (помеченных `@Test`);
- **создания** экземпляра тестового класса;
- **вызова** `@BeforeEach` → `@Test` → `@AfterEach`.

Упрощённый тестовый раннер:

```kotlin
@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class Test

@Target(AnnotationTarget.FUNCTION)
@Retention(AnnotationRetention.RUNTIME)
annotation class BeforeEach

fun runTests(testClass: KClass<*>) {
    val instance = testClass.createInstance()

    val beforeEach = testClass.declaredFunctions
        .filter { it.hasAnnotation<BeforeEach>() }

    val tests = testClass.declaredFunctions
        .filter { it.hasAnnotation<Test>() }

    for (test in tests) {
        beforeEach.forEach { it.call(instance) }

        try {
            test.call(instance)
            println("✅ ${test.name}")
        } catch (e: Exception) {
            println("❌ ${test.name}: ${e.cause?.message ?: e.message}")
        }
    }
}
```

### 4. ORM (Object-Relational Mapping)

ORM-библиотеки (Hibernate, Exposed) через reflection:
- маппят **свойства класса → колонки таблицы**;
- читают аннотации (`@Entity`, `@Column`, `@Id`);
- генерируют SQL-запросы.

```kotlin
@Target(AnnotationTarget.CLASS)
@Retention(AnnotationRetention.RUNTIME)
annotation class Table(val name: String)

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Column(val name: String = "")

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Id

@Table("users")
data class UserEntity(
    @Id @Column("user_id") val id: Int,
    @Column("user_name") val name: String,
    @Column val email: String,
)

fun generateCreateTable(kClass: KClass<*>): String {
    val table = kClass.findAnnotation<Table>()
        ?: error("Class ${kClass.simpleName} is not annotated with @Table")

    val columns = kClass.declaredMemberProperties.joinToString(",\n    ") { prop ->
        val col = prop.findAnnotation<Column>()
        val colName = col?.name?.ifEmpty { prop.name } ?: prop.name
        val type = when (prop.returnType.classifier) {
            Int::class -> "INTEGER"
            String::class -> "TEXT"
            Double::class -> "REAL"
            Boolean::class -> "BOOLEAN"
            else -> "TEXT"
        }
        val pk = if (prop.hasAnnotation<Id>()) " PRIMARY KEY" else ""
        "$colName $type$pk"
    }

    return "CREATE TABLE ${table.name} (\n    $columns\n);"
}

fun main() {
    println(generateCreateTable(UserEntity::class))
    // CREATE TABLE users (
    //     user_id INTEGER PRIMARY KEY,
    //     user_name TEXT,
    //     email TEXT
    // );
}
```

### 5. Валидация

Reflection позволяет создавать **декларативную валидацию** — правила задаются аннотациями, а валидатор читает их автоматически:

```kotlin
@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Min(val value: Int)

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class Max(val value: Int)

@Target(AnnotationTarget.PROPERTY)
@Retention(AnnotationRetention.RUNTIME)
annotation class NotBlank

data class RegistrationForm(
    @NotBlank val username: String,
    @Min(8) @Max(64) val password: String,
    @Min(18) val age: Int,
)

fun validate(obj: Any): List<String> {
    val errors = mutableListOf<String>()

    for (prop in obj::class.declaredMemberProperties) {
        val value = prop.getter.call(obj)

        prop.findAnnotation<NotBlank>()?.let {
            if (value is String && value.isBlank()) {
                errors += "${prop.name} must not be blank"
            }
        }
        prop.findAnnotation<Min>()?.let { min ->
            if (value is Int && value < min.value) {
                errors += "${prop.name} must be >= ${min.value}, got $value"
            }
            if (value is String && value.length < min.value) {
                errors += "${prop.name} length must be >= ${min.value}, got ${value.length}"
            }
        }
        prop.findAnnotation<Max>()?.let { max ->
            if (value is Int && value > max.value) {
                errors += "${prop.name} must be <= ${max.value}, got $value"
            }
            if (value is String && value.length > max.value) {
                errors += "${prop.name} length must be <= ${max.value}, got ${value.length}"
            }
        }
    }

    return errors
}

fun main() {
    val form = RegistrationForm(username = "", password = "123", age = 15)
    validate(form).forEach { println("⚠️ $it") }
    // ⚠️ username must not be blank
    // ⚠️ password length must be >= 8, got 3
    // ⚠️ age must be >= 18, got 15
}
```

### Цена reflection

Reflection — мощный инструмент, но у него есть цена:

| Аспект | Проблема |
|---|---|
| **Производительность** | Вызов через reflection в ~10–100× медленнее прямого вызова |
| **Type safety** | Ошибки обнаруживаются в рантайме, а не при компиляции |
| **Обфускация** | ProGuard/R8 переименовывают классы — reflection ломается |
| **Размер** | `kotlin-reflect` добавляет ~3 МБ к размеру приложения |

Когда **не** использовать reflection:
- если задача решается обычным полиморфизмом или generics;
- в горячих путях (вызывается тысячи раз в секунду);
- в Android без крайней необходимости (размер APK).

Альтернативы:
- **Compiler plugins** (kotlinx.serialization) — генерация кода при компиляции, нулевой рантайм-overhead;
- **KSP** — генерация кода на этапе сборки;
- **`reified` generics в `inline`-функциях** — type-safe доступ к типу без reflection.

### Упражнения: Reflection — применения

1. **Мини-сериализатор**: доработайте функцию `toJsonSimple` так, чтобы:
    - она поддерживала вложенные объекты (data class внутри data class);
    - игнорировала свойства, помеченные аннотацией `@Transient`;
    - переименовывала поля по аннотации `@JsonName(val name: String)`.

   Пример:
   ```kotlin
   annotation class JsonName(val name: String)

   data class Address(val city: String, val zip: String)
   data class Person(
       @JsonName("full_name") val name: String,
       val age: Int,
       @Transient val password: String,
       val address: Address,
   )

   println(toJson(Person("Alice", 25, "secret", Address("Moscow", "101000"))))
   // { "full_name": "Alice", "age": 25, "address": { "city": "Moscow", "zip": "101000" } }
   ```

2. **Мини-тест-раннер**: используя аннотации `@Test` и `@BeforeEach` из примера выше, напишите класс с 3–4 тестовыми методами и запустите их через `runTests()`. Добавьте подсчёт результатов: `Passed: X, Failed: Y`.

3. **Валидатор**: расширьте пример валидации, добавив аннотацию `@Email`, которая проверяет, что строка содержит `@` и `.` после `@`. Протестируйте на нескольких примерах.

4. Ответьте письменно:
    - почему `kotlinx.serialization` использует compiler plugin, а не reflection?
    - в каких случаях reflection — единственный вариант (нет альтернатив)?

---

## Вопросы

1. Чем `@Retention(RUNTIME)` отличается от `@Retention(SOURCE)`? Приведите пример, когда `SOURCE` достаточно.
2. Зачем нужны use-site targets? Что произойдёт, если не указать target для аннотации на `val`-параметре конструктора?
3. Почему reflection медленнее прямого вызова? Что происходит «под капотом»?
4. Как compiler plugins (kotlinx.serialization, Compose) отличаются от annotation processors (kapt/KSP)? В чём преимущества каждого подхода?
5. Зачем нужны annotation processors (KSP/kapt), если многое можно сделать через runtime reflection? В каких случаях вы бы выбрали генерацию кода, а в каких — reflection?
6. Можно ли через reflection вызвать `private`-функцию? Если да — стоит ли так делать?
7. В чём ключевые отличия Java Reflection и Kotlin Reflection? Какие вещи Kotlin reflection знает о типах, а Java — нет?
8. Почему Kotlin-свойство не всегда соответствует Java-полю? Как это влияет на библиотеки/фреймворки на Java и на use-site targets?

---

## Итоги

После этой практики студент должен понимать:

1. **Аннотации** — метаданные для кода. Объявляются через `annotation class`. Ключевые мета-аннотации: `@Target`, `@Retention`.
2. **Отличия от Java**: синтаксис `annotation class` vs `@interface`, `KClass` vs `Class`, `vararg` vs массивы, обязательные use-site targets.
3. **Стандартные аннотации Kotlin**: `@Deprecated`, `@Suppress`, `@OptIn`, `@JvmStatic`, `@JvmOverloads`, `@JvmField`, `@Throws`, `@Volatile`, `@Transient`.
4. **Use-site targets** (`@field:`, `@get:`, `@param:` и др.) уточняют, к какому элементу в байткоде привязана аннотация.
5. **Аннотации обрабатываются** компилятором, compiler plugins, annotation processors (kapt/KSP) или через reflection.
6. **Reflection** (`kotlin-reflect`) позволяет исследовать классы, свойства, функции и аннотации в рантайме.
7. **`KClass`**, **`KFunction`**, **`KProperty`** — ключевые типы Kotlin reflection.
8. **Java vs Kotlin reflection**: `Class/Method/Field` vs `KClass/KFunction/KProperty`, свойства vs поля/геттеры, Kotlin metadata (nullability, имена параметров) и interop через `kotlin.reflect.jvm.*`.
9. **Применения reflection**: сериализация, DI, тестовые фреймворки, ORM, валидация.
10. **Цена reflection**: производительность, потеря type safety, проблемы с обфускацией. Альтернативы: compiler plugins, KSP, `reified`.
