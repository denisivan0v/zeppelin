---
layout: post
title: "Reactive Extensions: sequences and subscriptions"
date: 2015-07-30 14:30:00
isStaticPost: false
categories: [reactive extensions rx sequences subscriptions disposable]
---

#### Подписки и управление временем жизни

Работая с observable-последовательностями, мы не можем знать когда она начинается и когда заканчивается. Этим знанием обладает источник, но не подписчик. Тем не менее, необходим детерменированный механизм управления подпиской - мы самостоятельно должны контролировать момент подписки и начала получения данных/событий/элементов последовательности, также как и момент окончания работы с observable-последовательностью. Давайте еще раз взглянем на интерфейс `IObservable`:

```c#
public interface IObservable<out T>
{
    IDisposable Subscribe(IObserver<T> observer);
}

```

Важно обратить внимание на два аспекта. Вызывая `Subscribe`, мы явно создаем подписку и начинаем работать с элементами последовательности. Однако возвращаемое значение метода - не какой-либо объект, определяющий абстракцию подписки (хотя внутри Rx такая абстракция есть), но объект, реализующий `IDisposable`. Именно этот объект дает нам возможность явно закончить работать с последовательностью, вызывав метод `Dispose`.

```c#
var subject = new Subject<int>();
var subscription1 = subject.Subscribe(x => { Console.WriteLine($"subscription1, value: {x}"); });
var subscription2 = subject.Subscribe(x => { Console.WriteLine($"subscription2, value: {x}"); });

// будет обработано и subscription1, и subscription2
subject.OnNext(1);

subscription2.Dispose();

// будет обработано только subscription1
subject.OnNext(2);

subscription1.Dispose();
```

Как и обычно при использование `IDisposable`-объектов можно вызывать `Dispose` сколько угодно раз.

В Rx есть несколько служебных типов, реализующих `IDisposable`, которые удобно использовать в различных ситуациях (позже мы рассмотрим некоторых из таких ситуаций). Их можно использовать, например, с помощью статического [класса][disposable-cs] `Disposable`, содержащего свойство `Empty` и фабричный метод `Create`:

```c#
public static class Disposable
{
    public static IDisposable Empty
    {
        get { return DefaultDisposable.Instance; }
    }

    public static IDisposable Create(Action dispose)
    {
        if (dispose == null)
            throw new ArgumentNullException("dispose");

        return new AnonymousDisposable(dispose);
    }
}
```

Стоит отдельно отметить, что подписчик, вызывая `Subscribe` и получая экземпляр `IDisposable` (подписку), обязан сам вызвать `Dispose`. Если этого не сделать, либо вообще проигнорировать возвращемое методом `Subscribe` значение, это может привести к наличию в памяти нежелательных объектов и утечке памяти (а также снижению производительности), т.к. ссылки на подписки сущесвуют в райнтайме Rx и они не будут собраны GC автоматически. Также нужно помнить про то, что будет с подписками после завершения последовательности или в случае возникновения исключения. Об этом чуть ниже.

#### Обработка исключений

Как уже отмечалось ранее, мы можем вызывать `Subscribe`, передавая не только `IObserver<T>`, но и делегаты. Вот [набор][subsribe-overloads] extension-методов для `Subscribe`:

```c#
IDisposable Subscribe<TSource>(this IObservable<TSource> source);
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
                               Action<TSource> onNext);
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
                               Action<TSource> onNext, 
                               Action<Exception> onError);
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
                               Action<TSource> onNext, 
                               Action onCompleted);
IDisposable Subscribe<TSource>(this IObservable<TSource> source, 
                               Action<TSource> onNext, 
                               Action<Exception> onError,
                               Action onCompleted);

```

Эти методы позволяют нам указывать делегаты, которые будут вызваны для каждого элемента последовательности (`onNext`), для обработки ошибок на стороне источника (`onError`) и для обработки окончания последовательности (`onCompleted`).

По-умолчанию, реализация метода `OnError` в `Subject<T>` пробрасывает исключение, если подписка была создана без определения делегата `onError`. Это означает, что при использовании try/catch/finally в catch нужно оборачивать именно вызов метода `OnError` источника последовательности. То есть подписчик ничего не узнает об исключении. Поэтому, если требуется обрабатывать исключения, нужно использовать перегрузку `Subscribe` с определением делегата `onError`.

Не верно:

```c#
var subject = new Subject<int>();
try
{
    subject.Subscribe();
}
catch (Exception)
{
    // сюда не придет выполнение
}

// исключение будет выбрашено в этой строке
subject.OnError(new Exception("Exception"));
```

Верно:

```c#
var subject = new Subject<int>();
subject.Subscribe(
    x => { },
    ex =>
    {
        // исключение будет обработано здесь
        Console.WriteLine($"Exception occured: {ex}");
    });
subject.OnError(new Exception("Exception"));
```

В случае возникновения исключения, а также в случае окончания последовательности, подпичики не "отключаются" от источника автоматически (это происходит только в случае использования [метода][subscribe-safe] `SubscribeSafe`). Поэтому важно не забывать отписываться явно, вызывая `Dispose`. Это также важно и в случае бесконечных последовательностей, когда `IObserver.OnCompleted` никогда не будет вызван.

#### Summary

Rx позволяет явно управлять временем жизни подписок, поэтому важно не забывать "диспозить" создаваемые при вызове `IObservable.Subscribe` подписки. Это нужно делать даже в случае окончания последовательности или возникновения исключения.

Исключения же обрабатываются не с помощью привычного try/catch/finally-паттерна, с помощью определения делегата `onError` при создании подписки. Также есть и другие способы, но об этом в следующих постах.

Stay tuned!

[subsribe-overloads]: https://msdn.microsoft.com/en-us/library/system.observableextensions.subscribe(v=vs.103).aspx
[disposable-cs]: https://github.com/Reactive-Extensions/Rx.NET/blob/a13e3ff05bdded5cef2bf40bface22f8fa4ae316/Rx.NET/Source/System.Reactive.Core/Reactive/Disposables/Disposable.cs
[subscribe-safe]: https://github.com/Reactive-Extensions/Rx.NET/blob/a13e3ff05bdded5cef2bf40bface22f8fa4ae316/Rx.NET/Source/System.Reactive.Core/Observable.Extensions.cs