# practice-04: системы сборки, Maven, Gradle

## План

1. Зачем нужны системы сборки
2. Maven — краткий обзор
3. Gradle — архитектура, Groovy DSL vs Kotlin DSL
4. Проект Gradle изнутри — структура, `build.gradle.kts`, `settings.gradle.kts`, wrapper
5. Таски — встроенные, кастомные, граф зависимостей
6. Плагины — `kotlin("jvm")`, `application`, сторонние
7. Репозитории — `mavenCentral()`, `google()`, `mavenLocal()`
8. Зависимости — конфигурации, транзитивность, связь с classpath
9. Convention plugins — `buildSrc`, переиспользование конфигурации
10. Публикация — `maven-publish`, GAV-координаты
11. Упражнения
12. Что важно вынести из практики
13. Идеи для устного обсуждения

---

## Зачем нужны системы сборки

На практике 3 мы вручную компилировали Kotlin-файлы через `kotlinc`, собирали JAR через `jar` и управляли classpath руками.

Это работает для одного файла. Но в реальном проекте:

- десятки и сотни файлов в разных пакетах;
- внешние библиотеки, у которых есть свои зависимости;
- нужно запускать тесты, генерировать документацию, публиковать артефакты;
- несколько разработчиков должны получать одинаковый результат сборки.

Вручную это не масштабируется. Системы сборки решают эти проблемы:

- **декларативно описывают** зависимости и шаги сборки;
- **автоматически скачивают** библиотеки из репозиториев;
- **кешируют** результаты — не пересобирают то, что не изменилось;
- **стандартизируют** структуру проекта.

Хороший вопрос студентам:

> Что именно делает IDEA, когда мы нажимаем кнопку `Run` в Gradle-проекте?

---

## Maven

Maven — старейшая из популярных систем сборки для JVM (2004). До сих пор широко используется в enterprise-проектах.

### Ключевые идеи Maven

- **Convention over configuration**: стандартная структура проекта — `src/main/kotlin`, `src/test/kotlin`, `target/`.
- **POM (Project Object Model)**: конфигурация в XML-файле `pom.xml`.
- **Жизненный цикл (lifecycle)**: фиксированная последовательность фаз — `validate → compile → test → package → verify → install → deploy`.

### Пример `pom.xml`

```xml
<project>
    <modelVersion>4.0.0</modelVersion>

    <groupId>com.example</groupId>
    <artifactId>my-app</artifactId>
    <version>1.0.0</version>

    <dependencies>
        <dependency>
            <groupId>org.jetbrains.kotlin</groupId>
            <artifactId>kotlin-stdlib</artifactId>
            <version>2.1.0</version>
        </dependency>
    </dependencies>
</project>
```

### Плюсы и минусы Maven

| Плюсы | Минусы |
|---|---|
| Стандартизированная структура | XML многословен |
| Огромная экосистема плагинов | Жёсткий жизненный цикл |
| Хорошая воспроизводимость сборки | Сложно делать нестандартные вещи |
| Широко используется в enterprise | Медленнее Gradle |

---

## Gradle

Gradle появился в 2012 году как ответ на ограничения Maven. Сейчас это основная система сборки для Android и Kotlin-проектов.

### Ключевые идеи Gradle

- **Гибкость**: конфигурация — это код (Groovy или Kotlin DSL), а не XML.
- **Инкрементальная сборка**: Gradle отслеживает входы и выходы тасков и пропускает те, что не изменились.
- **Build cache**: результаты тасков можно кешировать между машинами.
- **Граф тасков**: сборка — это направленный ациклический граф (DAG) тасков.

### Groovy DSL vs Kotlin DSL

Gradle поддерживает два языка для написания скриптов сборки:

| | Groovy DSL | Kotlin DSL |
|---|---|---|
| Файл | `build.gradle` | `build.gradle.kts` |
| Типизация | динамическая | статическая |
| Автодополнение в IDE | слабое | полноценное |
| Скорость конфигурации | чуть быстрее | чуть медленнее |
| Рекомендация JetBrains | — | ✓ |

В этом курсе используем **Kotlin DSL** (`build.gradle.kts`).

---

## Проект Gradle изнутри

### Структура каталогов

