---
info: Лекция 7 — Разработка Android-приложений
class: text-base
---

<style>
.two-cols-header {
  column-gap: 20px;
}

.two-cols {
  column-gap: 20px;
}
</style>


# Лекция 7. Разработка Android-приложений

---
layout: two-cols-header
---

## Что мы обсудим



::left::


- Android-разработка как применение Kotlin: контекст и история
- Android как платформа
- Инструменты разработки
- Базовое устройство Android-приложения
- UI в Android: от XML к Jetpack Compose
- Kotlin-фичи в Android-разработке
- Архитектурные подходы
- Jetpack-библиотеки
- Dependency Injection
- Данные, сеть и фоновые задачи
- Тестирование Android-приложений
- Публикация и сопровождение
- Современные тренды

::right::

<img src="https://upload.wikimedia.org/wikipedia/commons/d/d7/Android_robot.svg" width="260" />

> Цель лекции — познакомиться с современной Android-разработкой и увидеть, какую роль в ней играет Kotlin.


---

# Android-разработка как применение Kotlin

Android — одно из самых массовых применений Kotlin в индустрии.

- Kotlin официально поддерживается Google для Android-разработки
- современный Android-код часто пишут именно на Kotlin
- многие библиотеки Android используют Kotlin-first API
- Kotlin хорошо ложится на задачи мобильной разработки:
  - работа с состоянием
  - асинхронность
  - UI-код
  - обработка nullable-данных
  - декларативные DSL

> Android — хороший пример того, как язык влияет не только на синтаксис, но и на стиль разработки.

---
layout: two-cols-header
---

## Android: короткая историческая шкала

::left::

```text
2003 ──► Android Inc.
2005 ──► Google покупает Android
2008 ──► первый Android-смартфон
2017 ──► Kotlin получает официальную поддержку
2019 ──► Kotlin становится preferred language
2021 ──► Jetpack Compose 1.0
```

::right::

За это время Android-разработка заметно изменилась:

- Java → Kotlin
- XML-layouts → Jetpack Compose
- callbacks → coroutines
- ручная склейка зависимостей → DI-контейнеры
- Activity-heavy приложения → архитектура со слоями

> Android — платформа с большой историей, поэтому в ней много «старого» и «нового» одновременно.

---
layout: center
---

## Первый Android-смартфон

<img src="https://upload.wikimedia.org/wikipedia/commons/1/18/T-Mobile_G1_launch_event_2.jpg" width="420" />

T-Mobile G1 / HTC Dream, 2008 год.

---
layout: two-cols-header
---

## Как писали Android UI раньше

::left::

Классический Android UI строился вокруг XML-разметки и View system.

```xml
<TextView
    android:id="@+id/title"
    android:layout_width="wrap_content"
    android:layout_height="wrap_content"
    android:text="Hello, Android!" />
```

```kotlin
val title = findViewById<TextView>(R.id.title)
title.text = "Hello, Kotlin!"
```

::right::

Проблемы подхода:

- UI описан в XML, логика — в Kotlin/Java
- нужно вручную связывать View и код
- состояние экрана размазано по разным местам
- сложные экраны быстро становятся хрупкими
- много шаблонного кода

> Этот подход всё ещё встречается в реальных проектах, особенно в старых кодовых базах.

---

## Почему Kotlin хорошо подошёл Android

Kotlin не просто заменил Java синтаксически — он закрыл типичные боли Android-разработки.

| Боль Android-разработки | Что даёт Kotlin |
|---|---|
| `NullPointerException` | null-safety на уровне типов |
| модели данных с кучей boilerplate | `data class` |
| вложенные callbacks | coroutines и `suspend` |
| одноразовые состояния UI | `sealed class` / `sealed interface` |
| утилитные методы вокруг Android API | extension functions |
| декларативные конфигурации | DSL-подход |

> Поэтому Kotlin стал не просто «ещё одним языком», а частью современного Android-стека.

---
layout: two-cols-header
---

## От imperative UI к declarative UI

::left::

**Раньше:** меняем экран командами.

```kotlin
progressBar.isVisible = true
button.isEnabled = false
textView.text = "Loading..."
```

Мы вручную говорим UI, **что поменять**.

::right::

**Сейчас:** описываем экран как функцию от состояния.

```kotlin
@Composable
fun ProfileScreen(isLoading: Boolean) {
    if (isLoading) {
        CircularProgressIndicator()
    } else {
        Text("Profile loaded")
    }
}
```

UI сам перестраивается при изменении состояния.

> Это главный поворот, который привёл Android к Jetpack Compose.

---

# Android как платформа

---
layout: two-cols-header
---

## Android — это не просто «телефонная Java»

::left::

Android-приложение работает не напрямую «на железе», а внутри большой платформы:

- операционная система
- модель безопасности
- среда исполнения
- системные службы
- набор API для приложений
- магазин и механизм доставки обновлений

::right::

```text
┌────────────────────────────────┐
│        Your Application        │
├────────────────────────────────┤
│      Android Framework API     │
├────────────────────────────────┤
│          Android Runtime       │
├────────────────────────────────┤
│ Native Libraries / HAL / Binder│
├────────────────────────────────┤
│           Linux Kernel         │
└────────────────────────────────┘
```

> Поэтому Android-разработка — это не только язык, но и понимание платформенных правил.

---
layout: two-cols-header
---

## Android OS

::left::

Android основан на Linux kernel.

Отсюда приходят важные идеи:

- процессы
- файловая система
- пользователи и права доступа
- драйверы устройств
- управление памятью
- сетевой стек
- изоляция приложений

::right::

Но Android — не обычный Linux-дистрибутив:

- нет привычного desktop-окружения
- приложения живут в управляемой модели жизненного цикла
- доступ к возможностям устройства контролируется permissions
- многое завязано на системные службы Android

> Linux даёт фундамент, Android добавляет мобильную модель приложений.

---

## Open-source основа и роль Google

