# 设计模式

### 适配器模式、装饰器模式、代理模式区别

**适配器模式**：老接口不能修改，但是新功能需和老接口不一样，这样就需要一个适配器类来解决。**将原接口转化为客户希望的另一个接口，就是适配器模式！**

例如短信云厂商提供了一些接口，我这里传入了一些参数，但是这些参数和别人接口不一样，就需要在适配器类中添加这些参数来适配别人的接口。



**代理模式**：不用改接口，只添加新的功能。例如系统需要升级，原先的接口没有实现安全机制，现在需要实现安全机制，只需要新增代理器即可。又例如 Spring AOP 代理，就是在原本方法的基础上添加额外的操作，例如打印日志、权限校验等。

但是代理模式只能代理自己的类，如果是别人的一个 JAR 包，就不能代理了，除非进行反编译再修改，当然不太可能。所以就需要使用适配器模式。



**装饰器模式**：有一个装饰器类和被装饰类。**装饰模式的对象一定是从外部传入的**，再从使用上来看，**代理模式注重的是隔离限制**，让外部不能访问你实际的调用对象，比如权限控制，**装饰模式注重的是功能的拓展**，在同一个方法下实现更多的功能。



