---
layout: post
title: How To Modify Collections In O(1) Time
description: Using the decorator design pattern to modify list data without iterating through it.
readtime: 6 minute
toc: true
tags: programming, c#
---

## Ways To Modify a Collection

Let's consider trying to increment every value in a list, there's a few ways to do it.

### Iteration Approach [O(n)]

A classic, but slow, approach involves iterating over every item in the list and incrementing it. For bigger lists, this can get quite slow.

```cs
List<int> numbers = [1, 2, 3];
Print(numbers); //"1, 2, 3"
for (int i = 0; i < numbers.Count; i++)
{
    numbers[i] += 1;
}
Print(numbers); //"2, 3, 4"
```

### Enumeration Approach [O(1) best usage, O(n) worst usage]

Many languages implement disposable iterators (like C#'s [LINQ](https://www.c-sharpcorner.com/UploadFile/6bcc95/understanding-projections-in-linq-with-select-selectmany-e/) and Java's [Stream API](https://docs.oracle.com/javase/8/docs/api/java/util/stream/Stream.html)), whereby an element is transformed when it is encountered during iteration.

Using these APIs, you can produce an enumeration which will increment elements when they are encountered (lazily):

```cs
List<int> numbers = [1, 2, 3];
IEnumerable<int> incrementedEnumeration = numbers.Select(i => i + 1);
//you can later consume `incrementedEnumeration` to yield `2, 3, 4`.
```

The problem with this approach is that it is often used incorrectly. For example, if you produce a transformative enumeration but then convert it back into a list, you will have just undone all your hard work - because `Enumeration#ToList` implementations have to iterate over all the data to create a list from it.

If you need to iterate over the data at the end of your data pipeline and not before, then it might be worth using this approach. But, if you're ever intending to access data by index, or iterate over it more than once, then transformative enumerations are almost definitely going to result in unnecessary iterations and processing time.

### List-View Approach [O(1) always]

By leveraging the [decorator design pattern](https://refactoring.guru/design-patterns/decorator), we can achieve O(1) time complexity for modifications to list elements. The example I'll provide below is just for lists, but the idea can be adapted to all types of collections.

The solution here is to create a class that wraps around a list, and transforms the values before returning them - better called a transformative list-view. Using the decorator pattern in this way causes elements in the decorated list to be [lazily](https://en.wikipedia.org/wiki/Lazy_evaluation) incremented.

The basic C# example of this is as follows:

```cs
//list wrapper that adds one to every element of an integer list
public class AddsOneToElements : IReadOnlyList<int>
{
    private readonly IReadOnlyList<int> _underlyingData;

    public AddsOneToElements(IReadOnlyList<int> underlyingData)
    {
        this._underlyingData = underlyingData;
    }

    public int Count => _underlyingData.Count;

    /// <summary>
    /// Adds one to the element at <paramref name="index"/>
    /// before returning it.
    /// </summary>
    public int this[int index] => _underlyingData[index] + 1;

    public IEnumerator<int> GetEnumerator()
    {
        for (int i = 0; i < Count; i++)
            yield return this[i];
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

Then you would just use this decorated list as if it was a normal list:

```cs
List<int> numbers = [1, 2, 3];
IReadOnlyList<int> incremented = new AddsOneToElements(numbers);

Print(numbers);     //1, 2, 3
Print(incremented); //2, 3, 4
```

For context here, [IReadOnlyList](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.ireadonlylist-1?view=net-9.0) is an interface inherited by [List](https://learn.microsoft.com/en-us/dotnet/api/system.collections.generic.list-1?view=net-9.0) and the standard array type; it exposes the properties `Count` and `this[index]`, as well the `GetEnumerator` methods.

## Making a Reusable, Mapping List-View

The above list-view is powerful but single-purposed at the moment. However, it can be made much more useful and adaptable.

Instead of just adding one to the elements, we can create a list-view that applies an arbitrary mapping function to the data. Furthermore, instead of just consuming and producing integers, we can make the list-view generic.

```cs
public class MappingListView<TIn, TOut> : IReadOnlyList<TOut>
{
    private readonly IReadOnlyList<TIn> _underlyingData;
    private readonly Func<TIn, TOut> _mapFunc;

    public MappingListView(IReadOnlyList<TIn> underlyingData,
                           Func<TIn, TOut> mapFunc)
    {
        this._underlyingData = underlyingData;
        this._mapFunc = mapFunc;
    }

    public int Count => _underlyingData.Count;

    /// <summary>
    /// Applies the arbitrary mapping function to the element
    /// at <paramref name="index"/> before returning it.
    /// </summary>
    public TOut this[int index] => _mapFunc(_underlyingData[index]);

    public IEnumerator<TOut> GetEnumerator()
    {
        for (int i = 0; i < Count; i++)
            yield return this[i];
    }

    IEnumerator IEnumerable.GetEnumerator() => GetEnumerator();
}
```

Now what we have is an incredibly powerful, transformative list-view:

```cs
List<int> numbers = [1, 2, 3];
Print(numbers); //1, 2, 3

IReadOnlyList<int> doubled =
    new MappingListView<int, int>(numbers, x => x * 2);
Print(doubled); //2, 4, 6

IReadOnlyList<string> strings =
    new MappingListView<int, string>(doubled, x => x + " bottles");
Print(strings); //"2 bottles", "4 bottles", "6 bottles"
```

### C# Extension Method Idiom

C# has a concept called [extension methods](https://learn.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/extension-methods), we can use this to build a much cleaner API for creating these list views:

```cs
public static class ReadOnlyListExtensions
{
    public static IReadOnlyList<TOut> SelectView<TIn, TOut>(
        this IReadOnlyList<TIn> data, Func<TIn, TOut> mapFunc)
    {
        return new MappingListView<TIn, TOut>(data, mapFunc);
    }
}
```

Now when we do `IReadOnlyList#SelectView` it will create a mapping list-view for us:

```cs
List<int> numbers = [1, 2, 3];
Print(numbers); //1, 2, 3

IReadOnlyList<int> doubled = numbers.SelectView(x => x * 2);
Print(doubled); //2, 4, 6

IReadOnlyList<string> strings = doubled.SelectView(x => $"{x} bottles");
Print(strings); //"2 bottles", "4 bottles", "6 bottles"
```

## Real World Use Case

Let's say you're using a graphing library which consumes a list of `Coordinate` objects (not an enumeration). On every render call, this library is going to iterate through all the points and render them:

```cs
public void Render(IReadOnlyList<Coordinate> toRender)
{
  for (int i = 0; i < toRender.Count; i++) {
    DrawPoint(toRender[i]);
  }
}
```

Of course, in this example, it would make more sense for the `Render` method to consume an enumeration instead of a list, but let's say for the sake of argument that the points need to be index accessible.

If you have a list of `Coordinate` objects which you want to render, it's super simple:

```cs
List<Coordinate> coordinates = CalculateCoordinates();
Render(coordinates);
```

But if you intend to transform that list in any way before it gets to the rendering library then you may make the mistake of using an enumeration API as follows:

```cs
List<Coordinate> coordinates = CalculateCoordinates();
Func<Coordinate, Coordinate> mapFunc = coord => /* ... */;
List<Coordinate> mappedCoordinates = coordinates
                                 .Select(mapFunc) //modify the coorindates
                                 .ToList(); // !!!

//you've just iterated over every coordinate when you called `toList`,
//and now you're going to do it *again* in the render code.
//you've also allocated memory for a new list with the same size as the original.

Render(mappedCoordinates);
```

In actual fact, you should have created a list-view that transformed the coordinates when they were accessed in the render method.

```cs
List<Coordinate> coordinates = CalculateCoordinates();
Func<Coordinate, Coordinate> mapFunc = coord => /* ... */;
IReadOnlyList<Coordinate> mappedCoords =
    new MappingListView<Coordinate, Coordinate>(coordinates, mapFunc);

//no unnecessary memory allocations or list iterations
Render(mappedCoords);
```

For just a few thousand coordinates, the impact of an enumeration API will be negligable, but when you start talking about millions of coordinates, you will noticably feel the delay from unnecessarily iterating over them all.

### No Seriously, It's Real

This is almost exactly one the changes I submitted in a [pull request](https://github.com/ScottPlot/ScottPlot/pull/5129) to the open-source, graphing library [ScottPlot](https://scottplot.net/).

Before every render, a scatter graph would call `GetScatterPoints()`, which was implemented as follows:

```cs
public IReadOnlyList<Coordinates> GetScatterPoints()
{
    return Coordinates
        .Skip(MinRenderIndex)
        .Take(this.GetRenderIndexCount())
        .ToList();
    //but this `ToList()` call iterates over every single
    // coordinate in the list, and allocates a new list.
}
```

For many thousands of points, there were notable performance implications.

Instead, I proposed to wrap the underlying coordinates in two list-views which supported "skipping" and "taking" elements via offsetting the view and amending the reported size:

```cs
public IReadOnlyList<Coordinates> GetScatterPoints()
{
    return Coordinates
        .SkipView(MinRenderIndex)
        .TakeView(this.GetRenderIndexCount());
    //no allocations, no iterations, just a pure
    // view over the original data.
}
```

## Important Considerations

There are many great applications of list-views; they can be used almost anywhere you want to modify a collection. You can use them to resize lists, re-sample data within lists, skip over elements in a list, and much more.

Whilst they are very useful, there are some points to consider when you are using list-views, to ensure you get the most out of them.

### Reusability

Because of the fact that changes to the data (such as re-mapping in the `MappingListView`) are completed upon iteration of the list-view, it means that these changes will be applied multiple times if the list-view is iterated multiple times. This could be problematic if the changes are computationally expensive.

A mitigation strategy for this would be to create a cache in the list-view which gets populated when the data is first iterated, and then used for subsequent access requests. This would be effective, but equally has its own drawbacks as the space complexity of the list-view is now O(n), and it's not even really a list-view anymore, it's just a glorified cache.

For this reason, if the list-view is performing computationally expensive operations, I'd recommend re-using it as few times as possible. However, if it is performing relatively cheap computation (like a simple re-map or offset), it is very viable to re-use the list-view.

### Still Consider An Enumeration

This article isn't trying to persuade you to ditch enumerations, they absolutely have their place and it's very likely that you may be able to utilise them effectively in your solution.

However, scenarios where you cannot use an enumeration, or would end up calling collector methods (such as `ToList()` - which will iterate all the data and allocate memory for it), should be thoughtfully reviewed, with list-views strongly considered.
