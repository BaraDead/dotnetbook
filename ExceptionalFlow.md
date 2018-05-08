# Исключения

В нашем разговоре о потоке исполнения команд различными подсистемами пришло время поговорить про исключения или, скорее, исключительные ситуации. И прежде чем продолжить стоит совсем немного остановиться именно на самом определении. Что такое исключительная ситуация? Это такая ситуация, которая делает исполнение дальнейшего или текущего кода абсолютно не корректным. Не таким как задумывалось, проектировалось. Переводит состояние приложения в целом или же его отделой части (например, объекта) в состояние нарушенной целостности. Т.е. что-то экстраординарное, исключительное.

Почему это так важно - определить терминологию? Работа с терминологией очень важна, т.к. она держит нас в рамках. Вот, например: будет ли являться исключительной ситуация когда пользователь ввел в поле ввода чисел букву 'a'? Наверное, нет: мы можем легко проигнорировать ввод. Но если мы поделим любое целое число на ноль это будет исключительной ситуацией: на ноль делить нельзя. Дальнейшее выполнение программы бессмысленно, т.к. расчеты гарантированно не корректны. Исключительные ситуации, возникающие в приложении должны прерывать исполнение текущего, уже более не корректного, кода и искать способы исправить ситуацию. Здесь я попрошу вас обратить внимание на слово "прерывать". Оно очень интересно в первую очередь тем что помимо механизма исключений существует еще один механизм: механизм прерываний. И разница между этими двумя механизмами состоит в том что прерывания останавливают приложение на время, выполняют некоторый код и продолжают выполнение кода программы тогда как исключения работают известным всеми способом: полностью обрубают выполнение кода текущего метода, уводя поток исполнения инструкций процессором в выше-стоящие методы, способные возникшую ошибку обработать.

О чем пойдет речь в этом разделе:

  - Структура блоков try-catch/when-finally, when
  - Виды исключений: что тянется из CLR, а что - из более низкого слоя (Windows SEH)
  - Исключения с особым поведением: ThreadAbortException, OutOfMemoryException и прочие
  - Каким образом идет сборка стека вызовов и производительность выброса исключений
  - Асинхронные исключения
  - Structured Exception Handling
  - Vectored Exception Handling
  - Прерывания

## Состав и развертка блока обработки исключительных ситуаций

Если взглянуть на блок обработки исключительных ситуаций, то мы увидим всем привычную картину:

``` csharp
try {
    // 1
} catch (ArgumentsOutOfRangeException exception)
{
    // 2
} catch (IOException exception)
{
    // 3
} catch
{
    // 4
} finally {
    // 5
}
```

Т.е. существует некий участок кода от которого ожидается некоторое нарушение поведения. Причем не просто некоторое, а вполне конкретные ситуации. Однако, если заглянуть в результирующий код, то мы увидим что по факту эта самая конструкция, которая в C# выглядит как единое целое, в CLI на самом деле разделена на отдельные блоки. Т.е. не существует возможности построить вот такую единую цепочку обработки ошибок, однако есть возможность построить для одного и того же участка отдельные блоки `try-catch` и `try-finally`. И если переводить MSIL обратно в C#, то получим мы следующий код:

``` csharp
try {
    try {
        try {
            try {
                // 1
            } catch (ArgumentsOutOfRangeException exception)
            {
                // 2
            }
        } catch (IOException exception)
        {
            // 3
        }
    } catch
    {
        // 4
    }
} finally {
    // 5
}

// 6

```

Отлично. Однако если мы хотим увидеть картину с точки зрения "как оно все устроено", то полученный код выглядит все же несколько искусственно. Ведь эти блоки - конструкции языка и не более того. Как они разворачиваются в конечном коде? На данном этапе я ограничусь псевдокодом, однако без лишних подробностей он прекрасно покажет во что _примерно_ разворачивается конструкция:

```csharp
GlobalHandlers.Push(BlockType.Finally, FinallyLabel);
GlobalHandlers.Push(BlockType.Catch, typeof(Exception), ExceptionCatchLabel);
GlobalHandlers.Push(BlockType.Catch, typeof(IOException), IOExceptionCatchLabel);
GlobalHandlers.Push(BlockType.Catch, typeof(ArgumentsOutOfRangeException), ArgumentsOutOfRangeExceptionCatchLabel);

// 1

GlobalHandlers.Pop(4);
FinallyLabel:

// 5

goto AfterTryBlockLabel;
ExceptionCatchLabel:
GlobalHandlers.Pop(4);

// 4

goto FinallyLabel;
IOExceptionCatchLabel:
GlobalHandlers.Pop(4);

// 3

goto FinallyLabel;
ArgumentsOutOfRangeExceptionCatchLabel:
GlobalHandlers.Pop(4);

// 2

goto FinallyLabel;
AfterTryBlockLabel:

// 6

return;
```

