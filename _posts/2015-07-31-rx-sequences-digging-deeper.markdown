---
layout: post
title: "Digging deeper into Rx sequences"
date: 2015-07-31 22:40:00
isStaticPost: false
categories: [reactive extensions rx sequences create extension methods]
---

Перейдём к более детальному обзору последовательностей в Rx. Тут будут обязательны знания LINQ, т.к. все построено на extension-методах к базовым абстракциям Rx (query-операторах) и вспомогательных статических типах. Так же не обойдется без знания основ функционального программирования. 

#### Создание последовательностей
Пройдемся по фабричным методам для создания observable-последовательностей. Эти методы обычно получают в качестве параметра значение (экземпляр) некоторого типа, либо сам тип. С точки зрения функционального программирования это есть ничто иное, как [*анаморфизм*][anamorphism], или функция *unfold*. Далее я перечислю фабричные методы рядом с их реализацией с использованием  `Subject`.

**Observable.Return**

```c#
IObservable<int> observable = Observable.Return(1);
```

По поведению эквивалентно следующему:

```c#
var subject = new ReplaySubject<int>();
subject.OnNext(1);
subject.OnCompleted();
```

**Observable.Empty**

```c#
IObservable<int> observable = Observable.Empty<int>();
```

По поведению эквивалентно следующему:

```c#
var subject = new ReplaySubject<int>();
subject.OnCompleted();
```

**Observable.Never**

```c#
IObservable<int> observable = Observable.Never<int>();
```

По поведению эквивалентно следующему (бесконечная последовательность):

```c#
var subject = new ReplaySubject<int>();
```

**Observable.Throw**

```c#
IObservable<int> observable = Observable.Throw<int>(new Exception());
```

По поведению эквивалентно следующему:

```c#
var subject = new ReplaySubject<int>();
subject.OnError(new Exception());
```

**Observable.Create**

Метод `Create` - рекомендуемый способ создания observable-последовательностей. На это есть две причины:
 
1. Используя `Subject`, нам необходимо самостроятельно контролировать его состояние, а также время жизни, что может быть непросто в асинхронных сценариях.
2. Метод `Create` не блокирующий и "лениво" создает последовательность, в отличие от сценария с использованием `Subject`.

Вот сигнатуры метода `Create`:

```c#
IObservable<TSource> Create<TSource>(Func<IObserver<TSource>, IDisposable> subscribe);
IObservable<TSource> Create<TSource>(Func<IObserver<TSource>, Action> subscribe);
```

Пункт 2 в списке причин достигается за счет передачи делегата, который будет исполнен только после создания подписки:

```c#
public static class SequenceFactory
{
    public static IObservable<int> Create()
    {
        return Observable.Create((IObserver<int> observer) =>
        {
            // этот код будет исполнен только после создания подписки
            observer.OnNext(1);
            observer.OnCompleted();
            return Disposable.Create(() =>
            {
                // этот код выполнится после вызова Dispose у созданной подписки
                Console.WriteLine("Подписчик завершил работу");
            });
        });
    }
}
```

Сценарий же с использованием `Subject` не только "более императивен", но и возвратит завершенную последовательность (можно использовать `ReplaySubject` для получения того же результата, т.к. значения будут закэшированы, однако отписку придется контролировать вызывающему коду): 

```c#
public static class SequenceFactory
{
    public static IObservable<int> CreateUsingSubject()
    {
        var subject = new Subject<int>();
        subject.OnNext(1);
        subject.OnCompleted();
        return subject;
    }
}
```

#### Unfold и корекурсия

[*Unfold*][anamorphism] (наверное, можно перевести как развёртка (от слова "развёртывать")) - функция высшего порядка, позволяющая строить следующий элемент, используя начальное значение и вызывая переданную функцию. 

[*Корекурсия*][corecursion] - функция, позволяющая получить следующее значение последовательности, используя текущее.

Можно реализовать pull-вариант функции *unfold* с использованием `IEnumerable<T>` и корекурсии:

```c#
IEnumerable<T> Unfold<T>(T seed, Func<T, T> accumulator)
{
    var nextValue = seed;
    while (true)
    {
        yield return nextValue;
        nextValue = accumulator(nextValue);
    }
}
```

С помощью этой фукнции можно получить, например, массив натуральных чисел:

```c#
var values = Unfold(1, x => x + 1).Take(10).ToArray();
```

Посмотрим, как это можно сделать в push-варианте с помощью Rx.

