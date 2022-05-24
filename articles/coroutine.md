<h1>Памятка kotlin coroutine<h4/>
<h3>Решил собрать примеры практического использования корутин в котлине, чтобы не приходилось постоянно гуглить отдельные ответы на StackOverflow<h3/>
<br/>

Рассмотрим метод, который возвращает строку `"Hello world"` через ~1 секунду после вызова.<br/>
Представим, что этот метод обращается асинхронно к условному HelloMicroservice, а 1 секунда - задержка на ответ  
( `delay(1000)` не блокирует основной поток)
```kotlin
suspend fun runSuspendFun(): String {
    delay(1000)
    return "Hello world"
}
```
<br/>
Рассмотрим также три последовательных вызова этого метода<br/>

```kotlin
suspend fun notParallel() {
    runSuspendFun()
    runSuspendFun()
    runSuspendFun()
}
```
Так как по умолчанию suspend функции в kotlin последовательные и ни о каком параллелизме речи не идет,
то выполнение займет ~3 секунды.<br/>
Не круто - хотелось бы делать это параллельно. 
<br/><br/><br/>
В этом нам помогут coroutine builder'ы:<br/>
`launch {...}` - "fire and forget coroutine" (запустил корутину, не важен результат) <br/>
`async {...}` - "start a coroutine that computes some result" (запустил корутину, подожди результат)
<br/><br/><br/>
Способ параллельного вызова в случае, когда не требуется результат корутин  
Например, метод `runSuspendFun()` что-то сохраняет, используя POST запрос. Такой способ займет ~1 секунду
```kotlin
suspend fun parallelWithLaunch(): Unit = coroutineScope {
    launch { runSuspendFun() }
    launch { runSuspendFun() }
    launch { runSuspendFun() }
    println("Hello world")
}
```
Каждый блок `launch {...}` запускает корутину. Код внутри каждого `launch {...}` выполняется одновременно с другими<br/><br/>
Следует обратить внимание, что `launch {...}` не блокирует основной поток, 
а метод `runSuspendFun()` выполняется ~1 секунду, поэтому сначала выполнится `println("Hello world")`
(в нашем случае это не важно, так как не важен результат метода `runSuspendFun()`)
 
<br/><br/>
Способ параллельного вызова в случае, когда требуется результат корутин <br/>
Например, необходимо получить что-то с трех сервисов параллельно. Такой способ займет ~1 секунду
```kotlin
suspend fun parallelWithAsync(): Unit = coroutineScope {
    val deferredA = async { runSuspendFun() }
    val deferredB = async { runSuspendFun() }
    val deferredC = async { runSuspendFun() }

    doSomething(deferredA.await())
    doSomething(deferredB.await())
    doSomething(deferredC.await())
    println("Hello world")
}
```
Каждый блок `async {...}` запускает корутину. Код внутри каждого `async {...}` выполняется одновременно с другими.
В отличие от примера выше, строка кода `println("Hello world")` будет ждать все `await()` и только потом выполнится.
<br/><br/>

<h3>Существует особенность при итерации и запуске корутин<h4/>
Если запустить такой код, то метод будет выполняться ~3 секунды <br/>
(зависит от количества элементов, по одной секунде на каждый элемент)
```kotlin
suspend fun notParallelMap() {
    val list = listOf("foo", "bar", "buz")
    list.map { runSuspendFun() }
        .forEach { println(it) }
}
```
Или цикл for в стиле foreach (будем ждать по ~1 секунде на каждой итерации)<br/>
В итоге ~3 секунды на выполнение (зависит от количества элементов)
```kotlin
suspend fun notParallelForEach() {
    val list = listOf("foo", "bar", "buz")
    for (s in list) {
        println(runSuspendFun())
        println(s)
    }
}
```
Не круто зависнуть, если в коллекции сотни или тысячи элементов.
<br/><br/><br/>
Решение для первого случая - `async()` + `awaitAll()`, где `map()` элементов коллекции выполнялся параллельно

```kotlin
suspend fun parallelMap(): Unit = coroutineScope {
    val list = listOf("foo", "bar", "buz")
    list.map { async { runSuspendFun() } }
        .awaitAll()
        .forEach { println(it) }
}
```
<br/>

Решение для второго случая - запуск корутины на каждой итерации в блоке `launch()`<br/>

```kotlin
suspend fun notParallelForEach() = coroutineScope {
    val list = listOf("foo", "bar", "buz")
    for (s in list) {
        launch {
            println(runSuspendFun())
            println(s)
        }
    }
}
```
Теперь методы будут выполняться ~1 секунду независимо от количества элементов в коллекции.  
Поэтому следует помнить об этом при запуске корутин в циклах или итерируясь по коллекции

#Заключение
Надеюсь вам это поможет перестать постоянно лазить за отдельными примерами в StackOverflow и GitHub.