# practice-03: CLI, JVM toolchain, bytecode, JAR, classpath

## План

1. Зачем знать CLI, если есть IDEA
2. JDK, JVM, `javac`, `kotlinc`
3. Установка и проверка JDK
4. Компиляция и запуск Kotlin из командной строки
5. Что появляется после компиляции
6. Почему класс называется `MainKt`
7. Компиляция и запуск Java из командной строки
8. Разбор `.class`-файлов и байткода через `javap`
9. JAR-файлы
10. Classpath: compile-time vs runtime
11. Java + Kotlin вместе
12. Типичные ошибки
13. Упражнения
14. Что важно вынести из практики
15. Идеи для устного обсуждения

---

## Зачем знать CLI, если есть IDEA

IDEA сильно упрощает жизнь: создаёт проект, запускает сборку, подставляет classpath, находит `main`, показывает байткод.

Но в реальной жизни важно понимать, **что происходит под капотом**:

- как код превращается в `.class`;
- что именно запускает JVM;
- откуда берутся зависимости;
- почему код иногда компилируется, но не запускается;
- чем compile classpath отличается от runtime classpath.

Практический смысл темы:

- легче разбираться с Gradle / Maven;
- легче читать ошибки CI;
- легче понимать, что именно делает IDE;
- проще дебажить проблемы со сборкой и запуском;
- легче работать на сервере, в Docker и в CI/CD без GUI.

Хороший вопрос студентам в начале практики:

> Что именно делает IDEA, когда мы нажимаем кнопку `Run`?

---

## JDK, JVM, `javac`, `kotlinc`

На JVM у нас есть несколько разных сущностей, и их полезно не путать.

### JDK

**JDK** (Java Development Kit) — это набор инструментов для разработки:

- `java` — запуск JVM-программ;
- `javac` — компилятор Java;
- `javap` — просмотр структуры `.class`;
- `jar` — работа с JAR-архивами;
- стандартные библиотеки Java.

### JVM

**JVM** (Java Virtual Machine) — виртуальная машина, которая исполняет байткод `.class`.

То есть:

- `javac` и `kotlinc` **компилируют** код;
- `java` **запускает** получившийся байткод.

### `javac`

`javac` — компилятор **Java**.

Пример:

```text
javac Main.java
```

На выходе обычно получаем `.class`.

### `kotlinc`

`kotlinc` — компилятор **Kotlin**.

Для JVM он тоже компилирует код в `.class`.

Пример:

```text
kotlinc Main.kt
```

Важно: Kotlin на JVM — это не «отдельная виртуальная машина», а язык, который тоже компилируется в JVM-байткод.

---

## Установка и проверка JDK

### Проверка текущего JDK

```bash
java -version
javac -version
```

Если команды не найдены — JDK не установлен или не добавлен в `PATH`.

### Где скачать JDK

Рекомендация: **Azul Zulu 21**.

- https://www.azul.com/downloads/?version=java-21-lts

Альтернативы: Oracle JDK, Amazon Corretto, Eclipse Temurin.

### `JAVA_HOME` и `PATH`

`JAVA_HOME` — переменная окружения, указывающая на корень JDK.

```bash
# macOS / Linux: добавить в ~/.bashrc или ~/.zshrc
export JAVA_HOME=/path/to/jdk-21
export PATH="$JAVA_HOME/bin:$PATH"
```

```powershell
# Windows (PowerShell)
$env:JAVA_HOME = "C:\path\to\jdk-21"
$env:PATH = "$env:JAVA_HOME\bin;$env:PATH"
```

Проверка:

```bash
echo $JAVA_HOME
java -version
```

---

## Компиляция и запуск Kotlin из командной строки

### Установка `kotlinc`

Варианты:

- **SDKMAN** (macOS / Linux):

```bash
sdk install kotlin
```

- **Homebrew** (macOS):

```bash
brew install kotlin
```

- **Вручную**: скачать с https://github.com/JetBrains/kotlin/releases и добавить `bin/` в `PATH`.