```
my-project/
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties   # версия Gradle
├── src/
│   ├── main/
│   │   └── kotlin/
│   │       └── Main.kt
│   └── test/
│       └── kotlin/
│           └── MainTest.kt
├── build.gradle.kts                    # конфигурация сборки
├── settings.gradle.kts                 # имя проекта, подпроекты
├── gradlew                             # wrapper-скрипт (Unix)
└── gradlew.bat                         # wrapper-скрипт (Windows)
```

### `settings.gradle.kts`

Определяет имя корневого проекта и список подпроектов (для многомодульных сборок).

```kotlin
rootProject.name = "my-project"
```

### `build.gradle.kts`

Основной файл конфигурации сборки.

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    application
}

group = "com.example"
version = "1.0.0"

repositories {
    mavenCentral()
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0")
    testImplementation(kotlin("test"))
}

application {
    mainClass = "MainKt"
}
```

### Gradle Wrapper

Wrapper — это скрипт (`gradlew` / `gradlew.bat`), который автоматически скачивает нужную версию Gradle, если её нет локально.

**Почему это важно:**

- все разработчики и CI используют одну и ту же версию Gradle;
- не нужно устанавливать Gradle глобально;
- версия зафиксирована в `gradle-wrapper.properties`.

```properties
# gradle/wrapper/gradle-wrapper.properties
distributionUrl=https\://services.gradle.org/distributions/gradle-8.13-bin.zip
```

Запускать сборку всегда через wrapper:

```bash
./gradlew build      # Unix/macOS
gradlew.bat build    # Windows
```

---

## Таски

Таск (task) — это единица работы в Gradle. Каждый шаг сборки — это таск.

### Встроенные таски

```bash
./gradlew tasks          # список всех доступных тасков
./gradlew build          # компиляция + тесты + сборка JAR
./gradlew compileKotlin  # только компиляция Kotlin
./gradlew test           # запуск тестов
./gradlew clean          # удаление build/
./gradlew run            # запуск приложения (плагин application)
./gradlew jar            # сборка JAR без зависимостей
```

### Граф зависимостей тасков

Таски могут зависеть друг от друга. Gradle строит DAG и выполняет таски в правильном порядке.

```
build
  └── assemble
        └── jar
              └── compileKotlin
  └── check
        └── test
              └── compileTestKotlin
                    └── compileKotlin
```

Посмотреть граф можно через плагин `task-tree`:

```bash
./gradlew build taskTree
```

### Кастомный таск

Таск можно объявить прямо в `build.gradle.kts`:

```kotlin
tasks.register("hello") {
    group = "demo"
    description = "Prints a greeting"
    doLast {
        println("Hello from Gradle task!")
    }
}
```

```bash
./gradlew hello
# > Task :hello
# Hello from Gradle task!
```

### Фазы конфигурации и выполнения

Важно понимать разницу:

- **Фаза конфигурации**: Gradle читает все `build.gradle.kts` и строит граф тасков. Код вне `doFirst`/`doLast` выполняется здесь.
- **Фаза выполнения**: Gradle запускает нужные таски. Код внутри `doFirst`/`doLast` выполняется здесь.

```kotlin
tasks.register("demo") {
    println("Это выполнится при КОНФИГУРАЦИИ")  // всегда, даже если таск не запускали
    doLast {
        println("Это выполнится при ЗАПУСКЕ таска")
    }
}
```

---

## Плагины

Плагин — это набор тасков, конфигураций и соглашений, которые добавляются в проект.

### Типы плагинов

- **Core plugins** — встроены в Gradle: `java`, `application`, `maven-publish`.
- **Community plugins** — публикуются на [plugins.gradle.org](https://plugins.gradle.org).

### Подключение плагинов

```kotlin
plugins {
    // Kotlin DSL-нотация для плагинов Kotlin
    kotlin("jvm") version "2.1.0"

    // Core plugin (без версии)
    application

    // Community plugin (с версией)
    id("org.jmailen.kotlinter") version "5.0.1"
}
```

### Часто используемые плагины

| Плагин | Что добавляет |
|---|---|
| `kotlin("jvm")` | компиляция Kotlin, kotlin-stdlib |
| `application` | таск `run`, создание дистрибутива |
| `java-library` | разделение `api` и `implementation` |
| `maven-publish` | публикация артефактов |
| `org.jmailen.kotlinter` | линтер и форматтер Kotlin-кода |

---

## Репозитории

Репозиторий — это хранилище артефактов (JAR-файлов и их метаданных), откуда Gradle скачивает зависимости.

```kotlin
repositories {
    mavenCentral()   // Maven Central — основной публичный репозиторий
    google()         // Google Maven — для Android-библиотек
    mavenLocal()     // ~/.m2/repository — локальный репозиторий Maven
}
```

### Как Gradle ищет зависимость

1. Проверяет локальный кеш Gradle (`~/.gradle/caches/`).
2. Если не найдено — обходит репозитории в порядке объявления.
3. Скачивает JAR и POM (метаданные с транзитивными зависимостями).
4. Кеширует результат.

### Локальный кеш vs `mavenLocal()`

| | Gradle cache | `mavenLocal()` |
|---|---|---|
| Путь | `~/.gradle/caches/` | `~/.m2/repository/` |
| Управление | автоматически | вручную через `publishToMavenLocal` |
| Использование | всегда | явно добавить в `repositories {}` |

---

## Зависимости

### Конфигурации зависимостей

Конфигурация определяет, когда и как зависимость используется:

```kotlin
dependencies {
    // Попадает в compile classpath и runtime classpath
    implementation("com.google.guava:guava:33.4.0-jre")

    // Попадает в compile classpath потребителей (для библиотек)
    api("org.jetbrains.kotlinx:kotlinx-coroutines-core:1.9.0")

    // Только compile classpath, не попадает в runtime
    compileOnly("org.projectlombok:lombok:1.18.36")

    // Только runtime classpath, не нужна при компиляции
    runtimeOnly("org.postgresql:postgresql:42.7.5")

    // Только для тестов
    testImplementation(kotlin("test"))
    testImplementation("io.mockk:mockk:1.13.17")
}
```

### `implementation` vs `api`

Это различие важно для **библиотек** (проектов, которые сами являются зависимостью):

- `implementation` — зависимость **не видна** потребителям библиотеки (не утекает в их compile classpath).
- `api` — зависимость **видна** потребителям (утекает в их compile classpath).

```
Проект A  →  Библиотека B  →  Библиотека C

