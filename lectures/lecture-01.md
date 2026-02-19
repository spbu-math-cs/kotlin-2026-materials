---

# lecture-01: history & basic syntax

---
layout: two-cols-header
---

# Преподаватели

::left::

## Сергей Поспелов

- СП СПбГУ в 2023
- ПОВС ИТМО в 2025
- Veai
- t.me/@cepryh9
- лекции, практика Б10

## Илья Муромцев

- ИТМО КТ в 2012
- Veai
- t.me/@ilya_muromtsev
- практика Б09

::right::

## Артем Петров

- Yandex
- t.me/@artpetroff
- практика Б11

---
layout: image-right
image: ./imgs/frame.png
---

# Организационные моменты

- репозиторий: https://github.com/spbu-math-cs/kotlin-2026-materials
- чат: https://t.me/+DpwEEhUdRr01Yjli

---

# Структура

- 8 лекций каждые две недели
- 15 практик
  - углубление материала
  - live-coding
- 8 домашек с допбаллами
- проект
- тесты на практиках после лекций

---

# Оценивание

## Домашки

- выполняются на GitHub Classrooms
- выдаются раз в одну/две недели
- время на выполнение -- две недели
- могут быть бонусные баллы
- после дедлайна оцениваются в 50%
- автоматическая проверка + ревью
- баллы за домашку `hw = student_points / hw_points`
  - `hw_points` -- сумма обязательных баллов за домашки
  - `student_points` -- сумма баллов за домашки студента с учетом бонусных баллов
- плагиат/ЛЛМка -- 0 баллов

---

## Проект

- начинается во второй части курса
- критериии оценивания появятся позже TODO

## Тесты

- на каждой лекции проводится письменный тест
- проверяются знания из предыдущих лекций и практик
- 5 минут

---

## Итоговая оценка

```kotlin
fun getGrade(hw: Double, project: Double, tests: Double): String {
    val total = min(hw * 0.4 + project * 0.4 + tests * 0.2, 1.0)

    val grade = when {
        hw < 0.5 || project < 0.5 -> "F"
        total in 0.9..1.0 -> "A"
        total in 0.8..<0.9 -> "B"
        total in 0.7..<0.8 -> "C"
        total in 0.6..<0.7 -> "D"
        total in 0.5..<0.6 -> "E"
        total in 0.0..<0.5 -> "F"
        else -> error("Unexpected grade: $total")
    }

    return grade
}
```

---

# Что нужно?

- Аккаунт в GitHub - будем работать в классруме
- Установить Java SDK 21
- Установить IDE
  - macOS (ARM): https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1-aarch64.dmg
  - macOS (Intel) https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1.dmg
  - Windows: https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1.exe
  - Linux: https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1.tar.gz
- VSCode (не рекомендуется)

---

# Kotlin: background языка

## Что такое Kotlin

- Современный статически типизированный язык
- Основная платформа — JVM
- Полная совместимость с Java
- Разрабатывается JetBrains

---

## Андрей Бреслав

- Создатель языка Котлин
- Работал над языком с 2010 года по 2020

## Роман Елизаров

- С 2020 по 2023

### Михаил Зареченский

- C 2023 по настоящее время

---

## История: причины появления

- Java:
  - verbose
  - null-pointer exceptions
  - медленное развитие языка
- Нужен был язык:
  - современный, лаконичный
  - меньше бойлерплейта
  - с полной JVM-совместимостью
  - без необходимости переписывать экосистему

---

## История: ключевые вехи

- 2010 — начало разработки
- 2011 — публичный анонс
- 2016 — Kotlin 1.0 (обещание обратной совместимости)
- 2017 — Google объявляет Kotlin официальным языком Android
- 2019 — Kotlin-first для Android
- 2024 - Kotlin 2.0

---

## Популярность

- Один из самых используемых JVM-языков
- Де-факто стандарт для Android
- Активное использование в backend
- Большое коммьюнити и документация

---

## Применимость: Android

- Официально рекомендован Google
- Полная совместимость с Java-кодом
- Coroutines вместо callback’ов

---

## Применимость: backend

- JVM-экосистема
  - Spring
  - Hibernate
  - Kafka
- Ktor
  - асинхронный
  - coroutine-based

---

## Применимость: JetBrains и DSL

- Продукты JetBrains пишутся на Kotlin
- Kotlin используется для:
  - Gradle Kotlin DSL
  - конфигурационных DSL
  - внутренних DSL в компаниях