Проверка:

```bash
kotlinc -version
```

### Hello World: компиляция и запуск

Создадим файл `Hello.kt`:

```kotlin
fun main() {
    println("Hello from CLI!")
}
```

#### Вариант 1: компиляция в `.class` и запуск через `kotlin`

```bash
kotlinc Hello.kt -d out
```

В каталоге `out/` появится файл `HelloKt.class`.

Запуск через `kotlin` runner (он автоматически добавляет stdlib в classpath):

```bash
kotlin -cp out HelloKt
```

#### Вариант 2: запуск через `java`

Если запускать через `java`, нужно вручную добавить Kotlin stdlib в classpath:

```bash
java -cp "out:$KOTLIN_HOME/lib/kotlin-stdlib.jar" HelloKt
```

> **Обратите внимание**: без Kotlin runtime в classpath будет `NoClassDefFoundError` на классы из `kotlin.*`.

#### Вариант 3: компиляция в самодостаточный JAR

Можно собрать fat JAR вручную, включив в него Kotlin stdlib:

```bash
# 1. Компилируем
kotlinc Hello.kt -d out

# 2. Создаём манифест
echo "Main-Class: HelloKt" > manifest.txt

# 3. Собираем JAR (добавляя kotlin-stdlib)
jar cfm hello.jar manifest.txt -C out .
```

Запуск:

```bash
java -cp "hello.jar:$KOTLIN_HOME/lib/kotlin-stdlib.jar" HelloKt
```

> В учебных целях проще всего использовать `kotlin -cp out HelloKt` — он берёт stdlib на себя.

---

## Что появляется после компиляции

Важно, чтобы студенты увидели: результат компиляции — это не абстракция, а конкретные файлы.

### Пример

Пусть есть такой файл:

```kotlin
data class User(val name: String, val age: Int)

fun greet(user: User) {
    println("Hello, ${user.name}")
}

fun main() {
    greet(User("Alice", 20))
}
```

Компиляция:

```bash
kotlinc Main.kt -d out
```

После компиляции можно увидеть несколько `.class`:

- `User.class`
- `MainKt.class`

Если есть лямбды, вложенные классы или другие конструкции, могут появляться и дополнительные сгенерированные классы.

### Что полезно проговорить

- `data class` превращается в обычный JVM-класс;
- top-level функции попадают в сгенерированный класс по имени файла;
- Kotlin-специфические конструкции в итоге должны быть представлены в терминах JVM.

Это хорошее место, чтобы ещё раз подчеркнуть:

> Kotlin на JVM компилируется в обычный JVM-байткод.

---

## Почему класс называется `MainKt`

Это один из самых важных моментов в теме CLI.

Если в Kotlin есть **top-level функция**:

```kotlin
fun main() {
    println("hi")
}
```

то JVM не умеет хранить «функцию просто в файле». Ей нужен класс.

Поэтому Kotlin генерирует класс по имени файла:

- `Main.kt` → `MainKt.class`
- `Utils.kt` → `UtilsKt.class`
- `App.kt` → `AppKt.class`

Именно поэтому при запуске Kotlin-кода из CLI часто фигурирует имя `MainKt` или `HelloKt`.

### Что важно заметить

- top-level функция `main` не исчезает, а становится **статическим методом** внутри сгенерированного класса;
- имя класса зависит от имени файла;
- это же объясняет, почему Java вызывает top-level Kotlin-функции через `FileNameKt`.

Мини-пример:

```kotlin
fun sum(a: Int, b: Int): Int {
    return a + b
}
```

После компиляции эта функция окажется внутри класса `FileNameKt` как обычный JVM-метод.

---

## Компиляция и запуск Java из командной строки

Для сравнения — тот же процесс на Java.

Создадим файл `Hello.java`:

```java
public class Hello {
    public static void main(String[] args) {
        System.out.println("Hello from Java CLI!");
    }
}
```

Компиляция:

