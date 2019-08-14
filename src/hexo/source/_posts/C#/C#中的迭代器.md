---
title: C#中的迭代器
categories: C#
comments: true
date: 2019/08/14 21:53:33
updated: 2019/08/14 21:53:33
tags:
    - C#
    - 迭代器
---

# 迭代器
在C#中，foreach语句使得能够进行比for循环语句更直接和简单的对集合的迭代。.NET中[迭代器](https://baike.baidu.com/item/%E8%BF%AD%E4%BB%A3%E5%99%A8/3803342?fr=aladdin)是通过`IEnumerable`和`IEnumerator`接口来实现的(当然，这两个接口还有其对应的泛型版本：`IEnumerable<T> `和 `IEnumerator<T>`)。其源代码如下（部分代码省略）

```csharp
public interface IEnumerable
{
    // Interfaces are not serializable
    // Returns an IEnumerator for this enumerable Object.  The enumerator provides
    // a simple way to access all the contents of a collection.
    IEnumerator GetEnumerator();
}
```
[点击查看详细源码](https://referencesource.microsoft.com/#q=IEnumerable)

```csharp
public interface IEnumerator
{
    // Interfaces are not serializable
    // Advances the enumerator to the next element of the enumeration and
    // returns a boolean indicating whether an element is available. Upon
    // creation, an enumerator is conceptually positioned before the first
    // element of the enumeration, and the first call to MoveNext 
    // brings the first element of the enumeration into view.
    // 
    bool MoveNext();

    // Returns the current element of the enumeration. The returned value is
    // undefined before the first call to MoveNext and following a
    // call to MoveNext that returned false. Multiple calls to
    // GetCurrent with no intervening calls to MoveNext 
    // will return the same object.
    // 
    Object Current {
        get; 
    }

    // Resets the enumerator to the beginning of the enumeration, starting over.
    // The preferred behavior for Reset is to return the exact same enumeration.
    // This means if you modify the underlying collection then call Reset, your
    // IEnumerator will be invalid, just as it would have been if you had called
    // MoveNext or Current.
    //
    void Reset();
}
```
[点击查看详细源码](https://referencesource.microsoft.com/#q=IEnumerator)

# 一个基础例子
以下两个代码片段（生成从 0 到 9 的整数序列）等效，我们在 `C#` 中使用 `yield return` 上下文关键字定义迭代器方法。
```csharp
private static IEnumerable<int> GetSingleDigitNumbers()
{
    yield return 0;
    yield return 1;
    yield return 2;
    yield return 3;
    yield return 4;
    yield return 5;
    yield return 6;
    yield return 7;
    yield return 8;
    yield return 9;
}
```
```csharp
private static IEnumerable<int> GetSingleDigitNumbers()
{
    int index = 0;
    while (index++ < 10)
        yield return index - 1;
}
```

# 让我们自定义的数据类型实现迭代器
```csharp
public class MyIEnumerable : IEnumerable
{
    private string[] _strList;
    public MyIEnumerable(string[] strList)
    {
        _strList = strList;
    }
    public IEnumerator GetEnumerator()
    {
        //return new MyIEnumerator(_strList);
        for (int i = 0; i < _strList.Length; i++)
        {
            yield return _strList[i];
        }
    }
}
public class MyIEnumerator : IEnumerator
{
    private string[] _strList;
    private int position;
    public MyIEnumerator(string[] strList)
    {
        _strList = strList;
        position = -1;
    }
    public object Current
    {
        get { return _strList[position]; }
    }

    public bool MoveNext()
    {
        position++;
        if (position < _strList.Length)
            return true;
        return false;
    }

    public void Reset()
    {
        position = -1;
    }
}
```

调用
```csharp
public static void RunMyIEnumerable()
{
    string[] strList = new string[] { "第一个节点数据", "第二个节点数据", "第三个节点数据" };
    MyIEnumerable myIEnumerable = new MyIEnumerable(strList);
    // 1.获取IEnumerator接口实例
    var enumerator = myIEnumerable.GetEnumerator();

    // 2.判断是否可以继续循环
    while (enumerator.MoveNext())
    {
        // 3.取值
        Console.WriteLine(enumerator.Current);
    }
    Console.WriteLine("==========");
    foreach (var item in myIEnumerable) // 效果等同于上述方式
    {
        Console.WriteLine(item);
    }
}
```
> 我们调用GetEnumerator的时候，看似里面for循环了一次，其实这个时候没有做任何操作。只有调用MoveNext的时候才会对应调用for循环。
> 这也是`为什么Linq to Object中要返回IEnumerable?`因为IEnumerable是延迟加载的，每次访问的时候才取值。也就是我们在Lambda里面写的where、select并没有循环遍历(只是在组装条件)，只有在ToList或foreache的时候才真正去集合取值了。这样大大提高了性能。
以上引用自[农码一生](https://www.cnblogs.com/zhaopei)的博客[先说IEnumerable，我们每天用的foreach你真的懂它吗？](https://www.cnblogs.com/zhaopei/p/5769782.html)

# 参考
> 本文引用以下文章

[Iterators](https://docs.microsoft.com/zh-cn/dotnet/csharp/iterators)

[先说IEnumerable，我们每天用的foreach你真的懂它吗？](https://www.cnblogs.com/zhaopei/p/5769782.html)

[匹夫细说C#：庖丁解牛迭代器，那些藏在幕后的秘密](https://www.cnblogs.com/murongxiaopifu/p/4437432.html)
