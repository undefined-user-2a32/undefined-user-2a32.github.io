---
layout: post
title: "Золоте правило Coroutine Exception"
date: 2025-10-04 23:39:00 +0300
tags: [Android, Переклад]
---
Дещо вільний переклад статті [The Golden Rule of Coroutine Exceptions](https://proandroiddev.com/the-golden-rule-of-coroutine-exceptions-8d4538158ab5)

---

По суті, обробка винятків у корутинах — це **структурований паралелізм**. Уявіть собі це як сімейне дерево. Якщо дочірня корутина падає з винятком, вона повідомляє про це батьківську корутину. Потім батьківська корутина скасовує всі інші дочірні корутини, а потім скасовує себе, передаючи виняток далі по дереву. Це гарантує, що жодна корутина ніколи не буде втрачена чи залишиться без нагляду.

Давайте заглибимося в різні сценарії, з якими ви зіткнетеся.

## Випадок 1: Білдер `launch`

`launch` — це білдер корутин «запустив і забув». Використовуйте його, коли вам не потрібен результат. Його поведінка у випадку винятків спочатку може здатись дещо складною.

### Некоректно: `try-catch` поза `launch`

Це поширена помилка. Огортання виклику `launch` в блок `try-catch` **не** перехоплюватиме винятки, що виникають всередині цього блоку.

- **Чому?**: Функція `launch` негайно завершується, а код усередині неї виконується в іншому потоці на фоні. Блок `try-catch` завершується ще до того, як виняток встигне статися. Виняток передається до обробника верхнього рівня, і ваш додаток аварійно завершить роботу, якщо його не обробити.

**Приклад**

```kotlin
import kotlinx.coroutines.*  

fun main() = runBlocking {  
    println("Before launch")  
    try {  
        // This launch block will throw an exception  
        launch {  
            println("Inside launch: Throwing exception...")  
            delay(500) // Simulate some work  
            throw RuntimeException("Something went wrong!")  
        }  
    } catch (e: Exception) {  
        // ❌ THIS BLOCK WILL NEVER BE REACHED ❌  
        println("Caught exception: $e")  
    }  
    println("After launch")  
    delay(1000) // Keep the main coroutine alive to see the crash  
    println("Main finished")  
}
```

**Вивід:**

```text
Before launch  
After launch  
Inside launch: Throwing exception...  
Exception in thread "main" java.lang.RuntimeException: Something went wrong!
... (stack trace) ...
```

Зверніть увагу, що повідомлення «Caught exception» ніколи не виводиться. Додаток просто крашиться.

### Коректно: `try-catch` всередині `launch`

Щоб обробити виняток, характерний для корутини `launch`, використовуйте блок `try-catch` безпосередньо всередині її лямбди.

- **Чому?**: Це розміщує логіку обробки винятків безпосередньо там, де вони можуть виникнути - у межах виконання корутини. Таким чином, ви можете коректно обробити виняток без зупинки всього скоупу.

**Приклад**

```kotlin
import kotlinx.coroutines.*  

fun main() = runBlocking {  
    println("Before launch")  
    launch {  
        try {  
            // The risky code is now inside the try block  
            println("Inside launch: Doing some work...")  
            delay(500)  
            throw RuntimeException("Oops, failed!")  
        } catch (e: Exception) {  
            // ✅ This block correctly catches the exception  
            println("Caught exception inside launch: ${e.message}")  
        }  
    }  
    println("After launch")  
    delay(1000) // Wait for the launch to complete  
    println("Main finished")  
}
```

**Вивід:**

```text
Before launch  
After launch  
Inside launch: Doing some work...  
Caught exception inside launch: Oops, failed!  
Main finished  
... (stack trace) ...
```

Тут програма коректно обробляє виняток і не призводить до збою.

## Випадок 2: Білдер `async`
`async` використовується, коли вам потрібно виконати завдання та отримати результат пізніше. Результат виконання огорнутий в об'єкт `Deferred`. `async` обробляє винятки по-іншому — він зберігає виняток у собі, доки ви не звернетесь по результат.

### Некоректно: `try-catch` огортає виклик `async` 

Так само, як і з `launch`, огортання в `try-catch` виклику `async` нічого не робить.

- **Чому?**: `async` миттєво повертає об'єкт `Deferred` . Виняток трапляється пізніше в корутині та зберігається всередині цього `Deferred` поки його не запитають про результат виконання.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    println("Before async")  
    val deferredResult: Deferred<String> = async {  
        println("Inside async: About to fail...")  
        delay(500)  
        throw IllegalStateException("Async operation failed!")  
        "This will never be returned"  
    }  
    println("After async")  
    try {  
        // The exception is thrown here, when we ask for the result  
        val result = deferredResult.await()  
        println("Result: $result")  
    } catch (e: Exception) {  
        // ✅ The exception is caught correctly here  
        println("Caught exception on await: ${e.message}")  
    }  
    println("Main finished")  
}
```

**Вивід:**

```text
Before async  
After async  
Inside async: About to fail...  
Caught exception on await: Async operation failed!  
Main finished  
Exception in thread "main" java.lang.IllegalStateException: Async operation failed!  
... (stack trace) ...
```

### Альтернатива: `try-catch` всередині `async`

- **Чому?**: `try-catch` всередині `async` може обробляти винятки локально, і на основі обробки можна повернути результат із самого блоку catch. У цьому випадку, коли повертається об'єкт `Deferred`, не буде винятку, оскільки його вже оброблено.

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    println("Before async")  
    val deferredResult: Deferred<String> = async {
		try {
			println("Inside async: About to fail...")  
			delay(500)  
			throw IllegalStateException("Async operation failed!")  
			"This will never be returned"  
		} catch (e: Exception){
		// ✅ The exception is caught correctly here  
		    println("Caught exception inside async: ${e.message}")
		     "Exception occurred inside aync"
	    }  
    }  
    println("After async")  
    try {  
        val result = deferredResult.await()  
        println("Result: $result")  
    } catch (e: Exception) {  
        // ❌ THIS BLOCK WILL NEVER BE REACHED ❌  
        println("Caught exception on await: ${e.message}")  
    }  
    println("Main finished")  
}
```