Android имеет open-source основу — **Android Open Source Project**, или **AOSP**.

Но реальная экосистема Android шире, чем AOSP:

| Часть экосистемы | Что это |
|---|---|
| AOSP | открытая основа Android |
| Google Mobile Services | Google Play, Maps, Firebase Cloud Messaging и другие сервисы |
| Производители устройств | Samsung, Xiaomi, Google, OnePlus и другие |
| Google Play | основной канал распространения приложений |

> Поэтому Android одновременно открытая платформа и экосистема с сильной ролью Google.

---
layout: two-cols-header
---

## Android Runtime: Dalvik и ART

::left::

Исторически Android использовал **Dalvik** — виртуальную машину, оптимизированную для мобильных устройств.

Позже её заменил **ART** — Android Runtime.

```text
Kotlin / Java source
        │
        ▼
 JVM bytecode
        │
        ▼
 DEX bytecode
        │
        ▼
 Android Runtime
```

::right::

Почему не обычная JVM?

- мобильные устройства ограничены по памяти и батарее
- приложения должны запускаться быстро
- система должна уметь управлять множеством приложений
- Android использует собственный формат байткода — DEX

> Kotlin-код компилируется не «прямо в Android», а проходит через JVM-экосистему и Android-инструменты сборки.

---

## JVM bytecode vs DEX bytecode

| JVM bytecode | DEX bytecode |
|---|---|
| хранится в `.class` файлах | хранится в `.dex` файлах |
| рассчитан на стековую виртуальную машину | рассчитан на регистровую виртуальную машину |
| один `.class` обычно описывает один класс | один `.dex` объединяет много классов |
| используется обычной JVM | используется Android Runtime |

> DEX — это не «другой язык», а другой формат байткода, оптимизированный под модель исполнения Android.

---
layout: two-cols-header
---

## APK и AAB

::left::

**APK** — привычный формат Android-приложения для установки на устройство.

Внутри находятся:

- скомпилированный код
- ресурсы
- манифест
- нативные библиотеки
- подпись приложения

::right::

**AAB** — Android App Bundle, формат публикации в Google Play.

Google Play может собрать из него оптимизированный APK под конкретное устройство:

- нужная архитектура процессора
- нужные языки
- нужная плотность экрана
- dynamic features

> Разработчик чаще публикует AAB, а пользователь в итоге получает подходящий APK.

---
layout: two-cols-header
---

## Sandbox и permissions

::left::

Каждое Android-приложение работает в своей песочнице.

```text
App A ──► свои файлы, свой процесс, свой UID
App B ──► свои файлы, свой процесс, свой UID
```

По умолчанию приложение не может свободно читать данные других приложений.

::right::

Для доступа к чувствительным возможностям нужны permissions:

- камера
- микрофон
- геолокация
- контакты
- уведомления
- файлы и медиа
- Bluetooth

Некоторые разрешения пользователь выдаёт во время работы приложения.

> Модель безопасности — одна из причин, почему Android-код нельзя воспринимать как обычную JVM-программу.

---

# Инструменты разработки

---
layout: two-cols-header
---

## Android Studio

::left::

**Android Studio** — основная среда разработки для Android.

Она построена на базе IntelliJ IDEA и добавляет Android-специфичные возможности:

- создание и настройка Android-проекта
- редактор UI и Compose Preview
- запуск приложения на устройстве или эмуляторе
- просмотр логов через Logcat
- профилирование CPU, памяти, сети и батареи
- интеграция с Gradle и Android SDK

::right::

<img src="https://upload.wikimedia.org/wikipedia/commons/9/92/Android_Studio_Trademark.svg" width="260" />

> Для Kotlin-разработчика Android Studio ощущается знакомо, потому что основана на той же платформе, что и IntelliJ IDEA.

---
layout: two-cols-header
---

## Android SDK

::left::

**Android SDK** — набор инструментов и API для разработки под Android.

В него входят:

- платформенные API разных версий Android
- build tools
- platform tools
- Android Debug Bridge, или `adb`
- emulator
- system images для виртуальных устройств

::right::

```text
Android Studio
      │
      ▼
 Android SDK ──► build tools
      │          platform tools
      │          emulator
      │          system images
      ▼
 Android device
```

> IDE — это интерфейс для разработчика, SDK — фактический набор инструментов, которыми собирают и запускают приложение.

---

## Emulator

**Android Emulator** позволяет запускать приложение без физического устройства.

Зачем он нужен:

- быстро проверять приложение на разных версиях Android
- тестировать разные размеры и плотности экранов
- имитировать поворот экрана, геолокацию, сеть, батарею
- запускать автоматические проверки
- воспроизводить ошибки в контролируемой среде

> В реальной разработке обычно используют и эмулятор, и физическое устройство: у каждого варианта свои плюсы.

---
layout: center
---

## Быстрая проверка UI в эмуляторе

<img src="https://developer.android.com/static/develop/ui/compose/images/live-editing-of-literals.gif" width="620" />

> Эмулятор удобен не только для запуска приложения, но и для быстрой проверки изменений интерфейса.

---
layout: two-cols-header
---

## Сборка Android-приложения

::left::

Android-проект обычно собирается через **Gradle** и **Android Gradle Plugin**.

Они отвечают за:

- подключение зависимостей
- компиляцию Kotlin и Java
- преобразование байткода в DEX
- упаковку ресурсов
- сборку APK или AAB
- разные варианты сборки

::right::

```text
Kotlin / Java source
        │
        ▼
 Gradle + Android Gradle Plugin
        │
        ├──► compile
        ├──► dex
        ├──► package resources
        ├──► sign
        ▼
      APK / AAB
```

> В Android Gradle — не просто «сборщик», а центральная часть всей цепочки разработки.

---

# Базовое устройство Android-приложения

---
layout: two-cols-header
---

## Что лежит внутри Android-приложения

::left::

Типичное приложение состоит из нескольких важных частей:

