---
layout: post
title: "Digging deeper into Rx sequences"
date: 2015-07-30 22:40:00
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
var subject = new Subject<int>();
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

Метод `Create` - рекомендуемый способ создания observable-последовательностей по двум причинам:
 
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

Параметры метода:

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

Сигнальная функция, которая публикует только одно значение после прошествия заданного промежутка времени, задаваемого с помощью параметра типа `TimeSpan`. Последовательность завершается сразу, однако если воспользоваться перегрузкой с типом параметра `DateTimeOffset`, то последовательность завершится в указанное время. 

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

**Использование делегатов**

#### Summary

Rx позволяет явно управлять временем жизни подписок, поэтому важно не забывать "диспозить" создаваемые при вызове `IObservable.Subscribe` подписки. Это нужно делать даже в случае окончания последовательности или возникновения исключения.

Исключения же обрабатываются не с помощью привычного try/catch/finally-паттерна, с помощью определения делегата `onError` при создании подписки. Также есть и другие способы, но об этом в следующих постах.

Stay tuned!

[anamorphism]: https://en.wikipedia.org/wiki/Anamorphism
[corecursion]: https://ru.wikipedia.org/wiki/%D0%9A%D0%BE%D1%80%D0%B5%D0%BA%D1%83%D1%80%D1%81%D0%B8%D1%8F