---

## Применимость: domain-specific библиотеки

- Язык хорошо подходит для DSL:
  - extension-функции
  - type-safe builders
- Примеры:
  - Gradle Kotlin DSL
  - kotlinx.serialization
  - kotlinx.html

---

## Основные отличия от Java

- Null-safety на уровне типов
- `if`, `when` — выражения
- Data-классы
- Extension-функции
- Нет checked exceptions
- Kotlin Coroutines
- Значительно меньше boilerplate-кода
- Меньше времени на код, больше времени на подумать

---

## Экосистема JVM

- JVM:
  - использует GC JVM (G1, ZGC и др.)
- Kotlin/Native:
  - собственный GC
- Управление памятью:
  - полностью автоматическое
  - без ручного контроля

---

## Типизация

- Статическая строгая типизация
- Type inference:
  - типы можно не писать
  - определены на этапе компиляции
- Smart-cast после проверок типов

---

## Несколько парадигм

- объектно-ориентированный
- функциональный

---

## Синтаксический сахар

- Data classes
- Named и default аргументы
- Destructuring
- Extension-функции
- Читаемый и компактный код

---

## Null-safety

- `T` и `T?` — разные типы
- Компилятор запрещает unsafe-доступ
- Большинство NPE устраняется на этапе компиляции

---

# Почему Kotlin в 2026 году

## Актуальность в индустрии

- Основной язык Android-разработки
- Активно используется в backend (Spring)
- Kotlin Multiplatform:
  - shared code для JVM / Android / iOS / Web
- Современная модель типов
- Хорошая работа с асинхронностью
- JVM-экосистема
- Легко перейти с Java
- Востребован на рынке
- Полезно для понимания общих концептов (архитектура, async, типы)

---

# Базовый синтаксис

## Hello world

```kt
fun main(args: Array<String>) {
  println("Hello, world!")
}

fun main() {
  println("Hello, world!")
}
```

- входная точка Kotlin-приложения - это main top-level функция
- принимает переменное количество String аргументов (можно опустить)
- `;` можно не писать, ура!

---

## Переменные

```kt
val a: Int = 1	// Немедленное присвоение

var b = 2		// 'Int' тип вычислен автоматически
b = a 		// Переназначение  'var' это ок

val c: Int		// Тип нужно указать, если не было инициализации
c = 3			// Отложенное присвоение
a = 4			// CE: val не может быть переприсвоен
```

- `val`/`var` -- изменяемые/неизменяемы переменные
- тип может быть вычислен по правой части
- присвоение может быть отложено

---

## Переменные

```kt
const val NAME = "Kotlin"	// вычисляется в compile-time
val nameLowered = NAME.lowercase()	 // не может быть вычислено в compile-time
```

- `const val` -- compile-time константа
- только для примитивных типов + `String`
- для`const val` используйте uppercase


---

# Типы

## Базовые типы

- Все типы — **reference-типы** (нет примитивов как в Java)
- На JVM примитивы используются под капотом (boxing/unboxing)
- Основные типы:
  - `Int`, `Long`, `Short`, `Byte`
  - `Float`, `Double`
  - `Boolean`
  - `Char`
  - `String`

---

## Числовые типы

```kt
val i: Int = 42
val l: Long = 42L
val d: Double = 3.14
val f: Float = 3.14f
```

- нет неявных преобразований

```
val x: Int = 10
val y: Long = x.toLong()
```

---

# Функции

```kt
fun sum(a: Int, b: Int): Int {
	return a + b
}

fun mul(a: Int, b: Int) = a * b

fun printMul(a: Int, b: Int): Unit {
	println(mul(a, b))
}

fun printMul1(a: Int = 1, b: Int) {
	println(mul(a, b))
}

fun printMul2(a: Int, b: Int = 1) = println(mul(a, b))
```

---

# Ветвления

```kt
fun maxOf(a: Int, b: Int): Int {
	if (a > b) {
		return a
	} else {
		return b
	}
}
```

можно сократить до

```kt
fun maxOf(a: Int, b: Int) =
  if (a > b) {
    a
  } else {
    b
  }
```

или до

```kt
fun maxOf(a: Int, b: Int) = if (a > b) a else b
```

---

# when

```kt
when (x) {
    1 -> print("x == 1")
    in 2..4 -> print("x in [2..4]")
    else -> {
        print("x < 1 || x > 4")
    }
}
```

