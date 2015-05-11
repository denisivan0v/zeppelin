---
layout: post
title: "Reactive Extensions overview"
date: 2015-05-08 13:00:00
isStaticPost: false
categories: [reactive extensions observable subject rx]
---

#### Что такое Reactive Extensions

Итак, продолжая тему Reactive, пробежимся по библиотеке [Reactive Extensions][rx]. Как следует из первой же строки описания проекта, Reactive Extensions (Rx) - это библиотека для создания асинхронных, основанных на событиях, программ с помощью observable-последовательностей и LINQ-style операторов. Используя Rx, мы **представляем** асинхронные потоки данных абстракцией [Observables][observables], делаем к ним **запросы** посредством [LINQ-операторов][linq-operators] и **параметризуем** конкурентный доступ к асинхронных потокам данных с помощью [Планировщиков (Schedulers)][schedulers]. Итого, Rx = Observables + LINQ + Schedulers.


#### В каких случаях мне использовать Rx?

*Далее я буду использовать довольно много текста из книги Lee Campbell "Introduction to Rx", онлайн-версия которой доступна [по ссылке][introtorx].*

Если вы работаете с предметной областью, где приходится оперировать с **последовательностью событий**, то Rx - то, о чем неплохо бы вспомнить.

**Нужно использовать Rx**

Rx создана для управления такими видами событий как:

* UI-события, то есть при разработке клиентских приложений
* Доменные события, такие как "Заказ оформлен", "Заказ утвержден", "Списание средств выполнено" и т.п. К этому приходят, например, применяя практику [проектирования по предметной области (Domain-Driven Design, DDD)][ddd]
* Всяческие инфраструктурные события, источником которых может быть filewatcher, WMI, ETW и т.п.
* События из broadcast-источников, таких как шины сообщений, всевозможные датчики (тут можно вспомнить про модные сейчас [IoT][iot]-устройства) и прочее
* События из [Complex event processing (CEP)][cep] сервисов

**Можно использовать Rx** для асинхронных вызовов методов, возвращающих `Task` или `Task<T>`, т.к. это есть ничто иное, как последовательность из одного элемента. Возможно также использование и с APM-методами вида `BeginXXX`/`EndXXX`.

**Не стоит использовать Rx** для выполнения задач, которые успешно решаются с помощью `IEnumerable<T>`, тем самым заменяя pull-модель взаимодействия push-моделью.

#### Основы Rx

Как я отмечал [в предыдущем посте][the-basics], в основе Rx лежит паттерн Observer. В Rx есть два базовых интерфейса:

```c#
public interface IObservable<out T>
{
	IDisposable Subscribe(IObserver<T> observer);
}

```

и

```c#
public interface IObserver<in T>
{
	void OnNext(T value);
	void OnError(Exception error);
	void OnCompleted();
}
```

Все, что реализует `IObservable<T>` стоит воспринимать как *поток* объектов типа T. Не нужно путать `IObservable<T>` с также существующей в .NET абстракцией `Stream` из неймспейса `System.IO`, т.к. последняя поддерживает возможность чтения и записи, а также "просмотра" потока (seek). Тем не менее, в Rx присутствуют концепции следования по потоку (push), освобождения ресурсов (закрытия) и завершения работы с потоком (EOF). Однако, кроме этого Rx предоставляет возможность конкурентного доступа, трансформации потока, слияния потоков, агрегации и расширения (expanding). 

Важно также, что observables - это не коллекции в привычном понимании. Значения из `IObservable<T>` обычно не материализованы, как этого можно ожидать от коллекции. Таким образом, observables - это *последовательности*. Чтобы не путать их с последовательностями `IEnumerable<T>` Lee Campbell предлагает воспринимать экземпляры `IEnumerable<T>` как данные **в статичном состоянии**, в то время как экземпляры `IObservable<T>` - **данные в движении**.

Также важно упомянуть, что `IObservable<T>` является функционально сопряженным (functional dual) к `IEnumerable<T>`. Напомню, как выглядят `IEnumerable<T>`