- `AndroidManifest.xml`
- `Application`
- `Activity`
- resources
- Kotlin/Java-код
- зависимости
- скомпилированные assets и native-библиотеки

::right::

```text
project/
├── settings.gradle.kts
├── build.gradle.kts
├── gradle.properties
├── gradlew
├── gradle/
│   └── wrapper/
└── app/
    ├── build.gradle.kts
    └── src/main/
        ├── AndroidManifest.xml
        ├── java/com/example/app/
        │   └── MainActivity.kt
        └── res/
            ├── drawable/
            ├── mipmap/
            ├── values/
            └── xml/
```

> Android-проект — это не только код: ресурсы и манифест являются частью модели приложения.

---
layout: two-cols-header
---

## AndroidManifest.xml

::left::

`AndroidManifest.xml` описывает приложение для системы Android.

В нём указывают:

- package / namespace приложения
- компоненты приложения
- разрешения
- intent filters
- минимальные требования
- тему приложения

::right::

```xml
<manifest>
    <uses-permission android:name="android.permission.CAMERA" />

    <application
        android:name=".App"
        android:theme="@style/AppTheme">

        <activity
            android:name=".MainActivity"
            android:exported="true" />
    </application>
</manifest>
```

> Манифест — контракт между приложением и операционной системой.

---
layout: two-cols-header
---

## Application и Activity

::left::

`Application` — объект, живущий на уровне всего процесса приложения.

Обычно там инициализируют:

- DI-контейнер
- логирование
- аналитику
- глобальные библиотеки

::right::

`Activity` — экран или точка входа в экранный интерфейс.

Она отвечает за:

- создание UI
- получение intent
- взаимодействие с lifecycle
- связь с ViewModel или другим слоем состояния

> В современных приложениях Activity часто становится тонкой оболочкой вокруг Compose UI или Fragment-контейнера.

---
layout: two-cols-header
---

## Эволюция роли Activity

::left::

**Раньше:** несколько `Activity` в одном приложении.

```text
LoginActivity
      │
      ▼
CatalogActivity
      │
      ▼
DetailsActivity
      │
      ▼
CheckoutActivity
```

Часто одна `Activity` соответствовала одному экрану.

::right::

**Сейчас:** чаще одна `Activity` на всё приложение.

```text
MainActivity
      │
      ▼
Compose Navigation / Fragment Container
      │
      ├── LoginScreen
      ├── CatalogScreen
      ├── DetailsScreen
      └── CheckoutScreen
```

Экраны и навигация живут внутри UI-слоя.

---
layout: two-cols-header
---

## Activity lifecycle

::left::

`Activity` живёт не линейно: Android может останавливать, возобновлять и пересоздавать экран.

Основные callbacks:

- `onCreate()`
- `onStart()`
- `onResume()`
- `onPause()`
- `onStop()`
- `onDestroy()`
- `onRestart()`

::right::

<img src="https://developer.android.com/guide/components/images/activity_lifecycle.png" width="380" />

> Lifecycle нужен, чтобы не терять состояние, не держать лишние ресурсы и корректно переживать переходы между экранами.

---
layout: two-cols-header
---

## Resources

::left::

Android отделяет ресурсы от кода.

В `res/` обычно лежат:

- строки
- цвета
- темы
- изображения
- иконки
- XML-конфигурации
- layout-файлы в старом UI-подходе

::right::

```xml
<!-- res/values/strings.xml -->
<resources>
    <string name="app_name">Booking App</string>
    <string name="save">Save</string>
</resources>
```

```kotlin
stringResource(R.string.save)
```

> Ресурсы помогают поддерживать локализацию, разные экраны, темы и конфигурации устройств.

---
layout: two-cols-header
---

## Configuration changes

::left::

Устройство может менять конфигурацию прямо во время работы приложения:

- поворот экрана
- смена языка
- изменение темы
- изменение размера окна
- подключение внешней клавиатуры
- foldable-сценарии

::right::

Что может произойти:

```text
Activity работает
      │
      ▼
поворот экрана
      │
      ▼
Activity уничтожена и создана заново
```

Если состояние хранится только во View, оно может потеряться.

> Поэтому Android-разработка сильно завязана на lifecycle и правильное хранение состояния.

---

# UI в Android: как было раньше

---
layout: two-cols-header
---

## XML-разметка и View system

::left::

До Jetpack Compose интерфейс Android-приложений обычно описывали в XML.

```xml
<LinearLayout
    android:layout_width="match_parent"
    android:layout_height="match_parent"
    android:orientation="vertical">

    <TextView
        android:id="@+id/title"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Hello, Android!" />

    <Button
        android:id="@+id/button"
        android:layout_width="wrap_content"
        android:layout_height="wrap_content"
        android:text="Click" />
</LinearLayout>
```

::right::

Основные сущности:

- `View` — отдельный элемент интерфейса
- `ViewGroup` — контейнер для других `View`
- layout XML — декларативное описание дерева UI
- `Activity` или `Fragment` загружает layout и управляет им из кода

```text
LinearLayout
├── TextView
└── Button
```

> UI был декларативным в XML, но его поведение оставалось императивным в Kotlin/Java-коде.

---
layout: two-cols-header
---

## findViewById

::left::

Чтобы работать с View из кода, её нужно было найти по `id`.

```kotlin
class MainActivity : Activity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        setContentView(R.layout.activity_main)

        val title = findViewById<TextView>(R.id.title)
        val button = findViewById<Button>(R.id.button)

        button.setOnClickListener {
            title.text = "Clicked"
        }
    }
}
```

::right::

Минусы подхода:

- связь XML и кода проверяется поздно
- можно ошибиться с типом View
- много ручного связывания
- код быстро обрастает обращениями к UI-элементам
- состояние экрана легко размазать по обработчикам

> `findViewById` — хороший пример того, почему Android-код раньше часто становился многословным.

---
layout: two-cols-header
---

## RecyclerView

::left::

`RecyclerView` — основной старый инструмент для списков.