**Вивід:**

```text
Before async  
After async  
Inside async: About to fail...  
Caught exception inside async: Async operation failed!  
Result: Exception occurred inside aync  
Main finished
```

## Випадок 3: Зв'язки між батьками та дітьми (`coroutineScope`)

Саме тут структурований паралелізм справді проявляється. `coroutineScope` чекатиме на завершення всіх своїх дітей. Якщо будь-який дочірня корутина завершується невдачею, увесь скоуп негайно скасовує всі інші дочірні корутини, а потім і сам завершується з винятком.

### Помилка одного — помилка для всіх

- **Чому?**: Це запобігає роботі в неузгодженому стані. Якщо одна важлива частина операції збоїть, часто безпечніше скасувати всю операцію.

**Приклад**

```kotlin 
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    println("Starting the scope...")  
    try {  
        coroutineScope { // Create a new scope  
            launch {  
                try {  
                    println("Child 1: Working for 1000ms...")  
                    delay(1000)  
                    println("Child 1: Finished.") // This line will not be reached  
                } finally {  
                    println("Child 1: I was cancelled!")  
                }  
            }  
            launch {  
                println("Child 2: Working for 500ms then failing...")  
                delay(500)  
                throw RuntimeException("Child 2 failed!")  
            }  
        }  
    } catch (e: Exception) {  
        // The exception from Child 2 propagates up to the scope and is caught here  
        println("Caught exception in parent scope: ${e.message}")  
    }  
    println("Scope finished.")  
}
```

**Вивід:**

```text
Starting the scope...  
Child 1: Working for 1000ms...  
Child 2: Working for 500ms then failing...  
Child 1: I was cancelled!  
Caught exception in parent scope: Child 2 failed!  
Scope finished.  
```

Як бачите, коли Child 2 кинув виняток, Child 1 було скасовано перш ніж він встиг завершити свою роботу.

### Вкладений скоуп

Якщо ми додамо ще один скоуп корутини всередину існуючого скоупу, то він також буде скасований, як і будь-які інші завдання, що виконуються всередині батьківського скоупу корутини.

## Випадок 4: Ізоляція помилок (`supervisorScope`)