**Observable.Range**

Этот метод похож на одноименный метод статического класса `Enumerable`, но возвращает `IObservable<T>`. Чтобы получить значения, нужно начать "наблюдать" результат, создав подписку:

```c#
var range = Observable.Range(1, 10);
range.Subscribe(Console.WriteLine);
```

**Observable.Generate**

Это и есть реализация функции *unfold* в Rx. Вот сигнатура метода:

```c#
IObservable<TResult> Generate<TState, TResult>(TState initialState, 
                                               Func<TState, bool> condition, 
                                               Func<TState, TState> iterate, 
                                               Func<TState, TResult> resultSelector);
```

Параметры метода по порядку:

* начальное значение
* предикат для завершения последовательности
* корекурсивная логика
* функция для преобразования результата

`Observable.Range<int>` может быть реализован с помощью `Observable.Generate` следующим образом:

```c#
IObservable<int> Range(int start, int count)
{
    var max = start + count;
    return Observable.Generate(
        start, 
        value => value < max, 
        value => value + 1, 
        value => value);
}
```

**Observable.Interval**

Метод для получения последовательности целых чисел, начиная с 0, через заданный промежуток времени.

`Observable.Interval` также может быть реализован с помощью `Observable.Generate` следующим образом:

```c#
public static IObservable<long> Interval(TimeSpan period)
{
    return Observable.Generate(
        0l,
        i => true,
        i => i + 1,
        i => i,
        i => period);
}
```

Тут использована еще одна перегрузка `Observable.Generate` с параметром для создания timer related-последовательностей:

```c#
IObservable<TResult> Generate<TState, TResult>(TState initialState,
                                               Func<TState, bool> condition, 
                                               Func<TState, TState> iterate, 
                                               Func<TState, TResult> resultSelector, 
                                               Func<TState, TimeSpan> timeSelector);
```

Как видно из примера выше, `Observable.Interval` создает бесконечную последовательность для подписчика, поэтому важно не забывать про `Dispose`.

**Observable.Timer**

Сигнальный метод, который публикует только одно значение (равное 0) после прошествия заданного промежутка времени, задаваемого с помощью параметра типа `TimeSpan`. Последовательность завершается сразу, однако если воспользоваться перегрузкой с типом параметра `DateTimeOffset`, то последовательность завершится в указанное время. 

Есть и третья перегрузка с двумя параметрами типа `TimeSpan` для создания бесконечных временных последовательностей.

`Observable.Timer` также может быть реализован с помощью `Observable.Generate` следующим образом:

```c#
IObservable<long> Timer(TimeSpan dueTime)
{
    return Observable.Generate(
        0l,
        i => i < 1,
        i => i + 1,
        i => i,
        i => dueTime);
}
```

```c#
IObservable<long> Timer(TimeSpan dueTime, TimeSpan period)
{
    return Observable.Generate(
        0l,
        i => true,
        i => i + 1,
        i => i,
        i => i == 0 ? dueTime : period);
}
```

В свою очередь `Observable.Interval` может быть реализован с использованием `Observable.Timer`:

```c#
IObservable<long> Interval(TimeSpan period)
{
    return Observable.Timer(period, period);
}
```

#### Переход к push-модели

Возможно также создать observable-последовательность, выполнив переход из существующего синхронного или асинхронного императивного кода.

**Observable.Start**

Метод позволяет создать из экземпляра `Func<T>` или `Action` observable-последовательность с одним элементом, обработка которого будет выполнена асинхронно потоком из `ThreadPool`. В случае `Action` возращаемым значением `Observable.Start` является `IObservable<Unit>`. Тип `Unit` - конструкция из мира функционального программирования, представляющая собой нечно наподобие `void`. `Unit` не имеет значения и служит для передачи в метод `OnNext`:

```c#
IObservable<Unit> start = Observable.Start(
    () =>
    {
        Console.Write("Working away");
        for (int i = 0; i < 10; i++)
        {
            Thread.Sleep(100);
            Console.Write(".");
        }
    });

start.Subscribe(
  unit => Console.WriteLine("Unit published"), 
  () => Console.WriteLine("Action completed"));
```

Метод `Observable.Start` очень напоминает `Task`, т.к. позволяет создать значение, которое будет вычислено позже. Это отличает этот метод от `Observable.Return`, который возвращает observable-последовательность с одним заданным (вычисленным) элементом. 

**Observable.FromEventPattern**