Он нужен, когда элементов много:

- лента сообщений
- список товаров
- каталог фильмов
- список настроек
- результаты поиска

Идея: переиспользовать ячейки, а не создавать View для каждого элемента заново.

::right::

```text
visible items
┌──────────────┐
│ item view #1 │  ← переиспользуется
├──────────────┤
│ item view #2 │  ← переиспользуется
├──────────────┤
│ item view #3 │  ← переиспользуется
└──────────────┘

Adapter ──► создаёт ViewHolder
        └─► привязывает данные к View
```

> RecyclerView эффективен, но требует много инфраструктурного кода: adapter, view holder, diffing, layout manager.

---
layout: two-cols-header
---

## View Binding и Data Binding

::left::

Чтобы убрать ручной `findViewById`, появились вспомогательные подходы.

**View Binding** генерирует типобезопасный класс для XML-layout.

```kotlin
val binding = ActivityMainBinding.inflate(layoutInflater)
setContentView(binding.root)

binding.title.text = "Hello"
binding.button.setOnClickListener {
    binding.title.text = "Clicked"
}
```

::right::

**Data Binding** пошёл дальше: позволил связывать XML с данными и выражениями.

```xml
<TextView
    android:text="@{viewModel.title}" />
```

Что это решало:

- меньше `findViewById`
- меньше ошибок с типами
- проще связать layout и код

Но XML и логика всё равно оставались в разных мирах.

---

## Главные проблемы классического UI-подхода

| Проблема | Как проявлялась |
|---|---|
| UI разделён между XML и кодом | чтобы понять экран, нужно смотреть несколько файлов |
| состояние меняется вручную | `textView.text = ...`, `button.isEnabled = ...` |
| много шаблонного кода | adapters, holders, bindings, listeners |
| lifecycle влияет на UI | экран могут пересоздать при смене конфигурации |
| сложная навигация | Activity/Fragment back stack, saved state |

> Jetpack Compose появился как ответ на эту сложность: описывать UI прямо в Kotlin как функцию от состояния.

---
layout: center
---

# Jetpack Compose

<img src="https://upload.wikimedia.org/wikipedia/commons/9/98/Jetpack_Compose_logo.png" width="320" />

Современный декларативный UI-фреймворк для Android.

---

## История появления Compose

```text
2019 ──► первый публичный preview Jetpack Compose
2021 ──► Jetpack Compose 1.0
2022 ──► Material 3 в Compose
2023+ ─► Compose становится основным UI-подходом в Android
```

Почему появился Compose:

- XML + View system плохо масштабировались на сложные состояния
- UI был разделён между layout-файлами и Kotlin/Java-кодом
- RecyclerView, adapters, bindings и fragments давали много boilerplate
- индустрия уже двигалась к declarative UI: React, SwiftUI, Flutter

> Compose — это попытка сделать Android UI проще, ближе к Kotlin и лучше приспособленным к state-driven приложениям.

---
layout: two-cols-header
---

## Основная идея Compose

::left::

В Compose интерфейс описывается прямо на Kotlin.

```kotlin
@Composable
fun Greeting(name: String) {
    Text(text = "Hello, $name!")
}
```

`@Composable` функция не «рисует пиксели» вручную — она описывает, какой UI должен быть на экране.

::right::

Ключевая идея:

```text
state ──► UI
```

UI становится функцией от состояния:

- изменились данные
- Compose понял, какие части UI зависят от них
- нужные части интерфейса обновились

> Вместо ручного изменения View мы описываем результат, который хотим получить.

---
layout: two-cols-header
---

## Что такое @Composable

::left::

`@Composable` помечает функцию, которую может вызвать Compose runtime.

```kotlin
@Composable
fun UserCard(user: User) {
    Column {
        Text(user.name)
        Text(user.email)
    }
}
```

Такие функции описывают кусок UI и могут вызывать другие composable-функции.

::right::

Важные свойства:

- имя обычно начинается с большой буквы, как у компонента
- функция может быть вызвана много раз из-за recomposition
- она должна быть быстрой и без неожиданных side effects
- входные параметры должны описывать состояние UI
- результат не возвращается как `View`: UI строит Compose runtime

> `@Composable` — это не просто аннотация для красоты, а вход в другую модель исполнения UI-кода.

---
layout: two-cols-header
---

## Imperative vs declarative UI

::left::

**Imperative UI:** говорим, что поменять.

```kotlin
progressBar.isVisible = true
button.isEnabled = false
textView.text = "Loading..."
```

Проблема: легко забыть обновить одну из View.

::right::

**Declarative UI:** описываем экран для состояния.

```kotlin
@Composable
fun Screen(isLoading: Boolean) {
    if (isLoading) {
        CircularProgressIndicator()
    } else {
        Text("Loaded")
    }
}
```

Compose сам применяет нужные изменения.

---
layout: two-cols-header
---

## MVC: Model — View — Controller

::left::

**MVC** разделяет приложение на три роли:

- `Model` — данные и бизнес-логика
- `View` — отображение данных
- `Controller` — обработка пользовательского ввода

```text
User input
    │
    ▼
Controller ──► Model
    │           │
    ▼           ▼
  View ◄────── data
```

::right::

В Android классическая `Activity` часто становилась одновременно View и Controller.

```text
Activity
├── показывает UI
├── обрабатывает клики
├── ходит в Model
└── обновляет View
```

> На практике MVC в Android легко превращался в «Massive Activity».

---
layout: two-cols-header
---

## MVP: Model — View — Presenter

::left::

**MVP** выносит логику экрана в `Presenter`.

```text
View ── user event ──► Presenter ──► Model
 ▲                         │          │
 │                         ▼          │
 └──── render commands ◄── result ◄───┘
```

`View` обычно пассивная:

- показывает данные
- сообщает о событиях
- реализует интерфейс для Presenter

::right::

```kotlin
interface LoginView {
    fun showLoading()
    fun showError(message: String)
    fun openProfile()
}
```