Що робити, якщо ви не хочете, щоб збій однієї корутини вплинув на інші корутини того ж скоупу? Використовуйте для цього `supervisorScope`.

- **Чому?**: `supervisorScope` перевизначає політику скасування батька. Виняток від прямого дочірнього елемента `supervisorScope` _не_ скасує батьківський скоуп і інші дочірні корутини. Це ідеально підходить для інтерфейсних програм, де виконуються незалежні завдання, і одна помилка не повинна приводити до зависання екрану.

**Важливе зауваження**: `supervisorScope` застосовується лише до _прямих дочірніх елементів_. Якщо ви використовуєте `coroutineScope` всередині `supervisorScope`, цей внутрішній `coroutineScope` все одно дотримуватиметься правила "скасувати все".

**Приклад**

```kotlin
import kotlinx.coroutines.*  

fun main() = runBlocking {  
    println("Starting the supervisor scope...")  
    try {  
        supervisorScope { // Children failures are isolated  
            launch {  
                println("Child 1: Working for 500ms then failing...")  
                delay(500)  
                throw RuntimeException("Child 1 failed!")  
            }  
            launch {  
                try {  
                    println("Child 2: Working for 1000ms...")  
                    delay(1000)  
                    println("Child 2: Finished successfully!") // This line IS reached  
                } finally {  
                    println("Child 2: I was NOT cancelled!")  
                }  
            }  
        }  
    } catch (e: Exception) {  
        // This catch block won't be hit because the supervisorScope  
        // doesn't propagate the child's exception upwards.  
        println("Caught exception in parent scope: $e")  
    }  
    println("Supervisor scope finished.")  
}
```

**Вивід:**

```text

Starting the supervisor scope...  
Child 1: Working for 500ms then failing...  
Child 2: Working for 1000ms...  
Exception in thread "main" java.lang.RuntimeException: Child 1 failed! // The exception is still thrown, but uncaught  
	... (stack trace) ...  
Child 2: Finished successfully!  
Child 2: I was NOT cancelled!  
Supervisor scope finished.
```

Зачекайте, програма все одно завершилась з помилкою! Чому? Тому що виняток від Child 1 не був оброблений _ніде_. `supervisorScope` запобігає скасуванню інших корутин скоупу, але він не ігнорує винятки. Вам все одно потрібно їх обробити

**Правильний спосіб використання** `supervisorScope`

Ви повинні обробляти винятки в дітях `supervisorScope`.

```kotlin
import kotlinx.coroutines.*

fun main() = runBlocking {  
    println("Starting the supervisor scope...")  
    supervisorScope {  
        // Child 1 with its own exception handling  
        launch {  
            try {  
                println("Child 1: I'm going to fail.")  
                delay(500)  
                throw RuntimeException("Child 1 failed!")  
            } catch (e: Exception) {  
                println("Caught in Child 1: ${e.message}")  
            }  
        }  
        // Child 2 runs independently  
        launch {  
            println("Child 2: I will succeed.")  
            delay(1000)  
            println("Child 2: Finished successfully!")  
        }  
    }  
    println("Supervisor scope finished.")  
}
```

**Вивід:**

```text
Starting the supervisor scope...  
Child 1: I'm going to fail.  
Child 2: I will succeed.  
Caught in Child 1: Child 1 failed!  
Child 2: Finished successfully!  
Supervisor scope finished.
```

Тепер все працює ідеально! Child 1 аварійно завершується та обробляє свою власну помилку, тоді як Child 2 виконує свою роботу без змін.

## Випадок 5: Глобальний перехоплювач винятків (`CoroutineExceptionHandler`)

Це ваша остання лінія захисту. `CoroutineExceptionHandler` — це спеціальний елемент контексту, який ви можете додати до скоупу верхнього рівня, щоб перехоплювати необроблені винятки.