Также о чем хотелось бы упомянуть во вводной части - это фильтры исключительных ситуаций. Для платформы .NET это новшеством не является, однако является таковым для разработчиков на языке программирования C#: фильтрация исключительных ситуаций появилась у нас только в шестой версии языка. Особенностью исполнения кода по уверениям многих источников является то, что код фильтрации происходит *до* того как произойдет развертка стека. Это можно наблюдать в ситуациях, когда между местом выброса исключения и местом проверки на фильтрацию нет никаких других вызовов кроме обычных:

```csharp
static void Main()
{
    try
    {
        Foo();
    }
    catch (Exception ex) when (Check(ex))
    {
        ;
    }
}

static void Foo()
{
    Boo();
}

static void Boo()
{
    throw new Exception("1");
}

static bool Check(Exception ex)
{
    return ex.Message == "1";
}
```

![](./imgs/ExceptionalFlow/StackWOutUnrolling.png)

Как видно на изображении трассировка стека содержит не только первый вызов `Main` как место отлова исключительной ситуации, но и весь стек до точки выброса исключения плюс повторный вход в `Main` через некоторый неуправляемый код. Можно предположить что этот код и есть код выброса исключений, который просто находится в стадии фильтрации и выбора конечного обработчика если бы не одно _но_. Если посмотреть на результаты работы следующего кода (я добавил проброс вызова через границу между доменами приложения):

```csharp
    class Program
    {
        static void Main()
        {
            try
            {
                ProxyRunner.Go();
            }
            catch (Exception ex) when (Check(ex))
            {
                ;
            }
        }

        static bool Check(Exception ex)
        {
            var domain = AppDomain.CurrentDomain.FriendlyName; // -> TestApp.exe
            return ex.Message == "1";
        }

        public class ProxyRunner : MarshalByRefObject
        {
            private void MethodInsideAppDomain()
            {
                throw new Exception("1");
            }

            public static void Go()
            {
                var dom = AppDomain.CreateDomain("PseudoIsolated", null, new AppDomainSetup
                {
                    ApplicationBase = AppDomain.CurrentDomain.BaseDirectory
                });
                var proxy = (ProxyRunner) dom.CreateInstanceAndUnwrap(typeof(ProxyRunner).Assembly.FullName, typeof(ProxyRunner).FullName);
                proxy.MethodInsideAppDomain();
            }
        }
    }

```

То станет ясно что размотка стека в данном случае происходит еще до того как мы попадаем в фильтр. Взглянем на скриншоты. Первый взят до того как генерируется исключение:

![StackUnroll](./imgs/ExceptionalFlow/StackUnroll.png)

А второй - после:

![StackUnroll2](./imgs/ExceptionalFlow/StackUnroll2.png)

Изучим трассировку вызовов до и после попадания в фильтр исключений. Что же здесь происходит? Здесь мы видим что разработчики платформы сделали некоторую с первого взгляда защиту дочернего домена. Трассировка обрезана по крайний метод в цепочке вызовов, после которого идет переход в другой домен. На самом деле как по мне так это выглядит несколько странно. Чтобы понять, почему так происходит, вспомним основное правило для типов, организующих взаимодействие между доменами. Типы должны наследовать `MarshalByRefObject` плюс - быть сериализуемыми. Однако как бы ни был строг C#, типы исключений могут быть какими угодно. Это могут быть вовсе не Exception-based типы. А что это значит? Это значит что могут быть ситуации, когда исключительная ситуация внутри дочернего домена может привести у уводу в родительский домен объекта, который передан по ссылке, и у которого есть какие-либо опасные методы с точки зрения безопасности. Чтобы такого избежать, исключение сериализуется, проходит через границу доменов приложений и возникает вновь - с новым стеком. Давайте проверим эту стройную теорию:

```csharp
[StructLayout(LayoutKind.Explicit)]
class Cast
{
    [FieldOffset(0)]
    public Exception Exception;

    [FieldOffset(0)]
    public object obj;
}

static void Main()
{
    try
    {
        ProxyRunner.Go();
        Console.ReadKey();
    }
    catch (RuntimeWrappedException ex) when (ex.WrappedException is Program)
    {
        ;
    }
}

static bool Check(Exception ex)
{
    var domain = AppDomain.CurrentDomain.FriendlyName; // -> TestApp.exe
    return ex.Message == "1";
}

public class ProxyRunner : MarshalByRefObject
{
    private void MethodInsideAppDomain()
    {
        var x = new Cast {obj = new Program()};
        throw x.Exception;
    }

    public static void Go()
    {
        var dom = AppDomain.CreateDomain("PseudoIsolated", null, new AppDomainSetup
        {
            ApplicationBase = AppDomain.CurrentDomain.BaseDirectory
        });
        var proxy = (ProxyRunner)dom.CreateInstanceAndUnwrap(typeof(ProxyRunner).Assembly.FullName, typeof(ProxyRunner).FullName);
        proxy.MethodInsideAppDomain();
    }
}
```

