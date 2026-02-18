# Семестровый курс по Котлину

## Организационные моменты

- 8 лекций каждые две недели
- 15 практик
  - углубление материала
  - live-coding
- 8 домашек с допбаллами
- Проект
- Тесты на лекциях

### Оценивание

#### Домашки

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

#### Проект

- начинается во второй части курса
- критериии оценивания появятся позже TODO

#### Тесты

- на каждой лекции проводится письменный тест
- проверяются знания из предыдущих лекций и практик
- 5 минут

#### Итоговая оценка

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

## Лекции

### [Лекция 1](lectures/lecture-01.md)

- background языка
  - история
    - причины появления
    - популярность
  - основные отличия от Java
  - характеристики
    - сборка мусора
    - строго-типизированный
    - мультипарадигменный
  - применимость
    - Android
    - backend-разработка
    - JetBrains
    - domain-specific библиотеки
  - преимущества
- базовый синтаксис
  - объявления переменных
    - val
    - var
  - условия
  - when
  - циклы
  - функции
  - лямбды

### [Лекция 2](lectures/lecture-02.md)

- JDK, JVM
- компиляция и запуск
  - compile once, run anywhere
  - этапы компиляции
  - bytecode
    - стеково-регистровый
  - JIT
  - kotlinc
  - classpath
    - compiletime
    - runtime
  - java
  - jar-файлы
- системы сборки
  - maven
  - gradle
  - sbt

### [Лекция 3](lectures/lecture-03.md)

- типы
  - примитивные и ссылочные типы
  - nullable типы
  - массивы
    - инициализации
      - лямбды
    - двумерные массивы
  - представление в JVM
  - связь с Java
- ООП
- классы
  - class
  - методы, свойства
    - lateinit
    - модификаторы видимости
  - конструкторы
    - primary constructor
    - второстепенные конструкторы
    - init секция
  - data class
    - equals
    - hashcode
  - enum
  - object
  - data object
  - вложенные классы
  - companion object
- наследование
  - interface
  - abstract class
  - delegate
- глобальные переменные
  - const
- области видимости
  - модули
  - импорты

### [Лекция 4](lectures/lecture-04.md)

- многопоточность и корутины
  - модель памяти, happens before
  - Mutex
  - Atomic-и
  - synchronized
  - volatile переменные
  - корутины
  - suspension-поинты
  - внутреннее устройство
    - контекст
    - автомат
    - bytecode
  - работа с исключениями

### [Лекция 5](lectures/lecture-05.md)

- работа с сетью
  - клиент-серверная архитектура
  - HTTP протокол
    - GET/POST/PUT/UPDATE/DELETE
    - коды ответа
    - тело запроса
    - HTTPS
  - REST API
    - Swagger
    - OpenAPI
  - ktor

### [Лекция 6](lectures/lecture-06.md)

- Spring

### [Лекция 7](lectures/lecture-07.md)

- Android-разработка
  - история
  - архитектурные паттерны MVC, MVP, MVVM, MVI
  - DI
  - JetPack Compose

### [Лекция 8](lectures/lecture-08.md)

TODO

## Практики

### [Практика 1](practices/practice-01.md)

- IntellijIDEA
- hello world
- базовый синтаксис
  - объявления переменных
    - val
    - var
  - условия
  - when
  - циклы
  - функции
  - лямбды

### [Практика 2](practices/practice-02.md)

- null-safety:
  - ?.
  - !!
  - ?:
- стандартная библиотека
  - сахар
    - let
    - with
    - run
    - also
    - apply
    - use
  - коллекции
    - принципы проектирования коллекций в Kotlin
      - immutable
      - mutable
    - map
    - set
    - персистентные коллекции
    - HashSet vs LinkedHashSet
  - iterable
  - функциональная парадигма
    - функции для работы с коллекциями
      - filter
      - filterTo
      - map
      - mapTo
      - groupBy
      - associate
    - pipeline
    - asSequence

### [Практика 3](practices/practice-03.md)

- cli
  - установка JDK
  - javac, kotlinc
  - разбор .class
  - javap
  - запуск через jar
  - compile classpath
  - runtime classpath

### [Практика 4](practices/practice-04.md)

- maven
- gradle
  - таски
  - плагины
  - репозитории
  - зависимости
  - conventional plugins
  - публикация

### [Практика 5](practices/practice-05.md)

- Reflection

### [Практика 6](practices/practice-06.md)

- принцип SOLID
- основные паттерны разработки
  - visitor
  - strategy
  - template method
  - factory method
  - builder
  - facade
  - proxy
  - etc.

### [Практика 7](practices/practice-07.md)

- исключения
  - try
  - catch
  - finally
- продвинутые конструкции языка
  - extension функции
  - infix функции
  - операторы
  - delegate-properties
- generic-s
  - type bounds
  - invariant
  - covariant
- annotations

### [Практика 8](practices/practice-08.md)

- CoroutineContext
- CoroutineScope
- Job
- AsyncDeferred
- билдеры:
  - launch
  - async
  - coroutineScope

### [Практика 9](practices/practice-09.md)

- Channel
- Flow

### [Практика 10](practices/practice-10.md)

- Spring

### [Практика 11](practices/practice-11.md)

- Spring

### [Практика 12](practices/practice-12.md)

- JDBC
- JOOQ
- JPA
- Hibernate

### [Практика 13](practices/practice-13.md)

- логгирование
- мониторинг
- контейнеризация

### [Практика 14](practices/practice-14.md)

- Android-разработка
  - MVVM
  - JetPack Compose
  - DI: Dagger, Hilt, Koin

### [Практика 15](practices/practice-15.md)

- Android-разработка p2

## Домашки

### Домашка 1

TODO

### Домашка 2

TODO

### Домашка 3

TODO

### Домашка 4

TODO

### Домашка 5

TODO

### Домашка 6

TODO

### Домашка 7

TODO

### Домашка 8

TODO