```bash
javac Hello.java
```

Результат: файл `Hello.class` в текущем каталоге.

Запуск:

```bash
java Hello
```

> **Отличие от Kotlin**: Java не требует отдельного `kotlin-stdlib`, потому что работает только со стандартной Java-платформой.

---

## Разбор `.class`-файлов и байткода через `javap`

### Что такое `.class`

`.class` — скомпилированный байткод JVM. Это бинарный файл, который JVM умеет загружать и исполнять.

Один `.class` обычно соответствует одному классу.

### `javap` — дизассемблер байткода

`javap` — утилита из JDK для просмотра содержимого `.class`-файлов.

#### Базовый вызов

```bash
javap HelloKt.class
```

Покажет:

- имя класса;
- модификаторы;
- публичные методы.

#### С байткодом (`-c`)

```bash
javap -c HelloKt.class
```

Покажет JVM-инструкции каждого метода.

#### С приватными членами (`-p`)

```bash
javap -c -p HelloKt.class
```

### Пример: разбор Kotlin null-safety в байткоде

Напишем файл `NullDemo.kt`:

```kotlin
fun lenOrZero(s: String?): Int {
    return s?.length ?: 0
}

fun forceLen(s: String?): Int {
    return s!!.length
}

fun main() {
    println(lenOrZero(null))      // 0
    println(lenOrZero("hello"))  // 5
    println(forceLen("hello"))   // 5
}
```

Компиляция:

```bash
kotlinc NullDemo.kt -d out
```

Разбор:

```bash
javap -c out/NullDemoKt.class
```

### Что искать в байткоде

Для `lenOrZero` (safe-call + elvis):

- `IFNULL` / `IFNONNULL` — ветвление по `null`;
- два пути: вызов `String.length()` или возврат `0`.

Для `forceLen` (`!!`):

- вызов `kotlin/jvm/internal/Intrinsics.checkNotNull(...)` или аналогичная проверка;
- если значение `null`, будет брошено исключение.

> Это те же проверки, которые уже можно было видеть в IDEA через Show Kotlin Bytecode, но теперь — из командной строки.

### Пример: `data class` в байткоде

```kotlin
data class Point(val x: Int, val y: Int)
```

```bash
kotlinc Point.kt -d out
javap -c -p out/Point.class
```

Обратите внимание на сгенерированные методы:

- `equals(Object)`
- `hashCode()`
- `toString()`
- `copy(int, int)`
- `component1()`, `component2()`

### Идея для обсуждения со студентами

Спросить:

- почему функция не существует «сама по себе»;
- почему JVM требует класс;
- как это связано с тем, что Kotlin работает поверх JVM.

---

## JAR-файлы

### Что такое JAR

JAR (Java ARchive) — это **ZIP-архив**, содержащий:

- `.class`-файлы;
- `META-INF/MANIFEST.MF` — манифест с метаданными;
- ресурсы: конфиги, тексты, картинки и т.д.

Важно различать несколько ситуаций.

### Обычный JAR

Обычный JAR может просто содержать классы.

Сам по себе факт наличия JAR не означает, что его можно запустить через `java -jar`.

### Запускаемый JAR

Чтобы JAR можно было запустить командой:

```bash
java -jar app.jar
```

обычно нужен:

- `Main-Class` в manifest;
- все необходимые зависимости в доступном runtime.

### Создание JAR вручную

```bash
# 1. Компилируем
kotlinc Hello.kt -d out

# 2. Создаём манифест
echo "Main-Class: HelloKt" > manifest.txt

# 3. Собираем JAR
jar cfm hello.jar manifest.txt -C out .
```

Флаги `jar`:

- `c` — create;
- `f` — file;
- `m` — manifest;
- `-C out .` — перейти в каталог `out` и добавить все файлы.

### Просмотр содержимого JAR

```bash
jar tf hello.jar
```

Пример вывода:

```text
META-INF/
META-INF/MANIFEST.MF
HelloKt.class
```