Если B объявляет C через implementation:
  A не видит C при компиляции

Если B объявляет C через api:
  A видит C при компиляции
```

Правило: используйте `implementation` по умолчанию, `api` — только если типы из зависимости присутствуют в публичном API библиотеки.

### Транзитивные зависимости

Когда вы подключаете библиотеку, Gradle автоматически скачивает и её зависимости.

```bash
./gradlew dependencies --configuration runtimeClasspath
```

Посмотреть дерево зависимостей:

```
runtimeClasspath
\--- com.google.guava:guava:33.4.0-jre
     +--- com.google.guava:failureaccess:1.0.2
     +--- com.google.guava:listenablefuture:9999.0-empty-to-avoid-conflict-with-guava
     \--- com.google.errorprone:error_prone_annotations:2.28.0
```

### Связь с classpath из practice-03

| Конфигурация Gradle | Classpath из practice-03 |
|---|---|
| `implementation`, `api` | compile classpath + runtime classpath |
| `compileOnly` | только compile classpath |
| `runtimeOnly` | только runtime classpath |

---

## Convention Plugins

### Проблема: дублирование конфигурации

В многомодульном проекте каждый модуль имеет свой `build.gradle.kts`. Если в каждом нужно одинаково настроить Kotlin, линтер и тесты — конфигурация дублируется.

### Решение: `buildSrc`

`buildSrc` — специальный каталог, который Gradle компилирует перед основной сборкой. Код из него доступен во всех `build.gradle.kts`.

```
my-project/
├── buildSrc/
│   ├── build.gradle.kts
│   └── src/main/kotlin/
│       └── kotlin-conventions.gradle.kts   # convention plugin
├── module-a/
│   └── build.gradle.kts
├── module-b/
│   └── build.gradle.kts
└── settings.gradle.kts
```

`buildSrc/build.gradle.kts`:

```kotlin
plugins {
    `kotlin-dsl`
}

repositories {
    mavenCentral()
}
```

`buildSrc/src/main/kotlin/kotlin-conventions.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm")
    id("org.jmailen.kotlinter")
}

kotlin {
    jvmToolchain(21)
}

