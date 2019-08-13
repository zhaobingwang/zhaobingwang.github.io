---
title: 对象间映射框架AutoMapper了解一下
categories: 开发辅助
comments: true
date: 2019/04/11
updated: 2019/04/11
tags:
    - AutoMapper
    - 对象映射
    - 开发辅助
---

# 我们为什么要在对象之间做映射
> 对象映射可能发生在应用程序的许多地方，但主要发生在层之间的边界，比如UI层和服务层数据交互时，我们的系统与其他系统之间进行数据交互时等。此时使用对象之间的映射来隔离模型，降低层与层之间的耦合度。一个典型的场景就是`实体对象`和`数据传输对象`之间的映射。

# AutoMapper了解一下
对象之间的映射是一件非常无聊的事情，基于`MIT`协议的开源框架[AutoMapper](https://github.com/AutoMapper/AutoMapper)可以帮助我们做这些无聊的工作。

# 开始使用，先来一个简单的示例
安装`AutoMapper`的`Nuget`包
> PM> Install-Package AutoMapper

以下包含两种初始化配置及映射的方式
```csharp
// 第一种方式
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>();
    cfg.CreateMap<Source, Destination>();
});
var mapper = config.CreateMapper();
var dest = mapper.Map<Source, Destination>(new Source
{
    Id = 1,
    Name = "test"
});

// 第二种方式
Mapper.Initialize(cfg =>
{
    cfg.CreateMap<Source, Destination>();
});
var dest2 = Mapper.Map<Destination>(new Source
{
    Id = 1,
    Name = "test"
});
```

# Flattening（扁平化）
将复杂模型映射为简单模型
```csharp
public class Order
{
    private readonly List<OrderLineItem> _orderLineItems = new List<OrderLineItem>();
    public Customer Customer { get; set; }
    public OrderLineItem[] GetOrderLineItems()
    {
        return _orderLineItems.ToArray();
    }
    public void AddOrderLienItem(Product product, int quantity)
    {
        _orderLineItems.Add(new OrderLineItem(product, quantity));
    }
    public decimal GetTotal()
    {
        return _orderLineItems.Sum(li => li.GetTotal());
    }
}
public class Customer
{
    public string Name { get; set; }
}
public class Product
{
    public decimal Price { get; set; }
    public string Name { get; set; }
}
public class OrderLineItem
{
    public OrderLineItem(Product product, int quantity)
    {
        Product = product;
        Quantity = quantity;
    }
    public Product Product { get; private set; }
    public int Quantity { get; private set; }
    public decimal GetTotal()
    {
        return Quantity * Product.Price;
    }
}
```
现在我们希望将此对象映射为一个简单的`DTO`，只包含特定场景下的某些数据：
```csharp
public class OrderDto
{
    public string CustomerName { get; set; }
    public decimal Total { get; set; }
}
```

在`AutoMapper`中配置源/目标类型时，配置程序会尝试将源类型上的属性和方法与目标类型上的属性进行匹配。 如果对于目标类型的任何属性，源类型上不存在属性，方法或前缀为“Get”的方法，则`AutoMapper`会将目标成员名称拆分为单个单词（通过`PascalCase`约定）。
```csharp
var customer = new Customer
{
    Name = "test"
};
var order = new Order
{
    Customer = customer
};
var product = new Product
{
    Name = "Surface laptop",
    Price = 12999.99m
};
order.AddOrderLienItem(product, 3);

var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Order, OrderDto>();
});
var mapper = config.CreateMapper();
var dto = mapper.Map<Order, OrderDto>(order);
```
我们使用`CreateMap`方法在`AutoMapper`中配置了类型映射。 `AutoMapper`只能映射它知道的类型对，因此我们使用`CreateMap`显式注册了源/目标类型对。 要执行映射，我们使用Map方法。 在`OrderDto`类型上，`Total`属性与`Order`上的`GetTotal()`方法匹配。 `CustomerName`属性与`Order`上的`Customer.Name`属性匹配。 只要我们恰当地命名目标属性，我们就不需要配置单独的属性匹配。

# Projection(投影)
投影将源转换为目标，而不是展平对象模型。 如果没有额外的配置，`AutoMapper`需要一个展平的目标来匹配源类型的命名结构。 如果要将源值投影到与源结构不完全匹配的目标中，则必须指定自定义成员映射定义。
```csharp
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<CalendarEvent, CalendarEventForm>()
        .ForMember(dest => dest.EventDate, opt => opt.MapFrom(src => src.Date.Date))
        .ForMember(dest => dest.EventHour, opt => opt.MapFrom(src => src.Date.Hour))
        .ForMember(dest => dest.EventMinute, opt => opt.MapFrom(src => src.Date.Minute));
});
var mapper = configProjection.CreateMapper();
var calendarEventForm = mapperProjection.Map<CalendarEvent, CalendarEventForm>(new CalendarEvent
{
    Date = new DateTime(2019, 3, 23, 10, 10, 10),
    Title = "Go party."
});
```
以上实例的对象如下：
```csharp
public class CalendarEvent
{
    public DateTime Date { get; set; }
    public string Title { get; set; }
}
public class CalendarEventForm
{
    public DateTime EventDate { get; set; }
    public int EventHour { get; set; }
    public int EventMinute { get; set; }
    public string Title { get; set; }
}
```
# Configuration Validation(配置验证)
手工绘制的映射代码虽然繁琐，但具有可测试的优点。`AutoMapper`背后的灵感之一是不仅消除了自定义映射代码，而且消除了手动测试的需要。由于从源到目标的映射是基于约定的，因此您仍需要测试配置。
`AutoMapper`以`AssertConfigurationIsValid`方法的形式提供配置测试。假设我们的源和目标类型略有错误配置：
```csharp
public class Source
{
    public string SomeValue { get; set; }
}
public class Destination
{
    public string SomeValuefff { get; set; }
}
```
在`Destination`类型中，我们可能会使目标属性出错。 要测试我们的配置，我们只需创建一个单元测试来设置配置并执行`AssertConfigurationIsValid`方法(当源/目标 的属性不能匹配时，会抛出包含具体属性名的异常)：
```csharp
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>();
});
var mapper = config.CreateMapper();
var dest = mapper.Map<Destination>(new SourceConfigurationValidation
{
    SomeValue = "hi."
});
config.AssertConfigurationIsValid();
```
我们有`三种`方式可以处理配置错误：
- [Custom Value Resolvers](#Custom-Value-Resolvers)
- [Projection(投影)](#Projection(投影))
- Use the Ignore() option
```csharp
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>()
    .ForMember(dest => dest.SomeValuefff, opt => opt.Ignore());
});
```
默认情况下，`AutoMapper`使用目标类型来验证成员，它假定目标成员的所有属性都需要映射。 要修改此行为，请使用`CreateMap`重载指定要验证的成员列表：
```csharp
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>(MemberList.Source);
});
```
如果您需要完全的跳过验证，可使用`MemberList.None`。

# Lists and Arrays
`AutoMapper`只需要配置元素类型，而不是数组或列表类型。例如，我们可能有一个简单的源和目标类型：
```csharp
public class Source
{
    public int Value { get; set; }
}
public class Destination
{
    public int Value { get; set; }
}
```
支持C#中所有的集合类型：
```csharp
Mapper.Initialize(cfg => cfg.CreateMap<Source, Destination>());
var sources = new[]
{
    new Source { Value = 5 },
    new Source { Value = 6 },
    new Source { Value = 7 }
};
IEnumerable<Destination> ienumerableDest = Mapper.Map<Source[], IEnumerable<Destination>>(sources);
ICollection<Destination> icollectionDest = Mapper.Map<Source[], ICollection<Destination>>(sources);
IList<Destination> ilistDest = Mapper.Map<Source[], IList<Destination>>(sources);
List<Destination> listDest = Mapper.Map<Source[], List<Destination>>(sources);
Destination[] arrayDest = Mapper.Map<Source[], Destination[]>(sources);
```
对于非泛型可枚举类型，仅支持未映射的可分配类型，因为`AutoMapper`将无法“猜测”您尝试映射的类型。 如上例所示，没有必要显式配置集合类型，只需要配置其成员类型。 映射到现有集合时，首先清除目标集合。

# Nested Mappings(嵌套映射)
```csharp
public class OuterSource
{
    public int Value { get; set; }
    public InnerSource Inner { get; set; }
}
public class InnerSource
{
    public int OtherValue { get; set; }
}
public class OuterDest
{
    public int Value { get; set; }
    public InnerDest Inner { get; set; }
}
public class InnerDest
{
    public int OtherValue { get; set; }
}
```
配置及映射
```csharp
var configNested = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<OuterSource, OuterDest>();
    cfg.CreateMap<InnerSource, InnerDest>();
});
config.AssertConfigurationIsValid();

var source = new OuterSource
{
    Value = 5,
    Inner = new InnerSource { OtherValue = 15 }
};
var mapperNested = config.CreateMapper();
var destNested = mapper.Map<OuterSource, OuterDest>(source);
```

# Custom Type Converters
有时候，我们需要完全控制一种类型到另一种类型的转换。通常，当一种类型看起来与另一种类型不同时，转换函数已经存在，并且我们希望从“更宽松”类型转变为更强类型，例如将`string`的源类型到目标类型`Int32`。

比如我们需要将以下类型：
```csharp
public class Source
{
    public string Value1 { get; set; }
    public string Value2 { get; set; }
    public string Value3 { get; set; }
}
```
转换为：
```csharp
public class Destination
{
    public int Value1 { get; set; }
    public DateTime Value2 { get; set; }
    public Type Value3 { get; set; }
}
```
当我们尝试按之前的规则映射这两种类型，`AutoMapper`会抛出异常（在映射时和配置检查时）。因为`AutoMapper`不知道从字符串到`int`，`DateTime`或`Type`的任何映射。我们必须提供自定义类型转换器（Custom Type Converters）来做映射，我们有三种方法：
```csharp
void ConvertUsing(Func<TSource, TDestination> mappingFunction);
void ConvertUsing(ITypeConverter<TSource, TDestination> converter);
void ConvertUsing<TTypeConverter>() where TTypeConverter : ITypeConverter<TSource, TDestination>;
```
然后为`AutoMapper`提供自定义类型转换器的实例，或者只提供`AutoMapper`将在运行时实例化的类型。然后，我们上面的源/目标类型的映射配置变为：
```csharp
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<string, int>().ConvertUsing(s => Convert.ToInt32(2));
    cfg.CreateMap<string, DateTime>().ConvertUsing(new DateTimeTypeConverter());
    cfg.CreateMap<string, Type>().ConvertUsing(new TypeTypeConverter());
    cfg.CreateMap<SourceCustomTypeConverter, DestinationCustomTypeConverter>();
});
var mapper = config.CreateMapper();
var dest= mapper.Map<DestinationCustomTypeConverter>(new SourceCustomTypeConverter { Value1 = "1", Value2 = "2018-02-25", Value3 = $"{typeof(DestinationCustomTypeConverter)}" });
```
```csharp
public class SourceCustomTypeConverter
{
    public string Value1 { get; set; }
    public string Value2 { get; set; }
    public string Value3 { get; set; }
}
public class DestinationCustomTypeConverter
{
    public int Value1 { get; set; }
    public DateTime Value2 { get; set; }
    public Type Value3 { get; set; }
}
public class DateTimeTypeConverter : ITypeConverter<string, DateTime>
{
    public DateTime Convert(string source, DateTime destination, ResolutionContext context)
    {
        return System.Convert.ToDateTime(source);
    }
}
public class TypeTypeConverter : ITypeConverter<string, Type>
{
    public Type Convert(string source, Type destination, ResolutionContext context)
    {
        return Assembly.GetExecutingAssembly().GetType(source);
    }
}
```
在第一个映射中，从`string`到`Int32`，我们使用`C#`的`Convert.ToInt32()`函数（作为方法组提供）。其他两个自定义实现`ITypeConverter`。

# Custom Value Resolvers
有些时候，这个自定义值解析器（Custom Value Resolvers）逻辑是域逻辑，可以直接在我们的域上。 但是，如果此逻辑仅适用于映射操作，则会使我们的源类型混乱并产生不必要的行为。 在这些情况下，`AutoMapper`允许为目标成员配置自定义值解析器。 例如，我们可能希望在映射期间获得计算值：
```csharp
public class Source
{
    public int Value1 { get; set; }
    public int Value2 { get; set; }
}

public class Destination
{
    public int Total { get; set; }
}
```
无论出于何种原因，我们希望Total为源Value属性的总和。 由于某些其他原因，我们不能或不应该将此逻辑放在我们的Source类型上。 要提供自定义值解析器，我们需要首先创建一个实现IValueResolver的类型：
```csharp
public class CustomResolver : IValueResolver<Source, Destination, int>
{
    public int Resolve(Source source, Destination destination, int destMember, ResolutionContext context)
    {
        return source.Value1 + source.Value2;
    }
}
```
其中，`ResolutionContext`包含当前解析操作的所有上下文信息，例如源类型，目标类型，源值等。
当我们有了`IValueResolver`实现，我们就需要告诉`AutoMapper`在解析特定目标成员时使用这个自定义值解析器。我们有3种方法可以告诉`AutoMapper`使用自定义值解析器:
- MapFrom<TValueResolver>
- MapFrom(typeof(CustomValueResolver))
- MapFrom(aValueResolverInstance)

在下面的示例中，我们将使用第一种方式，通过泛型告诉`AutoMapper`自定义解析器类型：
```csharp
var config = new MapperConfiguration(cfg =>
{
    cfg.CreateMap<Source, Destination>()
    .ForMember(dest => dest.Total, opt => opt.MapFrom<CustomResolver>());
});
var source = new Source { Value1 = 2, Value2 = 3 };
var mapper = config.CreateMapper();
var dest = mapper.Map<DestinationCustomValueResolver>(source);
```
虽然目标成员（Total）没有任何匹配的源成员，但指定自定义值解析器使配置有效，因为解析器现在负责为目标成员提供值。 如果我们不关心我们的值解析器中的源/目标类型，或者想要通过映射重用它们，我们可以使用“object”作为源/目标类型：
```csharp
public class MultBy2Resolver : IValueResolver<object, object, int> {
    public int Resolve(object source, object dest, int destMember, ResolutionContext context) {
        return destMember * 2;
    }
}
```
以上示例中，我们只向`AutoMapper`提供了自定义解析器的类型，所以映射引擎将使用反射来创建值解析器的实例。 如果我们不希望`AutoMapper`使用反射来创建实例，我们可以直接提供它：
```csharp
Mapper.Initialize(cfg => cfg.CreateMap<Source, Destination>()
    .ForMember(dest => dest.Total,
        opt => opt.MapFrom(new CustomResolver())
    ));
```
`AutoMapper`将使用该特定对象，在解析器可能具有构造函数参数或需要由`IOC`容器构造的情况下非常有用。

# 我们在实际开发中如何使用？
我们如果要使用`Automapper`作为对象映射框架，我们首先需要对其进行初始化操作，可使用一下两种方式：
> 初始化操作我们只应该进行一次，可放在应用程序启动的入口进行。初始化操作相对而言，耗时较长，在初始化完成后，对对象进行映射的速度是很快的。
```csharp
Mapper.Initialize(cfg => cfg.CreateMap<Order, OrderDto>());
//or
var config = new MapperConfiguration(cfg => cfg.CreateMap<Order, OrderDto>());
```
然后创建映射关系：
```csharp
var mapper = config.CreateMapper();
// or
var mapper = new Mapper(config);
```
最后进行对象映射：
```csharp
OrderDto dto = mapper.Map<OrderDto>(order);
// or
OrderDto dto = Mapper.Map<OrderDto>(order);
```

当然我们也可以通过创建映射配置文件的方式进行
```csharp
public class Source2DestinationProfile : Profile
{
    public Source2DestinationProfile()
    {
        CreateMap<Source, Destination>()
            .ForMember(dest => dest.ViewName, options => options.MapFrom(src => src.DBName));
    }
}
class Source
{
    public string DBName { get; set; }
}
class Destination
{
    public string ViewName { get; set; }
}
```
初始化及映射：
```csharp
AutoMapper.Mapper.Initialize(cfg =>
{
    //cfg.AddProfile(new Source2DestinationProfile());
    cfg.AddProfile<Source2DestinationProfile>();
});
Source src = new Source() { DBName = "test" };
var dest = src.MapTo<Destination>();
```

# 其他
还有一些其他转换器和功能，访问[AutoMapper](http://docs.automapper.org/en/stable/index.html)了解详情。

# 附录
[本文GitHub源码](https://github.com/zhaobingwang/sample/tree/master/src-demo/AutoMapperConsoleApp)

[AutoMapper官方文档](http://docs.automapper.org/en/stable/index.html)