### Почему `kotlin` runner удобнее для учёбы

Команда `kotlin -cp out HelloKt` делает учебный сценарий максимально простым:

- не нужно собирать JAR;
- не нужно вручную добавлять `kotlin-stdlib` в classpath;
- достаточно скомпилировать код и запустить.

> Если вы хотите распространять программу как самодостаточный JAR, нужно включить все зависимости (в том числе `kotlin-stdlib`) в архив и прописать `Main-Class` в `MANIFEST.MF`.

> Если в JAR нет `Main-Class`, то при `java -jar` можно получить ошибку `no main manifest attribute`.

---

## Classpath: compile-time vs runtime

Это, пожалуй, главная идея всей практики.

### Что такое classpath

**Classpath** — список путей (каталоги и JAR-файлы), в которых компилятор и JVM ищут классы.

Аналогия: classpath — это `PATH` для классов.

> Разделитель путей: `:` на macOS/Linux и `;` на Windows.

### Compile classpath

Classpath **при компиляции** — набор классов, которые нужны компилятору для проверки типов и разрешения зависимостей.

```bash
kotlinc App.kt -cp lib/utils.jar -d out
```

Если зависимости нет в compile classpath, код часто просто не скомпилируется:

```text
error: unresolved reference: SomeUtilClass
```

### Runtime classpath

Classpath **при запуске** — набор классов, которые нужны JVM для выполнения программы.

```bash
kotlin -cp "out:lib/utils.jar" AppKt
# или через java (тогда нужен stdlib):
java -cp "out:lib/utils.jar:$KOTLIN_HOME/lib/kotlin-stdlib.jar" AppKt
```

Если зависимости нет в runtime classpath, код уже может быть успешно скомпилирован, но программа упадёт при запуске:

```text
Exception in thread "main" java.lang.NoClassDefFoundError: com/example/SomeUtilClass
```

### Интуитивная разница

Коротко:

- compile classpath — «чтобы собрать»;
- runtime classpath — «чтобы выполнить».

### Пример: проект из двух файлов

Файл `Utils.kt`:

```text
package util

fun greet(name: String): String = "Hello, $name!"
```

Файл `App.kt`:

```text
import util.greet

fun main() {
    println(greet("CLI"))
}
```

Пошаговая сборка:

```bash
# 1. Компилируем Utils
kotlinc Utils.kt -d out

# 2. Компилируем App, указывая compile classpath
kotlinc App.kt -cp out -d out

# 3. Запускаем (kotlin runner добавляет stdlib автоматически)
kotlin -cp out AppKt
# или через java (тогда нужно добавить kotlin-stdlib вручную):
# java -cp "out:$KOTLIN_HOME/lib/kotlin-stdlib.jar" AppKt
```

### Пример: подключение внешней библиотеки

Допустим, у нас есть `gson-2.11.0.jar`:

```text
import com.google.gson.Gson

data class User(val name: String, val age: Int)

fun main() {
    val json = Gson().toJson(User("Alice", 20))
    println(json) // {"name":"Alice","age":20}
}
```

Компиляция и запуск:

```bash
# Compile classpath: нужен Gson для проверки типов
kotlinc App.kt -cp lib/gson-2.11.0.jar -d out

# Runtime classpath: нужен Gson для выполнения
kotlin -cp "out:lib/gson-2.11.0.jar" AppKt
# или через java (тогда kotlin-stdlib тоже нужен):
# java -cp "out:lib/gson-2.11.0.jar:$KOTLIN_HOME/lib/kotlin-stdlib.jar" AppKt
```

### Типичные ошибки classpath

| Ошибка | Когда возникает | Причина |
|--------|----------------|---------|
| `unresolved reference` | Компиляция | Класс не найден в compile classpath |
| `ClassNotFoundException` | Рантайм | Класс не найден в runtime classpath |
| `NoClassDefFoundError` | Рантайм | Класс был при компиляции, но отсутствует при запуске |
| `NoSuchMethodError` | Рантайм | Версия класса в runtime отличается от compile-time |