Плюсы:

- логику проще тестировать отдельно от Android UI
- меньше кода в `Activity` / `Fragment`

Минусы:

- много интерфейсов и ручной связки
- Presenter часто знает слишком много о View

---
layout: two-cols-header
---

## MVVM: Model — View — ViewModel

::left::

**MVVM** делает состояние экрана отдельным объектом.

```text
          UiState
ViewModel ───────► View
    ▲              │
    │              ▼
    └── UI events ◄┘
```

`ViewModel`:

- хранит состояние экрана
- переживает configuration changes
- вызывает use cases / repositories
- отдаёт `UiState` наружу

::right::

С Compose MVVM особенно естественен:

```text
ViewModel ──► UiState ──► Composable UI
    ▲                         │
    └──────── UI events ◄─────┘
```

Composable-функции отображают состояние и отправляют события наверх.

> Compose не требует строго MVVM, но хорошо сочетается с state-driven архитектурой.

---
layout: two-cols-header
---

## Unidirectional Data Flow

::left::

**Unidirectional Data Flow**, или UDF, — однонаправленный поток данных.

```text
State ──► UI ──► Event ──► State
```

Данные идут вниз: от владельца состояния к UI.

События идут вверх: от UI к владельцу состояния.

::right::

```kotlin
@Composable
fun CounterScreen(
    count: Int,
    onIncrement: () -> Unit,
) {
    Button(onClick = onIncrement) {
        Text("Count: $count")
    }
}
```

Composable не решает, как менять состояние. Он только показывает `count` и сообщает о событии.

> UDF делает экран предсказуемым: одно состояние — один UI.

---
layout: two-cols-header
---

## Recomposition

::left::

**Recomposition** — повторный вызов `@Composable` функций при изменении состояния.

```kotlin
@Composable
fun Counter() {
    var count by remember { mutableStateOf(0) }

    Button(onClick = { count++ }) {
        Text("Count: $count")
    }
}
```

Когда `count` меняется, Compose заново вычисляет зависимый UI.

::right::

Важно:

- recomposition не означает «перерисовать всё приложение»
- Compose старается обновлять только нужные части
- `@Composable` функции должны быть быстрыми
- побочные эффекты нельзя запускать просто в теле composable

> Модель похожа на реакцию таблицы: изменили ячейку с данными — пересчитались зависящие формулы.

---
layout: two-cols-header
---

## Состояние: remember и state hoisting

::left::

`remember` хранит состояние между recomposition.

```kotlin
@Composable
fun SearchBox() {
    var query by remember { mutableStateOf("") }

    TextField(
        value = query,
        onValueChange = { query = it }
    )
}
```

::right::

**State hoisting** — подъём состояния выше по дереву UI.

```kotlin
@Composable
fun SearchBox(
    query: String,
    onQueryChange: (String) -> Unit,
) {
    TextField(
        value = query,
        onValueChange = onQueryChange
    )
}
```

> Хороший Compose-код отделяет состояние от отображения: так UI проще тестировать и переиспользовать.

---
layout: two-cols-header
---

## Preview

::left::

Compose позволяет смотреть UI без запуска всего приложения.

```kotlin
@Preview(showBackground = true)
@Composable
fun GreetingPreview() {
    Greeting(name = "Android")
}
```

Это ускоряет разработку экранов и компонентов.

::right::

Preview помогает проверять:

- разные состояния экрана
- светлую и тёмную темы
- размеры экранов
- локализацию
- отдельные UI-компоненты

> Preview хорошо сочетается с идеей маленьких независимых composable-функций.

---
layout: two-cols-header
---

## Material Design в Compose

::left::

Compose поставляется с готовыми Material-компонентами.

```kotlin
@Composable
fun SaveButton(onClick: () -> Unit) {
    Button(onClick = onClick) {
        Text("Save")
    }
}
```

Есть готовые элементы:

- `Button`
- `TextField`
- `Card`
- `Scaffold`
- `TopAppBar`
- `NavigationBar`

::right::

Material в Compose — это не только компоненты, но и тема приложения.

```kotlin
MaterialTheme(
    colorScheme = lightColorScheme(),
    typography = Typography(),
) {
    AppContent()
}
```

Тема задаёт:

- цвета
- типографику
- формы
- поведение компонентов

> Благодаря Material можно быстро собрать интерфейс, который выглядит как современное Android-приложение.

---

## Compose ecosystem

| Часть | Для чего нужна |
|---|---|
| Material Design в Compose | готовые компоненты и темы |
| Compose Navigation | навигация между экранами |
| Compose UI Testing | проверка UI в тестах |
| Compose Preview | быстрая разработка компонентов |
| Compose Multiplatform | общий declarative UI за пределами Android |

> Compose — это уже не отдельная библиотека для кнопок, а большая часть современной Android-экосистемы.

---

# Библиотеки современного Android-приложения

---

## Зачем столько библиотек?

Android-приложение решает много повторяющихся задач.

Вместо того чтобы каждый раз писать инфраструктуру вручную, обычно берут готовые библиотеки.

| Проблема | Типичные решения |
|---|---|
| lifecycle и состояние экрана | Lifecycle, ViewModel, Flow |
| навигация | Navigation, Compose Navigation |
| зависимости между объектами | Hilt, Dagger, Koin |
| списки и пагинация | Paging |

> Первая группа библиотек помогает организовать экран, состояние, навигацию и зависимости.

---

## Библиотеки для данных и фоновой работы

| Проблема | Типичные решения |
|---|---|
| локальные данные | Room, DataStore |
| сеть и JSON | Retrofit, OkHttp, kotlinx.serialization, Moshi |
| фоновые задачи | WorkManager, FCM |
| изображения | Coil, Glide |

> Вторая группа помогает получать, хранить, синхронизировать и показывать данные.

---
layout: two-cols-header
---

## Экран, состояние и навигация

::left::

Эта группа библиотек помогает строить UI, который переживает lifecycle Android.