- **Коли використовувати**: Він призначений в основному для логування, звітування про помилки або очищення необроблених винятків. Він найефективніший з `launch` у `CoroutineScope`, що використовує `SupervisorJob`, або в `GlobalScope`.
- **Коли він не працює**: Він не працюватиме для дочірніх корутин звичайного `coroutineScope`, оскільки батьківський об'єкт скасовується та сам обробляє свій виняток. Це не працюватиме для `async`, оскільки виняток інкапсолюється в об'єкті `Deferred` і кинеться лише при виклику `.await()`.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    // 1. Create the handler  
    val handler = CoroutineExceptionHandler { _, exception ->  
        println("Caught by CoroutineExceptionHandler: $exception")  
    }  
    // 2. Create a scope with a SupervisorJob and the handler  
    // A SupervisorJob is like a supervisorScope for a whole scope.  
    val scope = CoroutineScope(SupervisorJob() + handler)  
    // This launch will fail, and the handler will catch it  
    scope.launch {  
        println("Child 1: Failing...")  
        throw AssertionError("Something is wrong!")  
    }  
    // This launch will succeed, unaffected by the first one's failure  
    scope.launch {  
        delay(500)  
        println("Child 2: I'm alive!")  
    }.join() // wait for child 2 to finish for a clean exit  
    delay(1000) // Give time for the handler to run  
    println("Main finished.")  
}
```

**Вивід:**

```text
Child 1: Failing...  
Caught by CoroutineExceptionHandler: java.lang.AssertionError: Something is wrong!  
Child 2: I'm alive!  
Main finished.
```

Обробник перехопив помилку, а друга дочірня корутина не була скасована завдяки `SupervisorJob`.

## Випадок 6: `async` в `supervisorScope`

Ми дізналися, що `supervisorScope` запобігає тому, щоб збій однієї дочірньої корутини скасовував інші дочірні корутини скоупу Але як це працює з `async`? Правило для `async` залишається тим самим: виняток інкапсулюється в об'єкті `Deferred` і кидається лише при виклику `.await()`

- **Чому це важливо**: `supervisorScope` гарантує, що якщо одна операція `async` зазнає внутрішньої невдачі, інші дочірні корутини скоупу (як `launch` та `async`) продовжуватимуть виконуватися. Однак відповідальність за обробку винятку невдалої операції `async` все ще належить коду, який викликає `.await()`.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    println("Starting supervisor scope with async...")  
    supervisorScope {  
        // First async operation, destined to fail  
        val deferredFailure = async {  
            println("Async 1: I will fail in 500ms.")  
            delay(500)  
            throw IllegalStateException("Failure!")  
        }  
        // Second async operation, which will succeed  
        val deferredSuccess = async {  
            println("Async 2: I will succeed in 1000ms.")  
            delay(1000)  
            "Success!"  
        }  
        // Try to await the result of the failing async  
        try {  
            deferredFailure.await()  
        } catch (e: Exception) {  
            println("Caught expected failure from Async 1: ${e.message}")  
        }  
        // Awaiting the successful async works perfectly fine, it was not cancelled  
        try {  
            val result = deferredSuccess.await()  
            println("Result from Async 2: $result")  
        } catch (e: Exception) {  
            println("Caught unexpected failure from Async 2: $e")  
        }  
    }  
    println("Supervisor scope finished.")  
}
```

**Вивід:**

```text
Starting supervisor scope with async...  
Async 1: I will fail in 500ms.  
Async 2: I will succeed in 1000ms.  
Caught expected failure from Async 1: Failure!  
Result from Async 2: Success!  
Supervisor scope finished.
```

Це показує, що `deferredSuccess` не постраждав від винятку `deferredFailure`, але нам все одно довелося обробити виняток в місці виклику `await()`.

## Випадок 7: Скасування – це особливий вид винятку

Коли корутина скасовується, вона викидає виняток `CancellationException`. Це особливий виняток, який здебільшого ігнорується механізмами корутин.

