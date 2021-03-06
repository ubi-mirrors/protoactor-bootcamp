# Урок 6. Реализация шаблона маршрутиэатора с применением акторов.

Реализация шаблона маршрутизатора не всегда требует применения встроенных маршрутизаторов платформы Proto.Actor. Когда выбор маршрута зависит от содержимого сообщения или некоторого состояния, проще реализовать обычный актор, потому что такой приём позволяет использовать все преимущества акторов. Кроме того, если вы решите создать свой маршрутизатор по аналогии с маршрутизаторами Proto.Actor, вам придётся решать проблемы, связанные с многопоточный обработкой.

В этом уроке мы рассмотрим несколько реализаций шаблона маршрутизатора с использованием обычных акторов. Начнём с версии, выполняющей маршрутизацию по содержимому сообщений. В следующем разделе мы используем методы become/unbecome для реализации маршрутизации в зависимости от состояния маршрутизатора. После этого мы обсудим, почему для реализации шаблона маршрутизатора необязательно использовать отдельный актор, а вместо этого маршрутизацию можно встроить в актор, обрабатывающий сообщения.

#### Маршрутизация по содержимому.

Чаще всего маршрутизация сообщений производится по их содержимому. В начале этого модуля мы с вами рассматривали пример такого маршрутизатора. Когда скорость автомобиля ниже предельно допустимой, значит, правила дорожного движения не были нарушены, и такое сообщение требуется передать на этап удаления. Когда скорость выше предельно допустимой, значит, нарушение имеет место, и обработка должна быть продолжена.

![](images/5_6_1.png)

На рис. показано, как по содержимому сообщения выбирается тот или иной поток обработки. В данном случае маршрутизация осуществляется на основе значения скорости, но с таким же успехом решение может приниматься на основе любых других проверок содержимого сообщения. Мы не показываем здесь код реализации этой версии, потому что он до статочно прост, чтобы вы, с текущим уровнем знания платформы Proto.Actor, смогли написать его самостоятельно. В следующем разделе мы рассмотрим маршрутизацию на основе состояния.

#### Маршрутизация на основе состояния.

Этот подход подразумевает изменение порядка маршрутизации при изменении состояния маршрутизатора. Простейшим примерам может служить переключение маршрутизатора между двумя состояниями: **on** и **off**. Когда маршрутизатор находится в состоянии **on**, он пересылает сообщения как обычно, а состояние **off** передаёт все сообщения задаче удаления. Для реализации этого примера нельзя использовать маршрутизатор Proto.Actor, потому что нам требуется, чтобы маршрутизатор обладал состоянием, а состояние маршрутизаторов Proto.Actor по умолчанию не является потокобезопасным. Поэтому используем самый обычный актор. Состояние можно реализовать как атрибут класса, но поскольку есть возможность изменять поведение актора в течение его жизненного цикла с помощью методов become/unbecome, используем их в роли механизма представления состояния.

В нашем примере имеются два состояния, **on** и **off**. Когда актор находится в состоянии **on**, сообщения должны маршрутизироваться как обычно, а в состоянии **off** сообщения должны просто уничтожаться. Для этого создадим два метода обработки сообщений. Когда потребуется переключить состояние, мы просто заменим функцию приёма с помощью become. Для изменения состояния в данном примере используются два сообщения: `RouteStateOn()` и R`outeStateOff()`.

```c#
public class SwitchRouter : IActor
{
    private readonly PID _normalFlow;
    private readonly PID _cleanUp;
    private readonly Behavior _behavior;

    public Task ReceiveAsync(IContext context) => _behavior.ReceiveAsync(context);

    public SwitchRouter(PID normalFlow, PID cleanUp)
    {
        _normalFlow = normalFlow;
        _cleanUp = cleanUp;
        _behavior = new Behavior(Off);
    }

    private Task On(IContext context)
    {
        switch (context.Message)
        {
            case Started _:
                break;
            case RouteStateOn msg:
                Console.WriteLine("Received on while already in on state");
                break;
            case RouteStateOff msg:
                _behavior.Become(On);
                break;
            default:
                context.Forward(_normalFlow);
                break;
        }
        return Actor.Done;
    }

    private Task Off(IContext context)
    {
        switch (context.Message)
        {
            case Started _:
                break;
            case RouteStateOn msg:
                _behavior.Become(On);
                break;
            case RouteStateOff msg:
                Console.WriteLine("Received off while already in off state");
                break;
            default:
                context.Forward(_cleanUp);
                break;
        }
        return Actor.Done;
    }
}
```



В нашем примере имеются два состояния, **on** и **off**. Когда актор находится в состоянии **on**, сообщения должны маршрутизироваться, как обычно, а в состоянии **off** сообщения должны просто уничтожаться. Для этого создадим два метода обработки сообщений. Когда потребуется переключить состояние, мы просто заменим функцию приёма с помощью метода become контекста актора. Для изменения состояния в данном примере используются два сообщения: `RouteStateOn()` и `RouteStateOff()`.

#### Реализации маршрутизаторов.

До сих пор мы рассматривали разные примеры маршрутизаторов и способы их реализации. Но все они представляли чистые реализации шаблона «Маршрутизатор»; сами маршрутизаторы никак не обрабатывали сообщения, а просто пересыпали их соответствующим адресатам. 

Это типичное решение на этапе предварительного проектирования, но иногда целесообразнее включить маршрутизацию прямо в актор, обрабатывающий сообщения, как показано на рисунке. Такой приём имеет смысл, когда результаты обработки могут влиять на выбор следующего этапа.

![](images/5_6_2.png)

В нашем примере с дорожной камерой актор GetSpeed определял скорость. Когда он терпел неудачу или скорость оказывалась ниже установленного порога, сообщение передавалось на удаления; иначе сообщение передавалось для дальнейшей обработки, в нашем случае - актору GetTime. Чтобы воплотить эту схему, нам потребуется реализовать два шаблона:

- задача обработки.
- маршрутизатор.

При этом оба компонента, GetSpeed и SpeedRouter, можно реализовать в одном акторе. Сначала актор выполняет задачу обработки и по ее результату определяет, куда дальше отправить сообщение - задаче GetTime или на удаление. Решение о реализации этих компонентов в одном акторе или в двух зависит от требований к возможности повторного использования. Если необходимо, чтобы GetSpeed была отдельной функциеи?, мы не сможем совместить оба шага в одном акторе. Но если обрабатывающий актор также должен принимать решение о дальнейшей обработке сообщения, проще будет совместить эти два компонента. Другим фактором могло бы быть разделение обычного потока обработки и потока обработки ошибок для компонента GetSpeed.