### Резюме: compile vs runtime

| | Compile classpath | Runtime classpath |
|-|-------------------|-------------------|
| **Когда используется** | При компиляции (`kotlinc` / `javac`) | При запуске (`java`) |
| **Кем используется** | Компилятором | JVM |
| **Что проверяется** | Типы, сигнатуры, импорты | Наличие классов и методов |
| **Ошибка при отсутствии** | Ошибка компиляции | `ClassNotFoundException` / `NoClassDefFoundError` |

---

## Java + Kotlin вместе

Это логичное продолжение темы: и Java, и Kotlin на JVM в итоге живут в одном мире `.class`-файлов.

### Kotlin вызывает Java

#### `Greeter.java`

```java
public class Greeter {
    public static String greet(String name) {
        return "Hello, " + name;
    }
}
```

#### `Main.kt`

```kotlin
fun main() {
    println(Greeter.greet("Kotlin"))
}
```

Что здесь полезно обсудить:

- Kotlin умеет вызывать Java-код;
- Java-класс должен попасть в compile classpath;
- при запуске он тоже должен быть доступен JVM.

### Java вызывает Kotlin

#### `Utils.kt`

```kotlin
fun answer(): Int {
    return 42
}
```

#### `Main.java`

```java
public class Main {
    public static void main(String[] args) {
        System.out.println(UtilsKt.answer());
    }
}
```

Здесь особенно полезно обратить внимание студентов:

- top-level функция `answer` лежит не в `Utils`, а в `UtilsKt`;
- это следствие того, как Kotlin отображает top-level объявления на JVM.

---

## Типичные ошибки

### `ClassNotFoundException`

Обычно означает, что JVM не нашла класс по имени во время загрузки.

Типичные причины:

- забыли нужный JAR в runtime classpath;
- ошиблись в имени класса;
- запускаем не из той директории или не с тем `-cp`.

### `NoClassDefFoundError`

Похоже на предыдущую проблему, но часто всплывает, когда класс был доступен на этапе компиляции, а на runtime — нет.

Для учебной практики это хороший маркер проблемы с runtime classpath.

### Неправильный `Main-Class`

Если в JAR неверно указан entry point, `java -jar ...` не сможет корректно стартовать программу.

### Забыли Kotlin runtime

Если собрать JAR или `.class` без нужного Kotlin runtime, можно получить ошибки запуска.

Это особенно полезно проговорить после примера с ручной сборкой JAR.

### Неправильная версия JDK / target

Иногда код компилируется в окружении одной версии, а запускается в другой. Это уже более продвинутый случай, но студентам полезно знать, что такое тоже бывает.

---

## Упражнения

### Упражнение 1: Hello World из CLI

1. Создайте файл `Main.kt` с функцией `main`, которая печатает `"Hello, CLI!"`.
2. Скомпилируйте его:

```bash
kotlinc Main.kt -d out
```

3. Запустите через `kotlin` runner:

```bash
kotlin -cp out MainKt
```

4. Убедитесь, что вывод — `Hello, CLI!`.
5. Попробуйте запустить через `java` без `kotlin-stdlib` в classpath — какая ошибка?

### Упражнение 2: разбор байткода

1. Создайте файл `Demo.kt`:

```kotlin
data class Student(val name: String, val grade: Int)

fun bestStudent(students: List<Student>): Student? {
    return students.maxByOrNull { it.grade }
}

fun main() {
    val students = listOf(Student("Alice", 5), Student("Bob", 3))
    val best = bestStudent(students) ?: error("No students")
    println("Best: ${best.name}")
}
```

2. Скомпилируйте:

```bash
kotlinc Demo.kt -d out
```

3. Посмотрите содержимое каталога `out/`.
4. Запустите:

```bash
javap -c -p out/Student.class
```

Найдите:

- сгенерированные `equals`, `hashCode`, `toString`;
- методы `component1()`, `component2()`.

