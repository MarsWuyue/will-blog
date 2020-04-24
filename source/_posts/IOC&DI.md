---
title: IOC 和 DI
tag:
  - IOC
  - DI
categories:
  - 设计模式
---

我们经常都在说控制反转（IOC）和依赖注入（DI），在项目实践中也都在用，但是对他们的概念以及要解决的问题一知半解，希望看完这篇文章后，你可以了解到下面几个问题：

- 什么是IOC？
- 什么是DI？
- 他们有什么区别和联系？
- 为什么要使用他们/他们解决了什么问题？

## IOC

### 什么是IOC

> In software engineering, inversion of control (IoC) is a programming principle. IoC inverts the flow of control as compared to traditional control flow. In IoC, custom-written portions of a computer program receive the flow of control from a generic framework. A software architecture with this design inverts control as compared to traditional procedural programming: in traditional programming, the custom code that expresses the purpose of the program calls into reusable libraries to take care of generic tasks, but with inversion of control, it is the framework that calls into the custom, or task-specific, code.

还是先把wiki上的定义贴出来，这里“traditional control flow”可以理解成传统的过程程序设计（procedural programming），这个定义就变成了“IOC 反转 传统的过程程序设计 的控制流程”，依然比较晦涩难懂，下面我们先看一个小例子：

```c#
 class OrderService
 {
    public void CreateOrder(Order order)
    {
        SaveOrder(order);
        SendSMS(order);
    }

    private static void SendSMS(Order order)
    {
        // 通过短信给用户发送订单创建成功的通知
    }
 }
```

上面我们先调用***SaveOrder***保存订单信息，然后再通过***SendSMS***方法来给用户发送短信通知。如果业务需求改成通过邮件通知用户呢？有点编程经验的人都知道，这时候我们就会抽象出一个通知接口，然后分别实现短信通知和邮件通知了：

```c#
 public class OrderService
 {
     private INotifierService notifierService;
     public OrderService()
     {
         notifierService = new EmailNotifier();
     }
    public void CreateOrder(Order order)
    {
        SaveOrder(order);
        notifierService.send(order);
    }
 }
```

好了，通过上述改动，我们抽象出了一个INotifierService，然后他可以有EmailNotifier和SMSNotifier两种实现了，这回业务再想改，我们只要根据需求实现xxxNotifier即可。到目前为止，我们就实现了IOC了。
总结一下到底什么是IOC，其实IOC中所谓的“反转”，我认为可以理解成交出，那么，控制反转就是交出控制权，本来由OrderService负责的发送通知的功能交给NotifierService去做。理解了神秘的IOC之后，我们可以思考一下，其实IOC在我们编程过程中早就接触过了，做过WebForms的应该对```OnPageLoad()```方法不陌生吧？这其实就是控制反转，是框架把页面加载的控制权交给了应用程序（我们的业务代码），还有*代理（Delegates）、虚方法（Virtual Functions）和EventHandler*，这些其实都是控制反转。

### 为什么要用IOC

解释完什么是IOC了，我们来看一下IOC的好处，其实从上面两个例子很容易看出，*IOC使代码更加灵活和松耦合，达到了易维护和易扩展的目的*，还是上面的例子，通知只是创建订单过程中的一个流程，所以OrderService应该专注于实现订单业务以及控制业务流程，不需要去关心如何实现通知相关的功能。

## DI

### 什么是DI

老规矩，上wiki：

> In software engineering, dependency injection is a technique whereby one object supplies the dependencies of another object. A "dependency" is an object that can be used, for example as a service. Instead of a client specifying which service it will use, something tells the client what service to use. The "injection" refers to the passing of a dependency (a service) into the object (a client) that would use it. The service is made part of the client's state.[1] Passing the service to the client, rather than allowing a client to build or find the service, is the fundamental requirement of the pattern.

了解了IOC，那么我们还是回到刚才的例子，大家认为我们上面是好的设计吗？当然不是！虽然我们已经抽象了个EmailNotifier出来，但是对于OrderService来说，他还是得知道我应该用哪个实现类发通知，也就是说，如果业务又要修改为回访电话通知了，那我们不止要再实现个通知类，还需要修改OrderService中的代码，这显然不是什么好的设计，如果能改成下面这样呢：

```c#
 public class OrderService
 {
     private INotifierService notifierService;
     public OrderService(INotifierService notifierService)
     {
         this.notifierService = notifierService;
     }
    public void CreateOrder(Order order)
    {
        SaveOrder(order);
        notifierService.send(order);
    }
 }
```

哦了，这么看起来OrderService满意了，但是构造函数中的```INotifierService notifierService```哪儿来的？好了，DI开始它的表演了。
构造函数里面的参数是框架（比如java的Spring自带IOC容器，再比如LightInject和Autofac等）实现的，他们的实现方法就是提供一个容器，在程序启动的时候将你需要的东西都放到容器中，然后如果谁要用，容器就会自动把它给你。

好了，总结一下，什么是DI，DI的意思就是你的依赖是被自动注进来的，你只需要提个需求，说你想依赖啥啥啥就行了，提需求的方式也是多种多样的，比如你可以把你的需求写在构造函数上，也可以通过属性标签提需求，具体的使用方法可以查阅你使用的框架的官方文档。

### 为什么用DI

当然是DI好处多啊，上面已经展示过了，DI可以使你的代码进一步松耦合：比如上面的代码，自己体会

下面我说一下一般的DI框架提供的其他附赠服务：

- 代码简洁了：相信大家都曾经被new搞得头昏脑胀吧？我想要使用OrderService类，我需要new一个通知类，如果这个通知类的构造函数包括十几个参数，再如果你的OrderService依赖七八个其他类，你还要去了解每个依赖他们又需要依赖什么，这个递归做完估计头就已经昏了
- DI可以帮你实现单例：大家几乎都会在自己的Service中使用单例模式，使自己的service无状态，用DI，帮你直接搞定，少些了好几行代码，而且他还可以帮你控制单例的生命周期
- 用了DI，妈妈再也不用担心你写单元测试了，比如Moq，和LigheInject很好的结合，写单元测试的时候只需要专注当前要测试的类即可，其他的一律Mock掉

## IOC和DI

IOC和DI经常一起出现，甚至还有一些文章说IOC和DI是一个东西，还经常用IOC/DI这种形式来表示，我对这种观点并不是很赞同。通过上面的分析其实很明确，IOC和DI解决的问题是不一样的。但是他们通常组合起来解决问题，网上有下面的说法，我比较赞同：

- IOC是设计原则，DI是设计模式，或者说是一种实现模式

我认为不管他们叫什么，不管你是不是觉得“控制反转”中的“反转”表述的不正确（有人提议应该叫DOC，Delegate of control，我感觉也不错），这都不重要，重要的是理解他们到底是什么以及他们到底是用来解决什么问题的，并且能够在项目中合理运用。