- **Чому це важливо**: Виняток `CancellationException` сигналізує про те, що корутина була скасована як і очікувалося, що є нормальною частиною структурованого паралелізму. Хоча ви _можете_  перехопити його в блоці `try-catch`, зазвичай **не слід перехоплювати**. Якщо ви перехоплюєте його для виконання певної дії, вам слід прокинути його далі, щоб процес скасування міг завершитися належним чином. Найкраще місце для логіки очищення – це блок `finally`.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    val job = launch {  
        try {  
            println("Job: I'm working...")  
            delay(2000) // This is a suspension point  
            println("Job: I'm done.") // This will never be printed  
        } catch (e: CancellationException) {  
            // This is okay for logging, but you must re-throw it!  
            println("Job: I was cancelled. Re-throwing exception.")  
            throw e  
        } catch (e: Exception) {  
            println("Job: Caught some other exception: $e")  
        } finally {  
            // ✅ This is the correct place for cleanup logic  
            println("Job: Finally block executed for cleanup.")  
        }  
    }  
    delay(1000)  
    println("Main: I'm tired of waiting, cancelling the job.")  
    job.cancelAndJoin() // Cancel the job and wait for it to finish  
    println("Main: Job has been cancelled.")  
}
```

**Вивід:**

```text
Job: I'm working...  
Main: I'm tired of waiting, cancelling the job.  
Job: I was cancelled. Re-throwing exception.  
Job: Finally block executed for cleanup.  
Main: Job has been cancelled.
```

## Випадок 8:  Очищення з `NonCancellable`

Що робити, якщо ваш код очищення в `finally` також є suspend функцією (наприклад, запис у файл або мережевий виклик)? Якщо корутина вже скасована, будь-який новий виклик suspend функції всередині неї негайно кине `CancellationException`.

- **Рішення**: Щоб запустити suspend функцію в блоці очищення, потрібно переключитися на `NonCancellable` контекст. Це гарантує, що код всередині цього блоку буде виконано без скасування.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    val job = launch {  
        try {  
            println("Job: Working...")  
            delay(2000)  
        } finally {  
            println("Job: Entering finally block.")  
            // This suspending call would fail without NonCancellable  
            withContext(NonCancellable) {  
                println("Job: Starting crucial cleanup that takes 500ms...")  
                delay(500) // Simulate a suspending cleanup operation  
                println("Job: Crucial cleanup finished.")  
            }  
        }  
    }  
    delay(1000)  
    println("Main: Cancelling the job.")  
    job.cancelAndJoin()  
    println("Main: Job cancelled.")  
}
```

**Вивід:**

```text
Job: Working...  
Main: Cancelling the job.  
Job: Entering finally block.  
Job: Starting crucial cleanup that takes 500ms...  
Job: Crucial cleanup finished.  
Main: Job cancelled.
```

Незважаючи на те, що завдання було скасоване, код `delay(500)` всередині блоку `NonCancellable` зміг завершитися.

## Випадок 9: Вкладені скоупи та поширення винятків

Правила `coroutineScope` (fail-all) та `supervisorScope` (isolate-failure) стають ще цікавішими, коли ви їх вкладаєте. Головне пам'ятати, що правила застосовуються до _безпосередніх дітей_ цього скоупу.

### `coroutineScope` всередині `supervisorScope`

Виняток всередині внутрішнього `coroutineScope` скасує дочірні корутини свого скоупу, але він **не** скасує дітей зовнішнього `supervisorScope`.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    // Top-level scope that isolates its direct children's failures  
    supervisorScope {  
        // First child of supervisorScope, this will not be cancelled  
        launch {  
            delay(1000)  
            println("Supervisor's Child 1: I survived!")  
        }  
        // Second child of supervisorScope, which contains its own strict scope  
        coroutineScope {  
            launch {  
                delay(600)  
                // This sibling will be cancelled by the failure below  
                println("Inner Scope Sibling: I will be cancelled.")  
            }  
            launch {  
                delay(300)  
                println("Inner Scope Failing Child: I'm about to fail!")  
                throw RuntimeException("Failure in inner scope")  
            }  
        }  
    }  
    println("All done.")  
}
```


**Вивід:**

```text
Inner Scope Failing Child: I'm about to fail!  
Supervisor's Child 1: I survived!  
All done.  
Exception in thread "main" java.lang.RuntimeException: Failure in inner scope  
```

Зверніть увагу, що "Supervisor's Child 1" виводить своє повідомлення, доводячи, що скасування на нього не вплинуло. Але "Inner Scope Sibling" цього не робить, оскільки інша корутина всередині `coroutineScope` зазнала невдачі. Виняток все одно призводить до аварійного завершення, оскільки його не було перехоплено.

## Випадок 10: Детальний огляд ієрархії завдань 🌳

В основі структурованого паралелізму лежить концепція **ієрархії завдань**. Уявіть собі її як дерево завдань.

- Коли ви запускаєте корутину всередині іншої корутини (або скоупі), `Job` нової корутини стає **дочірнім** для батьківської `Job`.
- Батьківська `Job` має два ключові обов'язки:

1. Вона очікує завершення всіх своїх дочірніх `Job`, перш ніж зможе завершитись.
2. Якщо дочірня `Job` завершується невдачею з винятком (і це не `supervisorScope`), батьківська `Job` скасовує всіх інших дітей, а потім сама завершується з винятком.

Цей зв'язок між батьківською і дочірніми Job робить корутини «структурованими» та запобігає їхній втраті.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking { // This creates the root Job (parent)  
    println("Parent Scope: I'm the parent job.")  
    // Child Job 1  
    val job1 = launch {  
        println("  Child 1: I'm a child of the parent scope.")  
        delay(1000)  
        println("  Child 1: I'm done.")  
    }  
    // Child Job 2  
    val job2 = launch {  
        println("  Child 2: I'm also a child.")  
        delay(500)  
        // Let's create a grandchild  
        launch {  
            println("    Grandchild: My parent is Child 2.")  
            delay(500)  
            println("    Grandchild: I'm done.")  
        }  
        println("  Child 2: I'm done.")  
    }  
    println("Parent Scope: Waiting for my children to finish.")  
} // The runBlocking scope will not finish until job1 and job2 are complete.
```