tasks.test {
    useJUnitPlatform()
}
```

Теперь в каждом модуле достаточно одной строки:

```kotlin
// module-a/build.gradle.kts
plugins {
    id("kotlin-conventions")
}
```

---

## Публикация

### GAV-координаты

Каждый артефакт в Maven-репозитории идентифицируется тремя координатами:

- **G**roupId — организация/проект: `com.example`
- **A**rtifactId — имя модуля: `my-library`
- **V**ersion — версия: `1.0.0`

В `build.gradle.kts`:

```kotlin
group = "com.example"
version = "1.0.0"
// artifactId = имя проекта из settings.gradle.kts
```

### Плагин `maven-publish`

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    `maven-publish`
}

group = "com.example"
version = "1.0.0"

publishing {
    publications {
        create<MavenPublication>("mavenKotlin") {
            from(components["kotlin"])
        }
    }
}
```

### Публикация в локальный репозиторий

```bash
./gradlew publishToMavenLocal
```

Артефакт появится в `~/.m2/repository/com/example/my-library/1.0.0/`.

После этого другой проект может подключить его через:

```kotlin
repositories {
    mavenLocal()
    mavenCentral()
}

dependencies {
    implementation("com.example:my-library:1.0.0")
}
```

---

## Упражнения

### Упражнение 1: создание Gradle-проекта с нуля

1. Создайте новый Kotlin-проект в IDEA с системой сборки Gradle (Kotlin DSL).
2. Откройте `settings.gradle.kts` — найдите имя проекта.
3. Откройте `build.gradle.kts` — найдите:
   - подключённые плагины;
   - блок `repositories`;
   - блок `dependencies`.
4. Запустите встроенные таски:

```bash
./gradlew tasks
./gradlew build
./gradlew clean
```

5. Найдите в каталоге `build/` скомпилированные `.class`-файлы и JAR.
6. Ответьте письменно:
   - зачем нужен `gradlew`, если Gradle можно установить глобально?
   - что произойдёт, если запустить `./gradlew build` повторно без изменений кода?

### Упражнение 2: подключение внешней библиотеки

1. Добавьте в `build.gradle.kts` зависимость на `kotlinx-serialization`:

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    kotlin("plugin.serialization") version "2.1.0"
}

dependencies {
    implementation("org.jetbrains.kotlinx:kotlinx-serialization-json:1.8.1")
}
```

2. Напишите простой класс и сериализуйте его в JSON:

```kotlin
import kotlinx.serialization.Serializable
import kotlinx.serialization.json.Json

@Serializable
data class User(val name: String, val age: Int)

fun main() {
    val user = User("Alice", 30)
    val json = Json.encodeToString(user)
    println(json)

    val decoded = Json.decodeFromString<User>(json)
    println(decoded)
}
```

3. Запустите `./gradlew run`.
4. Запустите:

```bash
./gradlew dependencies --configuration runtimeClasspath
```

Найдите транзитивные зависимости `kotlinx-serialization-json`.

5. Ответьте письменно:
   - чем `implementation` отличается от `runtimeOnly`?
   - почему `kotlin-stdlib` не нужно добавлять вручную?

### Упражнение 3: линтер kotlinter

`kotlinter` — плагин для проверки и автоматического форматирования Kotlin-кода по правилам [ktlint](https://pinterest.github.io/ktlint/).

1. Подключите плагин в `build.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    id("org.jmailen.kotlinter") version "5.0.1"
}
```

2. Намеренно нарушьте форматирование в `Main.kt` — например, уберите пробелы вокруг операторов или добавьте лишние пустые строки:

```kotlin
fun main(){
val x=42
println(x)
}
```

3. Запустите проверку:

```bash
./gradlew lintKotlin
```

Посмотрите на вывод — kotlinter покажет файл, строку и описание нарушения.

4. Запустите автоматическое исправление:

```bash
./gradlew formatKotlin
```

5. Откройте файл — убедитесь, что форматирование исправлено.
6. Запустите `lintKotlin` снова — убедитесь, что ошибок нет.
7. Ответьте письменно:
   - чем `lintKotlin` отличается от `formatKotlin`?
   - почему линтер полезно добавить в CI?
   - как сделать так, чтобы `build` падал при нарушениях форматирования?

### Упражнение 4: публикация в `mavenLocal`

1. Создайте новый Gradle-проект `my-library` с простой функцией:

```kotlin
// src/main/kotlin/Greeter.kt
object Greeter {
    fun greet(name: String): String = "Hello, $name!"
}
```

2. Настройте `build.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    `maven-publish`
}

