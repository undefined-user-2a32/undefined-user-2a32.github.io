---
layout: post
title: "Життєвий цикл Android проти життєвого циклу AndroidX (KTX) — Під капотом"
date: 2025-10-11 00:04:00 +0300
tags: [Android, Переклад]
---
_Від колбеків до стейтмашини з безпечним API корутин._

![](https://miro.medium.com/v2/resize:fit:3200/1*XUIP2YAXP5F6H1c5Pgx1yw.png)

## 1) Зміна мислення: колбеки → стани

**Класичний ЖЦ** – це ланцюжок колбеків для `Activity`/`Fragment` (`onCreate`, `onStart`, …).

**ЖЦ AndroidX** - це модель ЖЦ будь-якого `LifecycleOwner` у вигляді **стейтмашини**, що надає API для **спостерігачів** і **корутин** для безпечного отримання даних та скасування підписки.

## Модель стану (AndroidX)

`INITIALIZED → CREATED → STARTED → RESUMED → (back down) → DESTROYED`

#todo
Події керують переходами; **стан** — це **де ми зараз знаходимось**. Відносно класичних колбеків це виглядає ось так:

![](https://miro.medium.com/v2/resize:fit:792/1*Wv0dKXcZAr2eaVAEQQY4gQ.png)

![](https://miro.medium.com/v2/resize:fit:1400/1*WhA0RhZTXGNmqB_PV0xA4w.png)

**ПРИМІТКА:** У AndroidX немає стану **PAUSED**

## 2) LifecycleOwner, Lifecycle та LifecycleRegistry (реалізація)

Будь-який компонент, що надає доступ до життєвого циклу, реалізує **`LifecycleOwner`**. `ComponentActivity` (батьківський для AppCompat) робить це, зберігаючи **`LifecycleRegistry`** та пересилаючи колбеки Activity як **події життєвого циклу**: `ON_CREATE`, `ON_START`, …, які змінюють стан реєстру.

> _Під капотом:_ `_LifecycleRegistry_` _зберігає поточний стан_, _`WeakReferenceвласника`_ та мапу обзерверів;  _`handleLifecycleEvent(event)`_ оновлює стан та надсилає зміни обезверам.

Чому `WeakReference`?

=> Щоб уникнути витоків памʼяті, якщо обзервери переживуть `LifecycleOwner`.

## 3) Обзервінг життєвого циклу: від анотацій до DefaultLifecycleObserver

Старий стиль:

```kotlin
class MyObserver : LifecycleObserver {

    @OnLifecycleEvent(Lifecycle.Event.ON_START)
    fun onStart() { /*...*/ }

    @OnLifecycleEvent(Lifecycle.Event.ON_STOP)
    fun onStop()  { /*...*/ }
}

lifecycle.addObserver(MyObserver())
```

`@OnLifecycleEvent` тепер **deprecated**. Використовуйте замість нього **`DefaultLifecycleObserver`** (методи Java 8 ) або **`LifecycleEventObserver`** (один колбек).

```kotlin
class MyObserver : DefaultLifecycleObserver {

    override fun onStart(owner: LifecycleOwner) { /*...*/ }

    override fun onStop(owner: LifecycleOwner)  { /*...*/ }
}
```

Обґрунтування для припинення підтримки та заміни.

## 4) Корутини + ЖЦ: два золотих інструменти

### A) `repeatOnLifecycle(state)`

Запускає ваш коду, коли життєвий цикл перебуває **як мінімум** в стані `state`; **скасовує** його, коли стан падає нижче; **перезапускає** при поверненні. Ідеально підходить для безпечної роботи Flow.

```kotlin
lifecycleScope.launch {
  repeatOnLifecycle(Lifecycle.State.STARTED) {
    vm.state.collect { render(it) }
  }
}
```

Обгрунтування такого дизайну API від авторів. ([Medium](https://medium.com/androiddevelopers/repeatonlifecycle-api-design-story-8670d1a7d333))

### B) `flowWithLifecycle(lifecycle, state)`

Оператор, який робить Flow залежним від ЖЦ **без** написання шаблону `repeatOnLifecycle`. Чудово підходить для _single_ Flow. Внутрішньо використовує `callbackFlow` та налаштовує контекст вищезазначеного потоку, додаючи семантику буферизації, подібну до `flowOn`.

```kotlin
viewLifecycleOwner.lifecycleScope.launch {  
  vm.state
    .flowWithLifecycle(viewLifecycleOwner.lifecycle, Lifecycle.State.STARTED)
    .collect(::render)
}
```

## 5) Fragment: `lifecycle` проти `viewLifecycleOwner.lifecycle`

Фрагменти мають **два** ЖЦ:

- ЖЦ самого **фрагмента** (існує від `onAttach` до `onDestroy`)
- ЖЦ **View** (існує від `onCreateView` до `onDestroyView`)

Для всього, що стосується **views** (байндинг, адаптери), збирайте дані в **`viewLifecycleOwner.lifecycle`** та визначайте скоуп через **`viewLifecycleOwner.lifecycleScope`**, щоб уникнути витоків, коли вʼю знищується, але фрагмент залишається.

```kotlin
viewLifecycleOwner.lifecycleScope.launch {
  repeatOnLifecycle(Lifecycle.State.STARTED) {
    vm.ui.collect(::renderOnView)
  }
}
```

## 6) Класичний підхід проти AndroidX підходу: що насправді змінюється в коді?

## Класичний спосіб (запуск/зупинка вручну)

```kotlin
override fun onStart() {  
  super.onStart()  
  subscribe() // start DB/network listener  
}  
  
override fun onStop() {  
  unsubscribe() // must remember to cancel!  
  super.onStop()  
}
```

## AndroidX + KTX спосіб (орієнтація на стан, без потреби ручного скасування)

```kotlin
lifecycleScope.launch {
  repeatOnLifecycle(Lifecycle.State.STARTED) {
    vm.updates.collect { render(it) } // auto-cancel/relaunch
  }
}
```

Цей спосіб унеможливлює **помилки контролю** та **витоки пам'яті**; це офіційно рекомендований шаблон для потоків, що залежать від ЖЦ.

## 7) Інтеграція з Compose (сучасний стек інтерфейсу користувача)

- **`LaunchedEffect(key)`** запускає корутину, прив'язану до композиції; вона скасовується, коли ефект **залишає композицію** (видаляється).
- Для роботи, яка повинна враховувати **ЖЦ власника** (наприклад, завдання лише в foreground), поєднуйте з `repeatOnLifecycle` овнера (Activity/Fragment) або використовуйте `flowWithLifecycle`. Гайд з документації корутин та ЖЦ Android.

## 8) Короткий огляд підкапотних механізмів

- **Хто змінює стан?** `ComponentActivity` пересилає кожен платформний колбек (`onStart`, …) до свого `LifecycleRegistry` через метод `handleLifecycleEvent`, який здійснює перехід стану та сповіщає спостерігачів.
- **Чому спостерігачі не спричиняють витоку пам'яті власників?** `Registry` зберігає `LifecycleOwner` у **WeakReference** (слабкому посиланні).
- **Чому `DefaultLifecycleObserver`?** Завдяки стандартним методам Java 8, інтерфейс може надавати порожні реалізації, дозволяючи вам перевизначати лише те, що потрібно; це також забезпечує кращу підтримку порівняно з анотаціями.
- **Семантика `repeatOnLifecycle`:** _Призупинити виконання до знищення; скасувати та перезапустити_ при переході нижче/вище порогового стану. Цей шаблон запобігає застаріванню колекторів після поворотів екрана/навігації.

## Усунення несправностей та граничні випадки (Перевірено на практиці)

- **Колектори спрацьовують після навігації / повороту екрана?** Ймовірно, ви використали `lifecycleScope` у Fragment для роботи з View. Перейдіть на використання **`viewLifecycleOwner.lifecycleScope`** та **`repeatOnLifecycle(STARTED)`**.
- **Плутаєтесь у `flowWithLifecycle` vs `repeatOnLifecycle`?** Віддавайте перевагу `flowWithLifecycle` для **одного** Flow; використовуйте `repeatOnLifecycle`, коли у вас є **кілька колекторів** в одному блоці або потрібен точний контроль. Офіційна документація демонструє обидва шаблони.
- **Попередження про deprecated `@OnLifecycleEvent`:** Замініть його на **`DefaultLifecycleObserver`** / **`LifecycleEventObserver`**. Перевірте release notes для Lifecycle.

## Шпаргалка на одній сторінці

**За замовчуванням використовуйте:**

**Робота зі станами (stateful collection) у Activity/Fragment**:

- `lifecycleScope.launch { repeatOnLifecycle(STARTED) { flow.collect { … } } }`

**Робота з View, у Fragment**:

- `viewLifecycleOwner.lifecycleScope.launch { repeatOnLifecycle(STARTED) { … } }`

**Single flow**:

- `flow.flowWithLifecycle(lifecycle, STARTED).collect { … }`

**Обзервери**: використовуйте **`DefaultLifecycleObserver`** (а не `@OnLifecycleEvent`).

## 11) Додаткові матеріали та ґрунтовна стаття про "внутрішні механізми"

- Офіційна документація та шаблони _Lifecycle_. [Android Developers](https://developer.android.com/topic/libraries/architecture/lifecycle)
- API довідка для `repeatOnLifecycle`. [Android Developers](https://developer.android.com/reference/androidx/lifecycle/RepeatOnLifecycleKt)
- Посібник: Корутини + ЖЦ (Flows, `flowWithLifecycle`). [Android Developers](https://developer.android.com/topic/libraries/architecture/coroutines)
- Примітка про deprecated `@OnLifecycleEvent`. [Android Developers](https://developer.android.com/jetpack/androidx/releases/lifecycle)
- Документація про жц View у Fragment (чому `viewLifecycleOwner`). [Android Developers](https://developer.android.com/guide/fragments/lifecycle)
- Огляд від спільноти про основи Lifecycle та `LifecycleRegistry`. [rommansabbir.com](https://rommansabbir.com/androidx-mastering-jetpack-lifecycle)
- Deep-dive `flowWithLifecycle` від Android DevRel. [Medium](https://medium.com/androiddevelopers/a-safer-way-to-collect-flows-from-android-uis-23080b1f8bda)

---

## TL;DR

- Сприймайте життєвий цикл як **дані** (стани), а не як **клубок колбеків**.
- Використовуйте **`repeatOnLifecycle(STARTED)`** або **`flowWithLifecycle`** для забезпечення безпечного скасування Flow.
- У Fragment віддавайте перевагу **`viewLifecycleOwner`** для всього, що взаємодіє з View.
- Реалізуйте **`DefaultLifecycleObserver`**, а не `@OnLifecycleEvent`.