**Вивід:**

```text
Parent Scope: I'm the parent job.  
Parent Scope: Waiting for my children to finish.  
  Child 1: I'm a child of the parent scope.  
  Child 2: I'm also a child.  
    Grandchild: My parent is Child 2.  
  Child 2: I'm done.  
    Grandchild: I'm done.  
  Child 1: I'm done.
```

Тут `runBlocking` — це або батьківська Job. Вона завершується лише після завершення обох `job1` та `job2.

## Випадок 11: `supervisorScope` проти `CoroutineScope(SupervisorJob())`

Це неочевидна, але важлива архітектурна відмінність.

- `supervisorScope { ... }`: Використовуйте цей **білдер** для створення локальної бульбашки, де помилки не поширюються. Він призначений для ізоляції _групи пов'язаних завдань_. Якщо всередині цього скоупу виникає необроблений виняток, він все ще _поширюється назовні_ блоку `supervisorScope`.
- `CoroutineScope(SupervisorJob())`: Ви створюєте **об'єкт скоупу** за допомогою цього. Усі корутини, запущені безпосередньо з цього скоупу, є незалежними. Цей шаблон ідеально підходить для компонентів з життєвим циклом, які керують кількома незалежними задачами верхнього рівня (наприклад, сервером або Android ViewModel). Збій в одній задачі не знищить сам скоуп і чи інші задачі.

**Приклад: Скоуп, подібний до ViewModel**
```kotlin
import kotlinx.coroutines.*  
  