В данном примере для того чтобы выбросить исключение любого типа из C# кода был проделан трюк с пиведением типа к несопоставимому. Мы создаем экземпляр типа `Program` - гарантированно не сериализуемого и бросаем исключение с его экземпляром в виде полезной нагрузки. Хорошие новости заключаются в том что вы получите `RuntimeWrappedException`, который внутри себя сохранит экземпляр нашего объекта типа `Program` и в C# перехватить такое исключение мы сможем. Однако есть и плохая новость, которая подтверждает наше предположение: вызов `proxy.MethodInsideAppDomain();` приведет к исключению `SerializationException`:

![](./imgs/ExceptionalFlow/SerializationError.png)

Т.е. проброс между доменами такого исключения не возможен, т.к. его нет возможности сериализовать. А это в свою очередь значит что оборачивание вызовов методов, находящихся в других доменах фильтрами исключений все равно приведет к развертке стека несмотря на то что при `FullTrust` настройках дочернего домена сериализация по сути не нужна.

### AppDomain.UnhandledException

### AppDomain.FirstChanceException

Для того чтобы иметь событие этого типа код-прослойка в кросс-доменном вызове метода должен эти исключения перехватить.

```csharp
static void Main()
{
    ProxyRunner.Go();
}

public class ProxyRunner : MarshalByRefObject
{
    private void MethodInsideAppDomain()
    {
        ThreadPool.QueueUserWorkItem(x =>
        {
            Foo();
        });
    }

    private void Foo()
    {
        throw new Exception("1");
    }

    public static void Go()
    {
        var dom = AppDomain.CreateDomain("PseudoIsolated", null, new AppDomainSetup
        {
            ApplicationBase = AppDomain.CurrentDomain.BaseDirectory
        });
        dom.FirstChanceException += (sender, args) =>
        {
            if(!(args.Exception is ArgumentOutOfRangeException))
                throw new ArgumentOutOfRangeException();
        };
        var proxy = (ProxyRunner)dom.CreateInstanceAndUnwrap(typeof(ProxyRunner).Assembly.FullName, typeof(ProxyRunner).FullName);
        proxy.MethodInsideAppDomain();
    }
}
```

В данном примере мы пробуем перехватить `FirstChanceException`. Это событие указывает нам о том что в домене возникло некоторое исключение. И событие это происходит *до* того как исключение начнет обрабатываться. Прошу заметить что к сожалению мы из этого события не можем никаким образом управлять перехватом исключения. Мы можем только понять и простить, залоггировав ошибку в журнал. Однако, мы можем убедиться в том что скорее всего причина обрезки стека кроется именно тут. Обработчик `FirstChanceException` может пойти тремя путями: отработать хорошо, отработать с исключением, которое не перехватывается, но его выброс приведет к повторному вызову события `FirstChanceException`. Тогда второй вариант - это когда мы вновь бросаем это же исключение и в конечном счете приложение упадет со `StackOverflowException` по причине зацикленности процесса. А третий - это если мы во второй раз это исключение не бросать не будем. Тогда возникает самое страшное: выброс `FatalExecutionEngineError`. Необработанное исключение в `FirstChanceException`. Эту радость не перехватить никак, а потому приложение вылетит раньше чем вы сможете что-то подумать. Вообще данное исключение говорит о том что сам CLR получил некоторую исключительную ситуацию, справиться он с ней не в состоянии и потому пишите письма. Данный случай задокументирован в примере, приведенном выше.

Так вот получается что для того чтобы выбросить `FirstChanceException`, мы внутри метода Throw должны для начала дернуть событие `FirstChanceException`, а уже потом заниматься цепочкой обработчиков исключений. Т.е. код, генерирующий `FirstChanceException` не находится в блоке обработки исключений, он находится вне его а потому на размотку влиять не может.

### [In Progress]

Также стоит остановиться на том, для чего же были введены фильтры исключений. Давайте взглянем на пример и он как по мне будет лучше тысячи слов:

```csharp
try {
    UnmanagedApiWrapper.SomeMethod();
} catch (WrappedException ex) when (ex.ErrorCode == ErrorCodes.DeviceNotFound)
{
    // ...
} catch (WrappedException ex) when (ex.ErrorCode == ErrorCodes.ConnectionLost)
{
    // ...
} catch (WrappedException ex) when (ex.ErrorCode == ErrorCodes.TimeOut)
{
    // ...
} catch (WrappedException ex) when (ex.ErrorCode == ErrorCodes.Disconnected)
{
    // ...
}
```

Согласитесь, это выглядит интереснее чем один блок `catch` и `switch` внутри с `throw;` в `default` блоке. Это выглядит более разграниченным, более правильным с точки зрения разделения ответственности. Ведь исключение с кодом ошибки по своей сути - ошибка дизайна, а фильтрация - это выправка нарушения архитектуры, переводя в кконцепцию раздельных типов исключений.