```c#
public interface IEnumerable
{
    IEnumerator GetEnumerator();
}

public interface IEnumerable<out T> : IEnumerable
{
    new IEnumerator<T> GetEnumerator();
}
```

и `IEnumerator<out T>`

```c#
public interface IEnumerator
{
    bool MoveNext();
    Object Current { get; }
    void Reset();
}

public interface IEnumerator<out T> : IDisposable, IEnumerator
{    
    new T Current { get; }
}
```

Для тех, кто помнит функциональный анализ и хочет поглубже разобраться, рекомендую посмотреть это [видео на Channel 9][inside-rx]. 
Если вкратце, то это значит, что также как `IEnumerable<T>` позволяет реализовать три свойства - следующее значение (`MoveNext()` и `Current`), обработку исключений (обычный try/catch/finally и `Dispose()`) и окончание последовательности (`MoveNext()`, возвращающий `false`) с помощью `IEnumerator<T>`, так же и `IObservable<T>` позволяет сделать то же самое с помощью `IObserver<T>` - методы `OnNext(T)`, `OnError(Exception)` и `OnCompleted()` соотвественно.

Rx предоставляет четкий контракт, которому необходимо следовать, если вы решите реализовывать базовые интерфейсы самостоятельно. В такой реализации стоит учитывать, что опционально может быть вызван метод `OnNext(T)` и далее возможен (но не обязятелен) вызов одного из `OnError(Exception)` или `OnCompleted()`. То есть, если последовательность завершается, то обязательно вызовом либо `OnError(Exception)`, либо `OnCompleted()`. Однако, ни один из этих трех методов может быть и не вызван вовсе, что дает возможность реализации пустых или бесконечных последовательностей.


###### Subject

Rx конечно же предоставляет реализацию этих базовых интерфейсов, тем самым избавляя нас от необходимости делать это самостоятельно - это [тип][subject] `Subject<T>`. Этот тип реализует сразу оба интерфейса, предоставляя вызывающему коду целостно использовать концепцию publisher/subscriber (или publisher/consumer). Вот пример:

```c#
var subject = new Subject<string>();
subject.Subscribe(Console.WriteLine);
subject.OnNext("a");
subject.OnNext("b");
subject.OnNext("c");
```

Это рабочий код, так как в Rx есть множество extension-методов, в том числе метод `Subscribe(Action<T>)`. В результате работы кода на консоль будет выведено `abc`.

Но `Subject<T>` - это самая простая имплементация. Существуют также еще три, существенно меняющие поведение при использовании каждой из них.

###### ReplaySubject

`ReplaySubject<T>` - [реализация][replay-subject], позволяющая "повторить" события для подписчиков, которые начали слушать последовательность не с начала. `ReplaySubject<T>`, созданный при помощи конструктора по-умолчанию кэширует все значения и воспроизводит их для всех подписчиков. Однако, это может привести к memory pressure, поэтому сущеуствует конструктор, позволяющий задать размер буфера, то есть количество кэшируемых значений. Код ниже выведет на консоль `bc`. Даже, если до вызова `Subscribe(Action<T>)` вызвать `OnCompleted()`.

```c#
var subject = new ReplaySubject<string>(2);
subject.OnNext("a");
subject.OnNext("b");
subject.OnNext("c");
subject.Subscribe(Console.WriteLine);
```

Существуют также конструкторы, позволяющие задать временно окно для кэширования, а также и то, и другое.

###### BehaviorSubject

`BehaviorSubject<T>` - [реализация][behavior-subject], запоминающяя последнее опубликованное значение. Для `BehaviorSubject<T>` требуется задать значение `T` по-умолчанию. Отличие от `ReplaySubject<T>` в том, что, во-первых, размер буфера равен 1 и, во-вторых, ничего не будет получено, если подписка была выполнена *после* завершения последовательности, то есть, например, вызова `OnCompleted()`. Код ниже выведет на консоль `bc`

