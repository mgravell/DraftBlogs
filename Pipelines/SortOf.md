# A Sort Of Problem

I *love* interesting questions, especially when they directly relate to things I need to do. A great question came up on Stack Overflow today about [how to efficiently sort large data](https://stackoverflow.com/q/48345753/23354). I gave an answer, but there's *so much more* we can say on the topic, so I thought I'd turn it into a blog entry, exploring pragmatic ways to improve sort performance when dealing with non-trivial amounts of data. In particular, this is remarkably similar to time I've spent trying to make our "tag engine" faster.  

## The problem

So, the premise is this:

- we have a complex entity, `SomeType`, with multiple properties
- we have a large number of these entities - lets say 16M+
- we want to sort this data using a sort that considers multiple properties - "this, then that"
- and we want it to be fast

Note that sorting data when it is already sorted or nearly-sorted is *usually* cheap under most common algorithms, so I'm going to be focusing only on the initial painful sort when the data is not at all sorted.

Because we're going to have so many of them, and they are going to be basic storage types only, this is a good scenario to consider a `struct`, and I was delighted to see that the OP in the question had already done this. We'll play with a few of the properties (for sorting, etc), but to simulate the usual context, there will be extra stuff that isn't relevant to the question, so we'll pad the size of the struct with some dummy fields up to 64 bytes. So, something like:

```
readonly partial struct SomeType
{
    public int Id { get; }
    public DateTime ReleaseDate { get; }
    public double Price { get; }

    public SomeType(int id, DateTime releaseDate, double price)
    {
        Id = id;
        ReleaseDate = releaseDate;
        Price = price;
        _some = _other = _stuff = _not = _shown = 0;
    }

#pragma warning disable CS0414 // suppress "assigned, never used"
    private readonly long _some, _other, _stuff, _not, _shown;
#pragma warning restore CS0414
}
```

Note: yes, I know that `double` is a terrible choice for something that describes money.

Note: `readonly struct` is a new C# feature, [described in more detail here](https://blogs.msdn.microsoft.com/mazhou/2017/11/21/c-7-series-part-6-read-only-structs/) - this is a good fit, and might help us avoid some large "load" costs.

For something interesting to do, we'll try sorting things "most recent, then cheapest".

## Inventing some data

The first thing we need is some data; a very basic seeded random data script might be something like:

```
var rand = new Random(data.Length);
for (int i = 0; i < data.Length; i++)
{
    int id = rand.Next();
    var releaseDate = Epoch
        .AddYears(rand.Next(50))
        .AddDays(rand.Next(365))
        .AddSeconds(rand.Next(24 * 60 * 60));
    var price = rand.NextDouble() * 50000;
    data[i] = new SomeType(
        id, releaseDate, price);
}
```

## Attempt 1: LINQ

LINQ is great; I love LINQ, and it makes some code very expressive. So let's try the most obvious thing first:

```
sorted = (from item in data
        orderby item.ReleaseDate descending,
                item.Price
        select item).ToArray();
```

This LINQ expression performs the sort we want, creating a copy of the data - but on my machine this takes about 17 seconds to run - not ideal. So that's the target to beat. The key thing about LINQ is that it is designed for *your* efficiency, i.e. the size and complexity of the code that you need to write, on the assumption that you'll only use it on reasonable data. We do not have reasonable data here.

## Attempt 2: `IComparable<T>`

Since we're talking about arrays, another obvious thing to do is `Array.Sort`; for the simplest version of that, we need to implement `IComparable<T>` on our type:

```
partial struct SomeType : IComparable<SomeType>
{
    int IComparable<SomeType>.CompareTo(SomeType other)
    {
        var delta = other.ReleaseDate
            .CompareTo(this.ReleaseDate);
        if (delta == 0) // second property
            delta = this.Price.CompareTo(other.Price);
        return delta;
    }
}
```

And then we can use:

```
Array.Sort<SomeType>(data);
```

to perform an in-place sort. The generic `<SomeType>` here is actually redundant, but I've included it to make it obious that I am using the generic API.

This takes just over 6 seconds for me, so: a huge improvement! Note that for the purpose of our tests, we will re-populate the data after this, to oensure that all tests start with randomized data. For brevity, assume we're doing this whenever necessary - I won't keep calling it out.

## Attempt 3: `IComparer<T>`

There's a second common sort API: `IComparer<T>` custom comparers. This has the advantages that a: you don't need to edit the target type, and b: you can support multiple different sorts against the same type via different custom comparers. For this, we add or own comparer:

```
sealed class SomeTypeComparer : IComparer<SomeType>
{
    private SomeTypeComparer() { }
    public static SomeTypeComparer Default { get; } = new SomeTypeComparer();
    int IComparer<SomeType>.Compare(SomeType x, SomeType y)
    {
        var delta = y.ReleaseDate
                .CompareTo(x.ReleaseDate);
        if (delta == 0) // second property
            delta = x.Price.CompareTo(y.Price);
        return delta;
    }
}
```

using it via:

```
Array.Sort<SomeType>(data, SomeTypeComparer.Default);
```

This takes around 8 seconds; we're not going in the right direction here!

## Attempt 4: `Comparison<T>`

Why stop at two ways to do the same thing, when we can have 3? For completeness, there's yet another primary `Array.Sort<T>` variant that takes a delegate, for example:

```
Array.Sort<SomeType>(data, (x, y) =>
{
    var delta = y.ReleaseDate
            .CompareTo(x.ReleaseDate);
    if (delta == 0) // second property
        delta = x.Price.CompareTo(y.Price);
    return delta;
});
```

This keeps the "do a sort" and "like this" code all in the same place, which is nice; but: it performs virtually identically to the previous attempt, at around 8 seconds.

# First intermission: what is going wrong?

We're doing a lot of work here, that much is true; but there are things that are exacerbating the situation:

- we have a large struct, which means we need to copy that data on the stack whenever we do anything
- because it needs to compare values to their neighbours, there are a *lot* of virtual calls going on

These costs are fine for reasonable data, but for larger volumes the costs start building up.

We need an alternative.

It happens that `Array.Sort` also has overloads that accept *two* arrays - the keys and the values. What this does is: perform the sort logic on the *first* array, but whenever it swaps data around: it swaps the corresponding items in **both** arrays. This has the effect of sorting the *second* array by the values of the *first*. In visual terms, it is like selecting two columns of a spreadsheet and clicking sort.

If we only had a single value, this would be great! For example...

## Attempt 5: dual arrays, single property

Let's pretend for a moment that we only want to sort by the date, in ascending order. Which isn't what we want, but: humour me.

What we *could* do is keep a `CreationDate[]` hanging around (reuse it between operations), and when we want to sort: populate the data we want to sort by into this array:

```
for (int i = 0; i < data.Length; i++)
    releaseDates[i] = data[i].ReleaseDate;
```

and then to sort:

```
Array.Sort(releaseDates, data);
```

For me, this takes about 150ms to prepare the keys, and 4.5s to execute the sort. Promising, although hard to tell if that is useful until we can handle the complex dual sort.

# Second intermission: how can we compose the sort?

We have two properties that we want to sort by, and a `Sort` method that only takes a single value. We could start looking at tuple types, but that is just making things even more complex. What we want is a way to *simplify* the complex sort into a single value. What if we could use something simple like an integer to represent our combined sort? Well, we can!

Many basic values can - either directly, or via a hack - be treated as a bitwise-sortable value. By bitwise sortable, I essentially mean: "sorts like the samme bits expressed as an unsigned integer would sort". Consider a 32-bit integer: obviously an unsigned integer sorts *just like* an unsigned integer, but a *signed* integer does not - negative numbers are problematic. What would be great is if `int.MinValue` was treated as 0, with `int.MinValue + 1` treated as 1, etc; we can do that by subtraction:

```
protected static ulong Sortable(int value)
{
    // re-base eveything upwards, so anything
    // that was the min-value is now 0, etc
    var val = unchecked((uint)(value - int.MinValue));
    return val;
}
```

The result of this is that `Sortable` will return 32-bits worth of data (the same as the input), but with `000...000` as the minimum expected value, and `111...111` as the maximum expected value.

Now; notice that we're only talking about 32 bits here, but we've returne a `ulong`; that's because we're going to pack *2* values into a single token.

For our *actual* data, we hae two pieces:

- a `DateTime`
- a `Double`

Now, that's 16 bytes worth of data, and we only have 8 to play with. This sounds like a dilemma, but *usually*: we can cheat by fudging the precision.

For many common applications - and especially things like a `ReleaseDate`, most of the bits in a `DateTime` are not useful. We probably don't need to handle every tick in a 10,000-year range. We can *almost certainly* use per-second precision - perhaps even per-*day* for a release date. Unix time in seconds using 32 bits has us covered until January 19, 2038. If we need *less precision than seconds*, we can extend that hugely; and we can often use a different epoch that fits our minimum expected data. Heck, starting at the year 2000 instead of 1970 buys 30 years even in per-second precision. Time in an epoch is bitwise-sortable.

Likewise, an obvious way of approximating a `double` in 32 bits would be to cast it as a `float`. This doesn't have the same range or precision, but *will usually be just fine* for sorting purposes. Floating point data in .NET [has a complex internal structure](https://en.wikipedia.org/wiki/IEEE_754), but fortunately making it bitwise-sortable can be achieved through some simple well-known bit hacks:

```
protected static unsafe ulong Sortable (float value)
{
    const int MSB = 1 << 31;
    int raw = *(int*)(&value);
    if ((raw & MSB) != 0) // IEEE first bit is the sign bit
    {
        // is negative; shoult interpret as -(the value without the MSB) - not the same as just
        // dropping the bit, since integer math is twos-complement
        raw = -(raw & ~MSB);
    }
    return Sortable(raw);
}
```

Putting these together, we havev all the tools we need to create a single composite value that is **totally meanningless** for all ordinary purposes, but which represents our sort perfectly.

## Attempt 6: dual arrays, dual property

We can create a method that composes our two properties:

```
static ulong Sortable(in SomeType item)
{
    return (~(ulong)item.ReleaseDate.ToMillenialTime()) << 32
                | Sortable((float)item.Price);
}
```

This might look complex, but what it does is:

- compute the time in seconds since 2000 as a 32-bit unsigned integer
- extends it to 64-bits
- *inverts it*; this has the same effect as "descending", since it reverses the order
- left shifts it by 32 bits, to place those 32 bits in the **upper** half of our 64 bits (padding on the right with zero)
- compute the bitwise-sortable representation of the price as a 32-bit unsigned integer
- throws that value into the lower 32 bits

We can prepare our data into a `ulong[]` that we keep around between sort operations:

```
for (int i = 0; i < data.Length; i++)
    sortKeys[i] = Sortable(in data[i]);
```

and finally sort:

```
Array.Sort(sortKeys, data);
```

The prepare operation is more complex now - and has gone up to 300ms, but the sort is *faster* at just over 4 seconds. We're moving in the right direction. Note that the prepare operation is [embarrassingly parallelizable](https://en.wikipedia.org/wiki/Embarrassingly_parallel), so we can trivially divide that over a number of cores (say: 16 blocks of 1M records per block) - and can often be further reduced by storing the data in the struct in similar terms to the sortable version (so: the same representation of time, and the same floating-point scale) - thus I'm not going to worry about the prepare cost here.

But we're still paying a lot of overhead from having to move around those big structs. We can avoid that by... just not doing that!

## Attempt 7: indexed

Rather than sorting our `SomeType[]` array, we could instead *leave that data alone*, forever. Never move the items around (although it is usually fine to replace them with updates). This has multiple advantages, but the one we're keen on is the reduction of cost copying the data.

So; we can declare an `int[] index` that is our *index* - it just tells us the offsets to look in the *actual* data. We can sort that index *as though* it were the actual data, and just make sure we go through the index. We need to initialize the index as well as the composite sortable value (although we can re-use the positions if we are re-sorting the data, as usually the data doesn't move much between cycles - we'll get another huge boost on re-sorts when the data hasn't drifted much; we *do not* need to reset the index when re-sorting the same data):

```
for (int i = 0; i < data.Length; i++)
{
    index[i] = i;
    sortKeys[i] = Sortable(in data[i]);
}
```

and sort:

```
Array.Sort(sortKeys, index);
```

The only complication is that now, to access the sorted data - instead of looking at `data[i]` we need to look at `data[index[i]]`, i.e. find the i'th item in the index, and use *that value* as the offset in the actual data.

This takes the time down to 3 seconds - we're getting there.

## Attempt 8: indexed, direct compare, no range checks

The introspective sort that `Array.Sort` does is great, but it is still going to be talking via the general `CompareTo` API on our key type (`ulong`), and using the array indexers extensively. The JIT in .NET is good, but we can help it out a *little bit* more by ... "borrowing" (ahem) the [`IntroSort` code](https://github.com/dotnet/coreclr/blob/775003a4c72f0acc37eab84628fcef541533ba4e/src/mscorlib/src/System/Collections/Generic/ArraySortHelper.cs), and:

- replacing the `CompareTo` usage on the keys with direct  integer operations
- replacing the array access with `unsafe` code that uses  `ulong*` (for the keys) and `int*` (for the index)

(as an aside, it'll be interesting to see how this behaves with `Span<T>`, but that's not complete yet).

I'm not going to show the implementation here, but it is available in the source project. Our code for consuming this doesn't change much, except to call our butchered sort API:

```
Helpers.Sort(sortKeys, index);
```

For me this now takes just over 2.5 seconds.

# Conclusion

So there we go; I've explored some common approaches to improrving sort performance; we've looked at LINQ; we've looked at basic sorts using comparables, comparers and comparisons (which are all different, *obviously*); we've looked at keyed dual-array sorts; we've looked at *indexed* sorts (where the source data remains unsorted); and finally we've hacked the introspective sort to squeeze a tiny bit more from it.

We've seen performance range from 17 seconds for LINQ, 8 seconds for the 3 basic sort APIs, then 4 seconds for our dual array sorts, 3 seconds for the indexed sort, and finally 2.5 seconds with our hacked and debased version.

Not bad for a night's work!

All the code discussed here is [avaiable on github](https://github.com/mgravell/SortOfProblem).