- **Lifecycle** — модель жизненного цикла компонентов
- **ViewModel** — состояние экрана вне `Activity` / `Fragment`
- **Flow** — поток изменений состояния
- **Navigation / Compose Navigation** — переходы между экранами

```text
ViewModel ──► StateFlow<UiState> ──► UI
    ▲                                 │
    └──────────── UI events ◄─────────┘
```

::right::

Что это решает:

- экран пересоздали после поворота — состояние не потерялось
- UI не хранит бизнес-логику
- навигация описана явно
- проще тестировать экран по состояниям

```kotlin
class ProfileViewModel : ViewModel() {
    val state: StateFlow<ProfileState> = TODO()
}
```

> Центральная идея: экран показывает состояние, а не управляет всем приложением сам.

---
layout: two-cols-header
---

## Dependency Injection

::left::

**DI** решает проблему создания и связывания объектов.

Без DI код быстро превращается в цепочку ручных конструкторов:

```kotlin
val api = UserApi(okHttpClient)
val dao = UserDao(database)
val repo = UserRepository(api, dao)
val vm = ProfileViewModel(repo)
```

DI-контейнер берёт эту сборку на себя.

::right::

Популярные решения:

| Библиотека | Идея |
|---|---|
| Dagger | compile-time DI, максимум строгости |
| Hilt | Android-friendly обёртка над Dagger |
| Koin | простой Kotlin DSL, runtime DI |

```kotlin
@HiltViewModel
class ProfileViewModel @Inject constructor(
    private val repository: UserRepository,
) : ViewModel()
```

> DI нужен не «для красоты», а чтобы код был тестируемым и не зависел от ручной сборки объектов.

---
layout: two-cols-header
---

## Данные: локально и по сети

::left::

Локальное хранение:

- **DataStore** — небольшие настройки вместо старого `SharedPreferences`
- **Room** — удобный слой поверх SQLite
- migrations — безопасное изменение схемы БД
- кеширование и offline-first подход

```kotlin
@Dao
interface UserDao {
    @Query("SELECT * FROM users")
    fun observeUsers(): Flow<List<UserEntity>>
}
```

::right::

Сеть:

- **OkHttp** — HTTP-клиент
- **Retrofit** — типобезопасное описание REST API
- **kotlinx.serialization / Moshi** — JSON
- **Coil / Glide** — загрузка изображений

```kotlin
interface UserApi {
    @GET("users/{id}")
    suspend fun getUser(@Path("id") id: Long): UserDto
}
```

> Обычно данные приходят из сети, кешируются локально и отображаются через состояние UI.

---
layout: two-cols-header
---

## Фоновые задачи и ограничения платформы

::left::

Android строго ограничивает работу в фоне, чтобы экономить батарею.

Исторические механизмы:

- `Service`
- `BroadcastReceiver`

Современное решение для отложенной работы:

- **WorkManager**

Для push-событий часто используют:

- **Firebase Cloud Messaging**

::right::

Примеры задач:

- синхронизировать данные ночью
- загрузить большой файл
- отправить аналитику позже
- показать push-уведомление
- повторить запрос при появлении сети

```text
App request ──► WorkManager
                  │
                  ├── constraints: Wi‑Fi, charging
                  └── retry / backoff / persistence
```

> В Android нельзя просто «запустить вечный поток»: фоновые задачи должны уважать правила платформы.

---

# Тестирование Android-приложений

---

## Почему Android-тесты сложнее обычных unit-тестов

Android-приложение зависит не только от Kotlin-кода, но и от платформы.

Что усложняет тестирование:

- lifecycle `Activity` и `Fragment`
- главный UI-поток
- ресурсы и контекст Android
- разрешения
- база данных и файловая система
- сеть
- разные версии Android и размеры экранов

> Поэтому в Android обычно используют несколько уровней тестов, а не один универсальный тип.

---

## Уровни тестирования

| Тип теста | Где запускается | Что проверяет |
|---|---|---|
| Unit tests | JVM на компьютере разработчика | чистую бизнес-логику |
| Instrumented tests | Android device / emulator | код, зависящий от Android framework |
| UI tests | device / emulator | поведение интерфейса глазами пользователя |
| Screenshot tests | JVM или device | визуальные изменения экранов |

```text
много быстрых unit-тестов
        ▲
        │
меньше интеграционных и UI-тестов
        ▲
        │
совсем мало end-to-end сценариев
```

> Чем ближе тест к реальному устройству, тем он полезнее для интеграции, но обычно медленнее и хрупче.

---
layout: two-cols-header
---

## Unit tests

::left::

Unit-тесты проверяют код, который не зависит напрямую от Android UI.

Хорошие кандидаты:

- use cases
- repositories с fake-зависимостями
- форматирование данных
- валидация форм
- reducers для UI state

```kotlin
@Test
fun `empty email is invalid`() {
    val result = validateEmail("")

    assertFalse(result.isValid)
}
```

::right::

Чтобы unit-тесты были простыми, код должен быть отделён от платформы.

```text
UI layer ──► ViewModel ──► UseCase ──► Repository
                         ▲
                         │
                    легко тестировать
```

> Архитектура напрямую влияет на тестируемость приложения.

---
layout: two-cols-header
---

## Instrumented и UI tests

::left::

Instrumented-тесты запускаются на реальном Android runtime.

Они нужны, когда код использует:

- `Context`
- ресурсы
- Room database
- Android framework API
- device-specific поведение

```kotlin
@RunWith(AndroidJUnit4::class)
class UserDaoTest { /* ... */ }
```

::right::

UI-тесты проверяют сценарии пользователя.

Для Compose есть отдельные testing API:

```kotlin
composeTestRule
    .onNodeWithText("Login")
    .performClick()

composeTestRule
    .onNodeWithText("Welcome")
    .assertIsDisplayed()
```

> UI-тесты полезны, но их стоит писать на ключевые сценарии, а не на каждую кнопку.

---
layout: two-cols-header
---