group = "com.example"
version = "1.0.0"

publishing {
    publications {
        create<MavenPublication>("mavenKotlin") {
            from(components["kotlin"])
        }
    }
}
```

3. Опубликуйте в локальный репозиторий:

```bash
./gradlew publishToMavenLocal
```

4. Найдите артефакт в `~/.m2/repository/com/example/my-library/`.
5. Создайте второй проект `my-app` и подключите библиотеку:

```kotlin
repositories {
    mavenLocal()
    mavenCentral()
}

dependencies {
    implementation("com.example:my-library:1.0.0")
}
```

6. Вызовите `Greeter.greet("World")` из `my-app` и запустите.

### Упражнение 5: исследование графа тасков

1. Добавьте плагин `task-tree` в `build.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm") version "2.1.0"
    id("com.dorongold.task-tree") version "4.0.0"
}
```

2. Посмотрите граф зависимостей таска `build`:

```bash
./gradlew build taskTree
```

3. Посмотрите, какие таски будут запущены, без реального запуска:

```bash
./gradlew build --dry-run
```

4. Ответьте письменно:
   - какие таски входят в `build`?
   - в каком порядке они выполняются и почему?
   - что означает `UP-TO-DATE` рядом с именем таска?

### Упражнение 7: convention plugin в `buildSrc`

1. Создайте многомодульный проект:

```
my-multiproject/
├── buildSrc/
├── module-a/
├── module-b/
└── settings.gradle.kts
```

`settings.gradle.kts`:

```kotlin
rootProject.name = "my-multiproject"
include("module-a", "module-b")
```

2. Настройте `buildSrc`:

`buildSrc/build.gradle.kts`:

```kotlin
plugins {
    `kotlin-dsl`
}

repositories {
    mavenCentral()
}
```

3. Создайте convention plugin:

`buildSrc/src/main/kotlin/kotlin-conventions.gradle.kts`:

```kotlin
plugins {
    kotlin("jvm")
}

kotlin {
    jvmToolchain(21)
}

tasks.test {
    useJUnitPlatform()
}
```

4. Подключите convention plugin в обоих модулях:

```kotlin
// module-a/build.gradle.kts
plugins {
    id("kotlin-conventions")
}
```

5. Добавьте по одному классу в каждый модуль и запустите:

```bash
./gradlew build
```

6. Ответьте письменно:
   - зачем нужен `buildSrc`, если можно просто скопировать конфигурацию?
   - что произойдёт, если изменить convention plugin — нужно ли обновлять каждый модуль?

---

## Что важно вынести из практики

После этой практики студент должен понимать:

1. Системы сборки решают проблему масштабирования ручной компиляции.
2. Maven — декларативный, XML-based, с фиксированным жизненным циклом.
3. Gradle — гибкий, код-based (Kotlin DSL), с инкрементальной сборкой.
4. Gradle Wrapper фиксирует версию Gradle и избавляет от необходимости глобальной установки.
5. Таск — единица работы; Gradle строит DAG тасков и выполняет их в правильном порядке.
6. Плагины добавляют таски и конфигурации; `kotlin("jvm")` — основной плагин для Kotlin-проектов.
7. Конфигурации зависимостей (`implementation`, `api`, `compileOnly`, `runtimeOnly`) определяют, в какой classpath попадает зависимость.
8. Convention plugins в `buildSrc` позволяют переиспользовать конфигурацию сборки.
9. `maven-publish` позволяет публиковать артефакты; GAV-координаты — стандартный способ идентификации.
10. Линтер (`kotlinter`) помогает поддерживать единый стиль кода и легко интегрируется в CI.

---

## Идеи для устного обсуждения

- Чем Gradle лучше Maven? В каких случаях Maven всё ещё предпочтительнее?
- Что такое инкрементальная сборка и почему она важна?
- Почему `implementation` предпочтительнее `api` в большинстве случаев?
- Что произойдёт, если добавить зависимость в `compileOnly`, но забыть добавить её в `runtimeOnly`?
- Зачем нужен `mavenLocal()`? Какие риски он несёт?
- Как Gradle решает конфликты версий транзитивных зависимостей?
- Почему код вне `doLast` в таске выполняется при конфигурации, а не при запуске?
- Как линтер помогает в командной разработке?