// This class simulates a long-living component like an Android ViewModel or a server.  
class Component {  
    // Create a scope for this component. Its Job is a SupervisorJob.  
    // An uncaught exception will be handled by the handler, but the scope itself remains active.  
    private val scope = CoroutineScope(SupervisorJob() + CoroutineExceptionHandler { _, throwable ->  
        println("Component Scope: Caught an error: $throwable")  
    })  
    // This task might fail, but it won't affect other tasks in the scope.  
    fun startRiskyTask() {  
        scope.launch {  
            println("Risky Task: Starting...")  
            delay(500)  
            throw RuntimeException("Something went wrong!")  
        }  
    }  
    // This task can be started independently and will continue to run.  
    fun startStableTask() {  
        scope.launch {  
            println("Stable Task: I'm running...")  
            delay(1500)  
            println("Stable Task: I finished successfully.")  
        }  
    }  
    fun destroy() {  
        println("Component: Destroying and cancelling scope.")  
        scope.cancel() // Cancel all coroutines when the component is destroyed.  
    }  
}  
fun main() = runBlocking {  
    val component = Component()  
    component.startRiskyTask()  
    component.startStableTask()  
    delay(2000) // Let the tasks run for a while.  
    component.destroy()  
    delay(500)  
}
```

**Вивід:**

```text
Risky Task: Starting...  
Stable Task: I'm running...  
Component Scope: Caught an error: java.lang.RuntimeException: Something went wrong!  
Stable Task: I finished successfully.  
Component: Destroying and cancelling scope.
```

Незважаючи на те, що Risky Task завершується невдало, Stable Task успішно завершується, оскільки скоуп використовує SupervisorJob. Сам скоуп залишається активним до виклику destroy().

## Випадок 12: Обробка тайм-аутів ⏰

Тривалі операції можуть бути проблемою. Корутини надають простий спосіб обмежити час виконання.

- `withTimeout(timeMillis) { ... }`: Ця функція запускає блок коду та викидає виняток `TimeoutCancellationException`, якщо він не завершується протягом заданого часу. `TimeoutCancellationException` - це підклас `CancellationException`, тобто він скасовує корутину.
- `withTimeoutOrNull(timeMillis) { ... }`: Це альтернатива, що не кидає виняток. Якщо часовий ліміт перевищено, корутина скасовується та повертає `null`, а не кидає виняток. Часто це простіший спосіб.

**Приклад**

```kotlin
fun main() = runBlocking {  
    // Using withTimeout (throws exception)  
    try {  
        withTimeout(1000L) {  
            println("Task 1: I have 1 second to complete.")  
            delay(1500L) // This will take too long  
            println("Task 1: I finished.") // This won't be printed  
        }  
    } catch (e: TimeoutCancellationException) {  
        println("Task 1 failed: ${e.message}")  
    }  
    println("---")  
    // Using withTimeoutOrNull (returns null)  
    val result = withTimeoutOrNull(1000L) {  
        println("Task 2: I also have 1 second.")  
        delay(1500L) // Again, too long  
        "Success" // This value will not be returned  
    }  
    if (result == null) {  
        println("Task 2 timed out and returned null.")  
    } else {  
        println("Task 2 finished with result: $result")  
    }  
}
```

**Вивід:**

```text
Task 1: I have 1 second to complete.  
Task 1 failed: Timed out waiting for 1000 ms  
---  
Task 2: I also have 1 second.  
Task 2 timed out and returned null.
```

## Випадок 13: Винятки під час очікування кількох завдань

Що робити, якщо ви запустити кілька `async` та очікуєте на їхні результати? Функція розширення `awaitAll()` ідеально підходить для цього.

- **Поведінка**: Якщо будь-яке з завдань `async`, на яке ви очікуєте, зазнає невдачі, `awaitAll()` негайно скасує всі інші завдання та повторно викличе виняток для того, яке завершилося невдачею. Це спосіб дотримання принципу "all-or-nothing" як у `coroutineScope`.

**Приклад**

```kotlin
import kotlinx.coroutines.*  
  
fun main() = runBlocking {  
    println("Starting multiple async jobs.")  
    try {  
        coroutineScope { // A scope is needed to launch coroutines  
            val deferred1 = async {  
                delay(1000)  
                println("Job 1: Success.")  
                "Result 1"  
            }  
            val deferred2 = async {  
                delay(500)  
                println("Job 2: Failing!")  
                throw IllegalStateException("Job 2 Error")  
            }  
            val deferred3 = async {  
                delay(1200)  
                // This will be cancelled before it can complete  
                println("Job 3: Success.")  
                "Result 3"  
            }  
            // Wait for all of them to complete  
            val results = awaitAll(deferred1, deferred2, deferred3)  
            println("All results: $results")  
        }  
    } catch (e: Exception) {  
        println("Caught exception in awaitAll: ${e.message}")  
    }  
}
```

**Вивід:**

```text
Starting multiple async jobs.  
Job 2: Failing!  
Job 1: Success.  
Caught exception in awaitAll: Job 2 Error
```

Як бачите, у момент, коли завдання 2 зазнає невдачі, `awaitAll` кидає свій виняток. В результаті завдання 3 скасовується. Але завдання 1 вже було завершене, тому його повідомлення надруковано.

Отже, це все це стосується обробки винятків у корутинах. Сподіваюсь, що ця стаття була вам цікавою.
Якщо у вас виникли запитання, давайте конектитись на LinkedIn
[https://www.linkedin.com/in/devbaljeet/](https://www.linkedin.com/in/devbaljeet/)