## Mocking, fakes и screenshot tests

::left::

В тестах часто заменяют реальные зависимости.

| Подход | Идея |
|---|---|
| Fake | простая рабочая реализация для теста |
| Mock | объект, у которого проверяют вызовы |
| Stub | заранее подготовленные ответы |

```kotlin
class FakeUserRepository : UserRepository {
    override suspend fun getUser() = User("Test")
}
```

::right::

Screenshot tests фиксируют внешний вид UI.

Они помогают ловить:

- случайные изменения layout
- сломанные темы
- проблемы с локализацией
- отличия на разных размерах экранов

```text
render screen ──► compare with golden image
```

> Для Android важно тестировать не только «правильный ответ», но и то, как приложение выглядит и ведёт себя на устройстве.

---

# Публикация и сопровождение

---
layout: two-cols-header
---

## От кода до пользователя

::left::

Путь Android-приложения к пользователю выглядит примерно так:

```text
code
  │
  ▼
tests
  │
  ▼
release build
  │
  ▼
signing
  │
  ▼
AAB / APK
  │
  ▼
Google Play
```

::right::

Что важно на release-этапе:

- debug и release builds отличаются
- release-сборка должна быть подписана
- обычно публикуют AAB
- R8 / ProGuard уменьшают и оптимизируют код
- CI/CD помогает делать сборки воспроизводимыми

> Мобильное приложение нельзя просто «задеплоить на сервер»: оно устанавливается на устройства пользователей.

---
layout: two-cols-header
---

## Google Play Console

::left::

**Google Play Console** — основной инструмент публикации Android-приложений.

Через неё управляют:

- релизами
- описанием приложения
- скриншотами
- рейтингами и отзывами
- странами распространения
- тестовыми треками
- политиками и privacy-разделами

::right::

Типичные каналы релиза:

```text
internal testing
      │
      ▼
closed testing
      │
      ▼
open testing
      │
      ▼
production
```

> Публикация — это не только загрузить файл, но и пройти проверки платформы и управлять жизненным циклом релиза.

---
layout: two-cols-header
---

## Crash reporting и analytics

::left::

После публикации важно понимать, что происходит у пользователей.

**Crash reporting** отвечает на вопрос:

- где приложение падает?
- на каких устройствах?
- в какой версии?
- у скольких пользователей?

Популярный инструмент: **Firebase Crashlytics**.

::right::

**Analytics** помогает понимать поведение пользователей:

- какие экраны открывают
- где бросают сценарий
- какие функции используют
- как меняется конверсия после релиза

```text
release ──► users ──► crashes / events ──► fixes
```

> Без мониторинга разработчик узнаёт о проблемах слишком поздно — из отзывов и плохих оценок.

---
layout: two-cols-header
---

## Staged rollout и backward compatibility

::left::

**Staged rollout** — постепенный выпуск версии на часть аудитории.

```text
1% users ──► 5% ──► 25% ──► 50% ──► 100%
```

Если новая версия падает, rollout можно остановить.

Это снижает риск массовой поломки.

::right::

**Backward compatibility** — необходимость поддерживать разные устройства и версии Android.

Что приходится учитывать:

- minSdk и targetSdk
- разные размеры экранов
- разные производители устройств
- старые API
- миграции данных

> Android-разработка не заканчивается на написании кода: приложение нужно безопасно доставлять и поддерживать.

---

# Kotlin-фичи в Android-разработке

---
layout: two-cols-header
---

## Null-safety и модели данных

::left::

Android API и сеть часто возвращают неполные или отсутствующие данные.

Kotlin заставляет явно работать с `null`:

```kotlin
data class User(
    val id: Long,
    val name: String,
    val avatarUrl: String?,
)

val avatar = user.avatarUrl ?: DEFAULT_AVATAR_URL
```

`data class` удобны для DTO, UI state и domain-моделей.

::right::

`sealed` типы хорошо описывают состояния экрана.

```kotlin
sealed interface ProfileState {
    data object Loading : ProfileState
    data class Content(val user: User) : ProfileState
    data class Error(val message: String) : ProfileState
}
```

Такой код явно показывает все варианты состояния.

> Kotlin помогает сделать невалидные состояния менее вероятными.

---
layout: two-cols-header
---

## Lambdas, DSL и декларативный UI

::left::

Compose выглядит естественно именно на Kotlin: UI собирается как вложенный DSL.

```kotlin
@Composable
fun ProfileActions(
    onSave: () -> Unit,
    onLogout: () -> Unit,
) {
    Row {
        Button(onClick = onSave) {
            Text("Save")
        }
        TextButton(onClick = onLogout) {
            Text("Logout")
        }
    }
}
```

Lambdas позволяют передавать поведение как часть описания UI.

::right::

Та же идея используется в Gradle-конфигурации Android-проекта.

```kotlin
android {
    namespace = "com.example.app"
    compileSdk = 36

    defaultConfig {
        minSdk = 24
    }
}
```

---
layout: two-cols-header
---

## Coroutines и Flow

::left::

Android-приложение постоянно делает асинхронные операции:

- сетевые запросы
- чтение базы данных
- загрузка изображений
- синхронизация
- обработка пользовательских событий

```kotlin
viewModelScope.launch {
    val user = repository.loadUser(id)
    _state.value = ProfileState.Content(user)
}
```

Coroutines позволяют писать асинхронный код почти как обычный последовательный код.

::right::

`Flow` хорошо подходит для потоков данных и UI state.

```kotlin
val state: StateFlow<ProfileState> = repository
    .observeUser(id)
    .map { ProfileState.Content(it) }
    .stateIn(viewModelScope, SharingStarted.WhileSubscribed(), ProfileState.Loading)
```

Типичный поток:

```text
Room / Network ──► Flow ──► ViewModel ──► Compose UI
```

> Coroutines и Flow — одна из причин, почему Kotlin стал таким удобным для Android.

---
layout: two-cols-header
---

## Как Kotlin повлиял на Android

