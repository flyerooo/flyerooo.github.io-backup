+++
date = "2018-12-02T00:00:00+08:00"
tag = ""
title = "Python代码整洁之道--使用装饰器改进代码"
undefined = ""

+++
本文为英文书籍 [_Clean Code in Python_](https://link.juejin.im?target=https%3A%2F%2Fwww.packtpub.com%2Fapplication-development%2Fclean-code-python) Chapter 5 Using Decorators to Improve Our Code 学习笔记，建议直接看原书

* 了解Python中装饰器的工作原理
* 学习如何实现应用于函数和类的装饰器
* 有效地实现装饰器，避免常见的执行错误
* 分析如何用装饰器避免代码重复（DRY）
* 研究装饰器如何为关注点分离做出贡献
* 优秀装饰器实例分析
* 回顾常见情况、习惯用法或模式，了解何时装饰器是正确的选择

虽然一般见到装饰器装饰的是方法和函数，但实际允许装饰任何类型的对象，因此我们将探索应用于函数、方法、生成器和类的装饰器。

还要注意，不要将装饰器与装饰器设计模式(Decorator Pattern)混为一谈。

#### 函数装饰

函数可能是可以被装饰的Python对象中最简单的表示形式。我们可以在函数上使用装饰器来达成各种逻辑——可以验证参数、检查前提条件、完全改变行为、修改签名、缓存结果（创建原始函数的存储版本）等等。

作为示例，我们将创建实现重试机制的基本装饰器，控制特定的域级异常(domain-level exception)并重试一定次数：

    # decorator_function_1.py
    import logging
    from functools import wraps
    
    logger = logging.getLogger(__name__)
    
    
    class ControlledException(Exception):
        """A generic exception on the program's domain."""
        pass
    
    
    def retry(operation):
        @wraps(operation)
        def wrapped(*args, **kwargs):
            last_raised = None
            RETRIES_LIMIT = 3
            for _ in range(RETRIES_LIMIT):
                try:
                    return operation(*args, **kwargs)
                except ControlledException as e:
                    logger.info("retrying %s", operation.__qualname__)
                    last_raised = e
            raise last_raised
    
        return wrapped
    

可以暂时忽略@wraps，之后再介绍  
retry装饰器使用例子：

    @retry
    def run_operation(task):
       """Run a particular task, simulating some failures on its execution."""
       return task.run()
    复制代码

因为装饰器只是提供的一种语法糖，实际上等于`run_operation = retry(run_operation)`  
比较常用的**超时重试**，便可以这样实现。

#### [定义一个带参数的装饰器](https://link.juejin.im?target=https%3A%2F%2Fpython3-cookbook.readthedocs.io%2Fzh_CN%2Flatest%2Fc09%2Fp04_define_decorator_that_takes_arguments.html%23id3)

我们用一个例子详细阐述下接受参数的处理过程。 假设你想写一个装饰器，给函数添加日志功能，同时允许用户指定日志的级别和其他的选项。 下面是这个装饰器的定义和使用示例：

    from functools import wraps
    import logging
    
    def logged(level, name=None, message=None):
        """
        Add logging to a function. level is the logging
        level, name is the logger name, and message is the
        log message. If name and message aren't specified,
        they default to the function's module and name.
        """
        def decorate(func):
            logname = name if name else func.__module__
            log = logging.getLogger(logname)
            logmsg = message if message else func.__name__
    
            @wraps(func)
            def wrapper(*args, **kwargs):
                log.log(level, logmsg)
                return func(*args, **kwargs)
            return wrapper
        return decorate
    
    # Example use
    @logged(logging.DEBUG)
    def add(x, y):
        return x + y
    
    @logged(logging.CRITICAL, 'example')
    def spam():
        print('Spam!')
    
    复制代码

初看起来，这种实现看上去很复杂，但是核心思想很简单。 最外层的函数 `logged()` 接受参数并将它们作用在内部的装饰器函数上面。 内层的函数 `decorate()` 接受一个函数作为参数，然后在函数上面放置一个包装器。 这里的关键点是包装器是可以使用传递给 `logged()` 的参数的。

定义一个接受参数的包装器看上去比较复杂主要是因为底层的调用序列。特别的，如果你有下面这个代码：

    @decorator(x, y, z)
    def func(a, b):
        pass
    复制代码

装饰器处理过程跟下面的调用是等效的;

    def func(a, b):
        pass
    func = decorator(x, y, z)(func)
    decorator(x, y, z) 的返回结果必须是一个可调用对象，它接受一个函数作为参数并包装它
    复制代码

#### 类装饰

有些人认为，装饰类是比较复杂的事情，而且这样的方案可能危及可读性。因为我们在类中声明一些属性和方法，但是装饰器可能会改变它们的行为，呈现出完全不同的类。

在这种技术被严重滥用的情况下，这种评价是正确的。客观地说，这与装饰函数没有什么不同；毕竟，类只是Python生态系统中的另一种类型的对象，就像函数一样。我们将在标题为“装饰器和关注点分离”的章节中一起回顾这个问题的利弊，但是现在，我们将探讨类的装饰器的好处：

* 代码重用和DRY。一个恰当的例子是，类装饰器强制多个类符合某个特定的接口或标准（通过在装饰器中仅检查一次，而能应用于多个类）
* 可以创建更小或更简单的类，而通过装饰器增强这些类
* 类的转换逻辑将更容易维护，而不是使用更复杂(通常是理所当然不被鼓励的)的方法，比如元类

回顾监视平台的事件系统，我们现在需要转换每个事件的数据并将其发送到外部系统。 但是，在选择如何发送数据时，每种类型的事件可能都有自己的特殊性。

特别是，登录的事件可能包含敏感信息，如登录信息需要隐藏， 时间戳等其他字段也可能需要特定的格式显示。

    class LoginEventSerializer:
        def __init__(self, event):
            self.event = event
    
        def serialize(self) -> dict:
            return {
                "username": self.event.username,
                "password": "**redacted**",
                "ip": self.event.ip,
                "timestamp": self.event.timestamp.strftime("%Y-%m-%d% H: % M"),}
    
    
    class LoginEvent:
        SERIALIZER = LoginEventSerializer
    
        def __init__(self, username, password, ip, timestamp):
            self.username = username
            self.password = password
            self.ip = ip
            self.timestamp = timestamp
    
        def serialize(self) -> dict:
            return self.SERIALIZER(self).serialize()
    复制代码

在这里，我们声明一个类，该类将直接映射到登录事件，包含其逻辑——隐藏密码字段，并根据需要格式化时间戳。

虽然这种方法可行，而且看起来是个不错的选择，但是随着时间的推移，想要扩展我们的系统，就会发现一些问题:

* 类太多：随着事件数量的增加，序列化类的数量将以相同的数量级增长，因为它们是一一映射的。
* 解决方案不够灵活：如果需要重用组件的一部分（例如，我们需要在另一种事件中隐藏密码），则必须将其提取到一个函数中，还要从多个类中重复调用它，这意味着我们没有做到代码重用。
* Boilerplate：serialize()方法必须出现在所有事件类中，调用相同的代码。虽然我们可以将其提取到另一个类中（创建mixin），但它似乎不是继承利用的好方式（ Although we can extract this into another class (creating a mixin), it does not seem like a good use of inheritance.）。

另一种解决方案是，给定一组过滤器（转换函数）和一个事件实例,能够动态构造对象，该对象能够通过滤器对其字段序列化。然后，我们只需要定义转换每种类型的字段的函数，并且通过组合这些函数中的许多函数来创建序列化程序。

一旦有了这个对象，我们就可以装饰类，以便添加serialize()方法，该方法将只调用这些Serialization对象本身：

    def hide_field(field) -> str:
        return "**redacted**"
    
    
    def format_time(field_timestamp: datetime) -> str:
        return field_timestamp.strftime("%Y-%m-%d %H:%M")
    
    
    def show_original(event_field):
        return event_field
    
    
    class EventSerializer:
        def __init__(self, serialization_fields: dict) -> None:
            self.serialization_fields = serialization_fields
    
        def serialize(self, event) -> dict:
            return {
                field: transformation(getattr(event, field))
                for field, transformation in self.serialization_fields.items()
            }
    
    
    class Serialization:
        def __init__(self, **transformations):
            self.serializer = EventSerializer(transformations)
    
        def __call__(self, event_class):
            def serialize_method(event_instance):
                return self.serializer.serialize(event_instance)
    
            event_class.serialize = serialize_method
            return event_class
    
    
    @Serialization(
        username=show_original,
        password=hide_field,
        ip=show_original,
        timestamp=format_time,
    )
    class LoginEvent:
        def __init__(self, username, password, ip, timestamp):
            self.username = username
            self.password = password
            self.ip = ip
            self.timestamp = timestamp

待续。。。