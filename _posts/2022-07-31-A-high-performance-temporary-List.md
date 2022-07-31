Hi, this is my first post and I wanted to make it about performance.

Here's the story, while coding the gameplay logic, the systems in our game use lots of temporary lists which are discarded by the end of frame. 

If we use the C# list in the System namespace, we create a lot of garbage and the GC has to run frequently. The CPU usage spikes caused by the GC is not acceptable for our games, even if it happens for some frames.

Today I want to talk about list optimizations I've done over time, merged in one data structure which I will call StackOnlyList. It's also inspired by [ValueStringBuilder](https://andrewlock.net/a-deep-dive-on-stringbuilder-part-6-vaulestringbuilder-a-stack-based-string-builder/) implementation.

I'm assuming the reader knows how List data structure works internally, just an array with keeping track of index and count, so I won't be focusing on the details of it.

Here's the aim of StackOnlyList:

1. Allow using stack allocated memory as an initial memory
2. Fallback to pooled memory if the list needs to grow or the stack memory is not available
3. Should be simple to use and hard to misuse
4. Use most recent C# features for performance

Let's start with the signature and fields of our list:

```csharp
public ref struct StackOnlyList<T> where T : IEquatable<T>
{
    // These fields are internal because they are used in unit tests
    internal Span<T> Span;
    internal T[] ArrayFromPool;
    public int Capacity { get; private set; }
    public int Count { get; private set; }

    ...
}
```

The list is a [ref struct]( **https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/builtin-types/struct#ref-struct**), which allows us to store stack allocated memory, which will be assigned to the [Span](https://docs.microsoft.com/en-us/dotnet/api/system.span-1?view=net-6.0) field. Ref structs can't escape to the managed heap. For example, you can't cache it on a class field. It's good for our implementation, since our list is only meant to be used for temp memory.

`ArrayFromPool` field is to store arrays returned from the shared memory, we'll eventually return the array so we keep a reference to it.

Notice that I've forced generic type *T* to implement the `IEquatable` interface. It's an extra optimization when working with structs, as using the default `Equals` implementation **causes boxing for structs.**

Here's the the constructor for stack memory:

```csharp
public StackOnlyList(Span<T> initialBuffer)
{
    ArrayFromPool = null;
    Span = initialBuffer;
    Capacity = initialBuffer.Length;
    Count = 0;
}
```

We can use the constructor like this:

```csharp
// The first 32 items come for free!
// Notice that stackalloc memory is implicitly cast to Span<T> constructor parameter
var list = new StackOnlyList<int>(stackalloc int[32]);
```

You need to be **very careful** when using this constructor though:

```cs
// This method may crash your program depending on the 'size' parameter and the size of the generic type T. If the size is a parameter you shouldn't use this constructor!
void PotentiallyDangerousMethod(int size)
{
    var list = new StackOnlyList<int>(stackalloc int[size]);
}

// Calling this method will throw StackOverflowException, but you can't even catch it (with a try/catch block), it's going to crash your program!
void DefinitelyDangerousMethod()
{
    for(int i = 0; i < 100_000; i++)
    {
        var list = new StackOnlyList<int>(stackalloc int[32]);
    }
}

// This method is safe, as each call to TempList will release the stackalloc memory before the next one is called.  
void SafeMethod()
{
    for(int i = 0; i < 100_000; i++)
    {
        TempList();
    }
}

void TempList()
{
    var list = new StackOnlyList<int>(stackalloc int[32]);
}
```

As you can see, there are some pitfalls, but that's ok as long as we know what we're doing.

Now, let's see the *safe* constructor:

```csharp
public StackOnlyList(int initialCapacity = 0)
{
	switch(initialCapacity)
	{
		case < 0:
		{
			throw new ArgumentOutOfRangeException($"Negative capacity '{initialCapacity}' is not allowed.");
		}
		case 0:
		{
			ArrayFromPool = null;
			Span = null;
			Capacity = 0;
			Count = 0;
			break;
		}
		case > 0:
		{
			ArrayFromPool = ArrayPool<T>.Shared.Rent(initialCapacity);
			Span = ArrayFromPool;
			Capacity = ArrayFromPool.Length;
			Count = 0;
			break;
		}
	}
}
```

We're using [ArrayPool.Shared](https://docs.microsoft.com/en-us/dotnet/api/system.buffers.arraypool-1.shared?view=net-6.0#system-buffers-arraypool-1-shared) for our pooled memory implementation, it's fast and thread-safe.

Notice that when using the ```Rent``` method, the returned Array will have greater or equal size of the passed parameter ```initialCapacity```, not the exact amount. 

Also, we will never use the ```ArrayFromPool``` field for accessing elements, always use the ```Span```, as it can be used for both *stack* and *pooled* mode.



We need to return this memory after we're done with it, so let's add a dispose method:

```csharp
public void Dispose()
{
    var toReturn = ArrayFromPool;

    // Prevent using existing data, if this struct is erroneously used after it is disposed.
    this = default;

    if(toReturn != null)
    {
        ArrayPool<T>.Shared.Return(toReturn);
    }
}
```

How to use it:

```csharp
public void ListExample()
{
    // Notice we're not using stackalloc constructor anymore.
    using var listWithAutoDispose = new StackOnlyList<int>(32);
 	// Do some operations... and it's auto disposed!
    
    // You need to manually dispose this!
    var listWithManualDispose = new StackOnlyList<int>(32);
	// Do some operations...
    listWithManualDispose.Dispose();
}
```

We're using [using declarations](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/proposals/csharp-8.0/using) which requires C# 8.0.

Let's see the Add method:

```csharp
public void Add(in T item)
{
    if(Capacity == Count)
    {
        var desiredCapacity = Capacity == 0 ? 4 : 2 * Capacity;
        Grow(desiredCapacity);
    }

    Span[Count++] = item;
}

void Grow(int desiredCapacity)
{
    if(Capacity == 0)
    {
        var newArray = ArrayPool<T>.Shared.Rent(desiredCapacity);
        ArrayFromPool = newArray;
        Span = newArray;
        Capacity = newArray.Length;
    }
    else
    {
        // First copy to new array, then return to the pool
        var previousSpan = Span;
        var newArray = ArrayPool<T>.Shared.Rent(desiredCapacity);
        Span = newArray;
        previousSpan.CopyTo(Span);

        if(ArrayFromPool != null)
        {
            ArrayPool<T>.Shared.Return(ArrayFromPool);
        }

        ArrayFromPool = newArray;
        Capacity = newArray.Length;
    }
}
```

First we need to check if we have enough memory, we do that by `Capacity == Count`, then we resize if required, after that set the item at the add index.

If the current capacity is zero, it's a simpler path. If capacity is greater than zero, we need to handle copying previous Span to the new Span, and releasing the pooled memory if it exists. Notice that we have to check `ArrayFromPool != null `, since we might be using `stackalloc` buffer initially and in that  case we don't have pooled memory yet.

Also we're passing the parameter with [in](https://docs.microsoft.com/en-us/dotnet/csharp/language-reference/keywords/in-parameter-modifier) keyword, which passes the argument as read-only reference. This is another optimization for working with big structs, we don't want them copied.

{% highlight csharp %}

public ref T this[int index]
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    get
    {
        CheckIndexGreaterOrEqualToCountAndThrow(index);
        return ref Span[index];
    }
}

{% endhighlight %}



See indexing to our list:

```cs
public ref T this[int index]
{
    [MethodImpl(MethodImplOptions.AggressiveInlining)]
    get
    {
        CheckIndexGreaterOrEqualToCountAndThrow(index);
        return ref Span[index];
    }
}

void CheckIndexGreaterOrEqualToCountAndThrow(int index)
{
    if(index >= Count)
        throw new IndexOutOfRangeException($"Index '{index}' is out of range. Count: '{Count}'.");
}
```

I've used [ref returns](https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/classes-and-structs/ref-returns) for the indexer, the default C# list doesn't allow this. It can be handy in some cases:

```cs
void RefReturnExample()
{
    using var list = new StackOnlyList<Matrix4x4>(initialCapacity:16);
    
    // ... Assume some list.Add done here

    // A 4x4 matrix has 16 float fields, total of 64 bytes, we don't want to copy it!
	ref var matrix = ref list[0];
    // You can also have read-only reference
    ref readonly var readonlyMatrix = ref list[0];
    
  	// We can also copy it, but the performance gains are gone!
    var matrix = list[0];
}
```

But there's some dangerous cases:

```csharp
void RefReturnDangerousUsage()
{
    using var list = new StackOnlyList<int>(stackalloc int[2]); 
    list.Add(0);

    ref var firstValue = ref list[0];
    list[0] = 1;
    // firstValue is also 1 after this assignment.
    
    // These adds cause the list to resize, we're switched to pooled memory.
    list.Add(0);
    list.Add(0);
    list.Add(0);
    
    list[0] = 2;
    // firstValue is not 2! It's still pointing to the stack memory, but we've switched to using pooled memory!
}
```

Now I will add the remaining important methods for performance:

```csharp
[MethodImpl(MethodImplOptions.AggressiveInlining)]
public Span<T> AsSpan()
{
    return Span[..Count];
}

[MethodImpl(MethodImplOptions.AggressiveInlining)]
public ReadOnlySpan<T> AsReadOnlySpan()
{
    return AsSpan();
}

[MethodImpl(MethodImplOptions.AggressiveInlining)]
public Span<T>.Enumerator GetEnumerator()
{
    return AsSpan().GetEnumerator();
}
```

I used `AggressiveInlining` in many of my methods. I didn't benchmark it, but assuming for very small methods it's better to inline. There's some good information about inlining in this post: [To Inline or not to inline](https://docs.microsoft.com/en-us/archive/blogs/vancem/to-inline-or-not-to-inline-that-is-the-question) 

Span-related methods are very important for preventing memory copying. `Span<T>.GetEnumerator` allows you to use very sweet syntax:

```cs
void SpanEnumeratorExample(ref StackOnlyList<Matrix4x4> items)
{
    foreach(ref var item in items)
    {
        TranslateMatrix(ref item);
    }
}

void TranslateMatrix(ref Matrix4x4 matrix)
{
    ...
}
```

Now, we have one last important thing to remember: **We always need to pass the list as ref to the methods:** 

```csharp
void IncorrectListUsage()
{
	var list = new StackOnlyList<int>();
    // list.Count is zero here
    AddToList(list);
    // list.Count is still zero here, since structs are value types they are copied when passed to function!
	// You also failed to Dispose the memory here, since on our local struct no field has changed.
    list.Dispose();
}

void AddToListIncorrectly(StackOnlyList<int> list)
{
	list.Add(0);    
}

void CorrectListUsage()
{
    var list = new StackOnlyList<int>();
    AddToList(ref list);
    list.Dispose();
}

void AddToList(ref StackOnlyList<int> list)
{
	list.Add(0);    
}
```

Rest of the implementation is ordinary. Also I didn't provide the whole IList interface, but you can extend it yourself! You can get the full source code from here: [StackOnlyList](github.com/vectorized-runner/StackOnlyList)

# Summary

- The StackOnlyList is a replacement for default C# List for high performance use, when we require temporary memory.

- There are some cases when the user needs to be careful: Working with stack allocated memory, passing to methods, using ref returns.



Please give some feedback (wherever I shared this!) as I don't have comments implemented here *yet*!