```c#
var subject = new BehaviorSubject<string>("x");
subject.OnNext("a");
subject.OnNext("b");
subject.Subscribe(Console.WriteLine);
subject.OnNext("c");
```

А этот код ничего не выведет

```c#
var subject = new BehaviorSubject<string>("x");
subject.OnNext("a");
subject.OnNext("b");
subject.OnCompleted();
subject.Subscribe(Console.WriteLine);
subject.OnNext("c");
```

###### AsyncSubject

`AsyncSubject<T>` - [реализация][async-subject], очень похожая на `BehaviorSubject<T>`. Она также кэширует последнее значение, однако публикует его только по окончанию последовательности. То есть этот код ничего не выведет

```c#
var subject = new AsyncSubject<string>();
subject.OnNext("a");
subject.OnNext("b");
subject.Subscribe(Console.WriteLine);
subject.OnNext("c");
```

а этот код выведет `c`

```c#
var subject = new AsyncSubject<string>();
subject.OnNext("a");
subject.OnNext("b");
subject.Subscribe(Console.WriteLine);
subject.OnNext("c");
subject.OnCompleted();
```

###### Интерфейс ISubject

Описанные выше типы реализуют `IObservable<T>` и `IObserver<T>` через специальный интерфейс 

```c#
public interface ISubject<in TSource, out TResult> : IObserver<TSource>, IObservable<TResult>
{
}
```

Точнее даже через интерфейс с одинаковыми типами `TSource` и `TResult`

```c#
public interface ISubject<T> : ISubject<T, T>, IObserver<T>, IObservable<T>
{
}
```

Важно заметить, что все четыре реализации, описанные выше реализуют лишь контракт `ISubject<T>` и не имеют общих базовых классов.

Также стоит упомянуть, что существует фабричный метод 

 ```c#
 public static ISubject<TSource, TResult> Create<TSource, TResult>(
 	IObserver<TSource> observer, 
	IObservable<TResult> observable)
{...}
 ```

позволяющий создать `ISubject<TSource, TResult>` из имеющихся `IObserver<TSource>` и `IObservable<TResult>`.

#### Summary

В этом посте мы пробежались по самой сути Rx, а также посмотрели на имеющиеся реализации базовых контрактов. Однако, в продакшен коде очень редко придется использовать интерфейс `IObserver<T>` и subject-типы. Тем не менее, их знание и понимание принципов работы очень важно для правильного использования Rx. Напротив, интерфейс `IObservable<T>` очень широко используется в Rx в качестве возвращаемого значения, представляя тем самым последовательность данных в движении. Для этого интерфейса реализовано огромное количество extension-методов для возможности использования Rx в LINQ-стиле для оперирования этими последовательностями. Что это за возможности - рассмотрим одном из следующих постов. 

Stay tuned!

[rx]: https://github.com/Reactive-Extensions/Rx.NET
[observables]: http://msdn.microsoft.com/library/dd990377.aspx
[linq-operators]: http://msdn.microsoft.com/en-us/library/hh242983.aspx
[schedulers]: http://msdn.microsoft.com/en-us/library/hh242963.aspx
[introtorx]: http://introtorx.com
[ddd]: http://en.wikipedia.org/wiki/Domain-driven_design
[iot]: http://en.wikipedia.org/wiki/Internet_of_Things
[cep]: http://en.wikipedia.org/wiki/Complex_event_processing
[the-basics]: /blog/the-basics-of-the-reactive
[inside-rx]: http://channel9.msdn.com/Shows/Going+Deep/Expert-to-Expert-Brian-Beckman-and-Erik-Meijer-Inside-the-NET-Reactive-Framework-Rx
[subject]: http://msdn.microsoft.com/en-us/library/hh229173(v=VS.103).aspx
[replay-subject]: http://msdn.microsoft.com/en-us/library/hh211810(v=VS.103).aspx
[behavior-subject]: http://msdn.microsoft.com/en-us/library/hh211949(v=VS.103).aspx
[async-subject]: http://msdn.microsoft.com/en-us/library/hh229363(v=VS.103).aspx