::left::

Kotlin не просто заменил Java в существующем Android-коде.

Он изменил дизайн новых Android API:

- Compose строится вокруг функций, lambdas и immutable state
- coroutines стали стандартным способом асинхронности
- Flow стал естественным форматом UI state и событий
- Gradle Kotlin DSL сделал сборку типизированной
- Hilt, Navigation, Room и другие библиотеки получили Kotlin-friendly API

::right::

Compose особенно сильно опирается на язык:

```kotlin
@Composable
fun ProfileScreen(state: ProfileState) {
    when (state) {
        ProfileState.Loading -> LoadingContent()
        is ProfileState.Content -> ProfileContent(state.user)
        is ProfileState.Error -> ErrorContent(state.message)
    }
}
```

Здесь сразу видны Kotlin-идеи:

- `sealed` состояния
- smart casts
- функции как UI-компоненты
- декларативная композиция

> Kotlin сделал современный Android более функциональным, декларативным и state-driven.

---
layout: two-cols-header
---

## Совместимость с Java

::left::

Android много лет развивался вокруг Java, поэтому совместимость критична.

Kotlin может:

- вызывать Java-код
- реализовывать Java-интерфейсы
- использовать старые Android-библиотеки
- постепенно заменять Java-файлы Kotlin-файлами

```kotlin
val intent = Intent(this, DetailsActivity::class.java)
```

::right::

Это позволяет мигрировать проект постепенно.

```text
old Java code
      │
      ├── Java Activity
      ├── Java Repository
      └── Kotlin ViewModel + Compose UI
```

В реальных компаниях часто живут смешанные кодовые базы.

> Kotlin стал успешен в Android не только из-за возможностей языка, но и из-за мягкой миграции с Java.

---

# Современные тренды

---
layout: two-cols-header
---

## Compose-first Android

::left::

Современная Android-разработка всё чаще начинается с Compose.

Что это меняет:

- UI пишется на Kotlin, без XML-layouts
- экран строится вокруг состояния
- проще делать Preview и UI-компоненты
- Material Design доступен как набор composable-компонентов
- меньше ручной связки View и кода

```text
UiState ──► @Composable screen
```

::right::

Но старый View-мир никуда не исчезает полностью:

- много существующих проектов написано на XML
- Compose и View можно смешивать
- миграция часто идёт постепенно
- RecyclerView, Fragment и XML всё ещё встречаются в реальных кодовых базах

> Тренд не в том, что старый Android исчез, а в том, что новый Android проектируют вокруг Compose и состояния.

---
layout: two-cols-header
---

## Kotlin Multiplatform

::left::

**Kotlin Multiplatform**, или KMP, позволяет переиспользовать Kotlin-код между платформами.

Обычно выносят общую бизнес-логику:

- модели данных
- сетевой слой
- кеширование
- валидацию
- use cases
- часть domain layer

```text
shared Kotlin code
 ├── Android app
 ├── iOS app
 └── desktop / web
```

::right::

Что обычно не шарят полностью:

- платформенный UI
- работу с permissions
- интеграцию с системными API
- специфичные возможности устройства

```text
commonMain ──► общая логика
androidMain ─► Android-specific код
iosMain ─────► iOS-specific код
```

> KMP не означает «один код для всего», скорее — общий Kotlin-слой там, где это действительно выгодно.

---
layout: two-cols-header
---

## Compose Multiplatform

::left::

**Compose Multiplatform** переносит идеи Compose за пределы Android.

На Compose можно писать UI для:

- Android
- Desktop
- iOS
- Web

Идея та же:

```text
state ──► declarative UI
```

::right::

Почему это важно для курса Kotlin:

- Kotlin становится языком не только Android, но и общей клиентской разработки
- один подход к UI может использоваться на нескольких платформах
- архитектура state-driven UI переиспользуется лучше, чем старый View-подход

Но есть ограничения:

- разная зрелость платформ
- разные нативные ожидания пользователей
- доступ к platform-specific API всё равно нужен

---
layout: two-cols-header
---

## Новые форм-факторы Android

::left::

Android давно не ограничивается обычными смартфонами.

Платформа живёт на разных устройствах:

- tablets
- foldables
- Wear OS
- Android Auto
- TV
- ChromeOS

Каждый форм-фактор меняет требования к UI.

::right::

Отсюда тренд на **adaptive UI**.

```text
phone ─────► one-pane layout
foldable ──► two-pane layout
tablet ────► master-detail layout
```

Разработчику приходится думать о:

- размере окна
- ориентации
- плотности экрана
- способе ввода
- сценарии использования устройства

> Хорошее Android-приложение сегодня должно быть готово не только к одному прямоугольнику смартфона.

---
layout: two-cols-header
---

## AI-инструменты в Android-разработке

::left::

AI-инструменты всё чаще становятся частью рабочего процесса разработчика.

Они помогают:

- генерировать UI-заготовки
- объяснять ошибки сборки
- писать тесты
- искать проблемы в логах
- мигрировать старый код
- подбирать API и примеры

::right::

Но в Android особенно важно проверять результат:

- lifecycle и permissions легко понять неправильно
- код может работать на одном устройстве и ломаться на другом
- generated UI не всегда учитывает accessibility
- зависимости и версии быстро меняются

```text
AI suggestion ──► review ──► run ──► test ──► ship
```

> AI ускоряет разработку, но не заменяет понимание платформы.

---

## Что важно вынести из лекции

- Android — большая платформа, а не просто «мобильная JVM».
- Kotlin стал одним из ключевых факторов современного Android.
- Compose изменил способ проектирования UI: состояние → интерфейс.
- Lifecycle, permissions, background limits и публикация — часть реальной разработки.
- Современный Android — это стек: Kotlin, Compose, Jetpack, Gradle, KMP и инструменты сопровождения.

> Если коротко: Android-разработка сегодня — это Kotlin-first, Compose-first и state-driven подход на сложной мобильной платформе.