Метод позволяет перейти к push-модели из существующей в .NET модели событий (event). Есть много перегрузок этого метода, чтобы иметь возможность использовать его во всех случаях (обычный `EventHandler`/специфичный наследник `EventHandler` и специфичный наследник `EventArgs`/ обобщенный (generic) `EventHandler<TEventArgs>`). Вот пример для второго случая - `UnhandledExceptionEventHandler` и `UnhandledExceptionEventArgs`:

```c#
IObservable<EventPattern<UnhandledExceptionEventArgs>> unhandledExceptionObservable =
    Observable.FromEventPattern<UnhandledExceptionEventHandler, UnhandledExceptionEventArgs>(
        x => AppDomain.CurrentDomain.UnhandledException += x,
        x => AppDomain.CurrentDomain.UnhandledException -= x);

```

**ToObservable()**

У `AsyncSubject<T>`, о котором я писал в [прошлом посте][latest-post], много общего с `Task<T>`. Они оба возвращают одно значение из асинхронного источника, кэшируют результат для последующих обращений. Одна из перегрузок метода `ToObservable()` - extension-метод к типу `Task<T>`, который работает следующим образом:

* если task в статусе `RanToCompletion`, значение добавляется в последовательность, после чего она завершается
* если task в статусе `Cancelled`, будет опубликована ошибка с экземпляром `TaskCanceledException`
* если task в статусе `Faulted`, будет опубликована ошибка внутренним исключением task-а
* если task еще не завершен, к task-е добавляется continuation, чтобы выполнить перечисленное выше


Вот пример:

```c#
var task = Task.Run(() => "Task started");
var observable = task.ToObservable();
observable.Subscribe(Console.WriteLine, () => Console.WriteLine("Observable completed"));
```

На консоль будет выведено

```
Task started
Observable completed
```

Еще одна перегрузка метода `ToObservable()` - extension-метод к типу `IEnumerable<T>`, который семантически использует `Observable.Create`, передавая в качестве делегата функцию, в которой перечисляются все элементы исходной последовательности. При использовании этого метода стоит помнить, что блокирующая синхронная pull-модель `IEnumerable<T>` противоположна асинхронной push-модели `IObservable<T>` и хорошо продумывать сценарии использования этого метода.

**Observable.FromAsync**

Cуществовавший до .NET 4.5 async pattern (пара методов BeginXXX/EndXXX), известный также как Asynchronous Programming Model (APM), был поддержан в Rx с помощью метода `Observable.FromAsyncPattern`, который на текущий момент в статусе *obsolete*, как и весь APM. Пришедший ему на замену способ реализации асинхронных сценариев с возвращаемым значением в виде экземпляра `Task` или `Task<T>` также поддержан в Rx посредством метода `Observable.FromAsync`. Вот пример, в котором на консоль выводятся адреса всех UDP-клиентов:

```c#
var udpClient = new UdpClient("www.contoso.com", 11000);
var udpResultsObservable = Observable.FromAsync(udpClient.ReceiveAsync);
udpResultsObservable.Subscribe(x => Console.WriteLine(x.RemoteEndPoint.Address));
``` 

#### Summary

Мы пробежались по всем способам создания observable-последовательностей, рассмотрели понятия *unfold* и *корекурсии* и их реализации в Rx. Научились создавать временные последовательности, из одного элемента и бесконечные. Так же как получать observable-последовательности из императивного кода. Вот методы, которые были рассмотрены:

* Factory Methods
 * `Observable.Return`
 * `Observable.Empty`
 * `Observable.Never`
 * `Observable.Throw`
 * `Observable.Create`
* Unfold methods
 * `Observable.Range`
 * `Observable.Interval`
 * `Observable.Timer`
 * `Observable.Generate`
* Paradigm Transition
 * `Observable.Start`
 * `Observable.FromEventPattern`
 * `Task.ToObservable`
 * `Task<T>.ToObservable`
 * `IEnumerable<T>.ToObservable`
 * `Observable.FromAsync`

Далее рассмотрим, как стоить запросы к observable-последовательностям.

Stay tuned!

[anamorphism]: https://en.wikipedia.org/wiki/Anamorphism
[corecursion]: https://ru.wikipedia.org/wiki/%D0%9A%D0%BE%D1%80%D0%B5%D0%BA%D1%83%D1%80%D1%81%D0%B8%D1%8F
[latest-post]: {% post_url 2015-05-12-reactive-extensions-overview %}