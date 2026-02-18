# practice-01: hello world & basic syntax

План:

1. Установка IDEA
2. Создание проекта
3. Проверка окружения
4. Hello World
5. Разбор базового синтаксиса

## Полезные ссылки

### Скачивание IDEA

- macOS (ARM): https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1-aarch64.dmg
- macOS (Intel) https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1.dmg
- Windows: https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1.exe
- Linux: https://download-cdn.jetbrains.com/idea/idea-2025.3.1.1.tar.gz

### Интерактивный плейграунд

https://play.kotlinlang.org/

### Официальный мини-курс

1. [Hello world](https://kotlinlang.org/docs/kotlin-tour-hello-world.html)
2. [Basic types](https://kotlinlang.org/docs/kotlin-tour-basic-types.html)
3. [Collections](https://kotlinlang.org/docs/kotlin-tour-collections.html)
4. [Control flow](https://kotlinlang.org/docs/kotlin-tour-control-flow.html)
5. [Функции](https://kotlinlang.org/docs/kotlin-tour-functions.html)
6. [Классы](https://kotlinlang.org/docs/kotlin-tour-classes.html)
7. [Null safety](https://kotlinlang.org/docs/kotlin-tour-null-safety.html)

## Создаём Котлин-проект в Intellij IDEA

1. `File->New->Project`
2. Указываем имя
3. Слева выбираем `Kotlin`
4. Систему сборки (Build System) выбираем `Gradle`
5. JDK выбираем Azul Zulu 21 (IDEA предложит скачать)
6. Остальное оставляем как есть
7. Нажимаем `Create`

## Примеры на Котлин

Ознакомьтесь со следующими примерами на языке Kotlin:

### Hello world

```kotlin
fun main() {
    println("Hello world!")
}
```

### Объявления переменных

```kotlin
fun main() {
    val a: Int
    // println(a)

    val b = 1
    // b++ CE
    var c = 0
    c++
    println(b == c) // true

    val hello = "Hello"
    val world = "world"
    println(hello + " " + world + "!") // Hello world!
    println("$hello $world!") // Hello world!

    val l: Long = 0

    val chr = '!'
    
    val (int1, int2) = listOf(10, 20)
    val (integer, string) = Pair(1337, "1337")
    
    val str: String?
    
    if (3 * 3 + 4 * 4 == 5 * 5) {
        str = null
    } else {
        str = "kak tak?.."
    }
    println(str) // null
    
    val matrix = arrayOf(arrayOf<Int>())
}
```

### Функции

```kotlin
fun sum(a: Int, b: Int): Int {
    return a + b
}

fun mult(a: Int, b: Int) = a * b
```

### Циклы

```kotlin
fun main() {
    var i = 0
    while (true) {
        if (i == 10) {
            break
        }
        i++
        if (i % 3 == 0) {
            continue
        }
        i++
    }
    
    for (i in 0 until 5) { // 0, 1, 2, 3, 4
        for (j in 0..2) { // 0, 1, 2
            println("($i, $j)")
        }
    }
    
    val primes = listOf(2, 3, 5, 7, 11, 13, 17)
    

    val x = 143
    
    for (p in primes) {
        if (x % p == 0) {
            println("Not prime!")
        }
    }
    println("Probably prime")
}

```

### Ввод-вывод

```kotlin
fun main() {
    val n = readln().toInt()

    val (m, k) = readln().split(" ").map { it.toInt() }

    val arr = readln().split(" ").map { it.toInt() }

    val res = MutableList(arr.size) { 0 }

    for (i in res.indices) {
        res[i] = arr[i] * i * m + k
    }

    println("arr[i] * i * m + k =\n")
    println(res.joinToString(" "))
    println("Sum = ${res.sum()}")
}

```

### Коллекции

```kotlin
fun main() {
    val t1 = listOf(1, 2, 3)
    // t1[0] = -1 CE
    val t2 = mutableListOf(1, 2, 3)
    t2[0] = -1
    
    val emptyList1 = emptyList<Int>()
    val emptyList2 = listOf<Int>()
    
    val setik = setOf(1, 1, 2)
    // setik.insert(3) CE
    val mutableSetik = setik.toMutableSet()
    mutableSetik.insert(3)
    println("${setik.size} ${mutableSetik.size}")

    val numbers = (0..10)
    val x2plus5 = numbers.map { it * 2 + 5 }
    println(x2plus5.toString())

    val allOdd = x2plus5.all { it % 2 != 0 }
    println(allOdd)

    val filtered = x2plus5.filter { it > 10 }
    println(filtered.toString())
}
```