или такой синтаксис

```kt
fun `как было раньше`(travaZelenee: Boolean, neboVyshe: Boolean) =
  when {
    travaZelenee && neboVyshe -> "сильно лучше"
    travaZelenee || neboVyshe -> "лучше"
    else -> "так же"
  }
```

- в случае, если разобраны не все ветки, компилятор выдаст ошибку
- если все варианты рассмотрены, можно не писать else

---

# `and` vs `&&`

```
if (a && b) { ... }     VS     if (a and b) { ... }     
if (a || b) { ... }     VS     if (a or b) { ... } 
```

- `&&` - short-circuit вычисление, `and` - нет.
- `||`, `or` -- аналогично

---

# Циклы

```kt
val items = listOf("apple", "banana", "kiwifruit")

for (item in items) {
    println(item)
}

for (index in items.indices) {
    println("item at $index is ${items[index]}")
}

for ((index, item) in items.withIndex()) {
    println("item at $index is $item")
}
```

---

# Циклы

```kt
val items = listOf("apple", "banana", "kiwifruit")

var index = 0
while (index < items.size) {
    println("item at $index is ${items[index]}")
    index++
}

var toComplete: Boolean
do {
    // что-то
    toComplete = //
} while(toComplete)

```

- можно инициализировать переменную внутри

---

# Циклы


```kt
myLabel@ for (item in items) {
    for (anotherItem in otherItems) {
        if (condition) break@myLabel
        else continue@myLabel
    }
}
```

- Есть метки и для break и для continue в циклах:


---

# Range

```kt
val x = 10
if (x in 1..<10) {
    println("fits in range")
}

for (x in 1..5) {
    print(x)
}

for (x in 9 downTo 0 step 3) {
    print(x)
}
```

- downTo и step - это extension функции, не ключевые слова.
- '..' это на самом деле T.rangeTo(that: T)

---

# null-safety

```kt
val notNullText: String = "Definitely not null"
val nullableText1: String? = "Might be null"
val nullableText2: String? = null

fun funny(text: String?) {
	if (text != null)
		println(text)
	else
		println("Nothing to print :(")
}

fun funnier(text: String?) {
	val toPrint = text ?: "Nothing to print :(" // Elvis оператор
	println(toPrint)
}
```

--- 

# Оператор безопасного вызова `?.`

```kt
var numberOfBooks: Int? = 6
if (numberOfBooks != null) {
    numberOfBooks = numberOfBooks.dec()
}
```
vs
```kt
var numberOfBooks: Int? = 6
numberOfBooks = numberOfBooks?.dec()
```

```kt
val a: List<Int>? = TODO()

val res: List<String> = a
  ?.filter { it % 2 == 0 }
  ?.map { it.toString() }
  ?: emptyList()


```

---

# Оператор `!!`

```kt
val s: String? = // что-то
// что-то
val len = s!!.length
```

- лучше не использовать
- говорит о плохом дизайне

---

# лямбды

```kt
val sum: (Int, Int) -> Int = { x: Int, y: Int -> x + y }
val mul = { x: Int, y: Int -> x * y }
```

```kt
val badProduct = items.fold(1, { acc, e -> acc * e })
val goodProduct = items.fold(1) { acc, e -> acc * e }
```

- можно не писать типы, если они есть в сигнатуре
- можно вынести лямбду за скобки

```kt
run({ println("Not Cool") })
run { println("Very Cool") }
```

- можно опустить скобки, если лямбда единственный параметр

--- 

# Полезные ресурсы

## Документация

https://kotlinlang.org/docs

## Интерактивный плейграунд

https://play.kotlinlang.org/

## Официальный мини-курс

1. [Hello world](https://kotlinlang.org/docs/kotlin-tour-hello-world.html)
2. [Basic types](https://kotlinlang.org/docs/kotlin-tour-basic-types.html)
3. [Collections](https://kotlinlang.org/docs/kotlin-tour-collections.html)
4. [Control flow](https://kotlinlang.org/docs/kotlin-tour-control-flow.html)
5. [Функции](https://kotlinlang.org/docs/kotlin-tour-functions.html)
6. [Классы](https://kotlinlang.org/docs/kotlin-tour-classes.html)
7. [Null safety](https://kotlinlang.org/docs/kotlin-tour-null-safety.html)

---