5. Запустите:

```bash
javap -c out/DemoKt.class
```

Найдите:

- вызов `maxByOrNull`;
- проверку на `null` в логике `?: error(...)`.

### Упражнение 3: multi-file проект

1. Создайте два файла.

`MathUtils.kt`:

```text
package math

fun factorial(n: Int): Long {
    require(n >= 0) { "n must be >= 0" }
    var result = 1L
    for (i in 2..n) result *= i
    return result
}
```

`App.kt`:

```text
import math.factorial

fun main() {
    print("Enter n: ")
    val n = readln().toInt()
    println("$n! = ${factorial(n)}")
}
```

2. Скомпилируйте оба файла:

```bash
kotlinc MathUtils.kt -d out
kotlinc App.kt -cp out -d out
```

3. Запустите с правильным classpath.
4. Запустите через `kotlin -cp out AppKt`.
5. Попробуйте запустить через `java -cp out AppKt` (без `kotlin-stdlib`) — посмотрите на ошибку.
6. Исправьте classpath для `java` и запустите снова.

### Упражнение 4: диагностика `ClassNotFoundException`

1. Скомпилируйте упражнение 3 как обычно.
2. Удалите из `out/` файл `math/MathUtilsKt.class`.
3. Запустите программу. Какая ошибка? Почему?
4. Верните файл на место и запустите снова.

### Упражнение 5: создание JAR вручную

1. Скомпилируйте multi-file проект из упражнения 3 в каталог `out/`.
2. Создайте файл `manifest.txt` с содержимым:

```text
Main-Class: AppKt
```

3. Соберите JAR:

```bash
jar cfm app.jar manifest.txt -C out .
```

4. Посмотрите содержимое:

```bash
jar tf app.jar
```

5. Запустите:

```bash
java -jar app.jar
```

Работает ли? Если нет — почему? Подсказка: `kotlin-stdlib`.

6. Добавьте `kotlin-stdlib.jar` в classpath при запуске и проверьте.

### Упражнение 6: Java вызывает Kotlin top-level функцию

1. Создайте файл `Utils.kt`:

```text
fun answer(): Int {
    return 42
}
```

2. Подумайте, как Java-код должен обратиться к этой функции.
3. Создайте `Main.java` и вызовите функцию из Kotlin.
4. Ответьте письменно:
    - почему используется `UtilsKt`, а не `Utils`;
    - от чего зависит имя класса;
    - что изменится, если переименовать файл.

### Упражнение 7: compile classpath есть, runtime classpath нет

Рассмотрите код:

```kotlin
fun main() {
    val result = ExternalUtil.normalize("test")
    println(result)
}
```

Ответьте на вопросы:

- на каком этапе возникнет проблема, если библиотека есть только при компиляции;
- какая ошибка наиболее вероятна;
- почему код мог успешно скомпилироваться.

---

## Что важно вынести из практики

После этой практики студент должен понимать:

1. Kotlin на JVM компилируется в `.class`.
2. JVM запускает байткод, а не `.kt`-файлы.
3. Top-level функции в Kotlin превращаются в методы класса `FileNameKt`.
4. `javap` помогает посмотреть, что реально сгенерировал компилятор.
5. JAR — это контейнер с классами и метаданными.
6. Compile classpath и runtime classpath — это разные вещи.
7. Ошибки сборки и ошибки запуска очень часто связаны именно с classpath.
8. Java и Kotlin на JVM хорошо взаимодействуют, потому что в итоге оба языка работают через `.class`.

---

## Идеи для устного обсуждения

- Почему `Main.kt` превращается в `MainKt.class`?
- Почему код может компилироваться, но не запускаться?
- Чем отличается `java -jar` от запуска через `-cp`?
- Чем запуск через `kotlin` runner отличается от запуска через `java`?
- Почему знание CLI помогает, даже если мы обычно работаем через IDEA?
- Какие действия из этого файла IDE делает автоматически?
