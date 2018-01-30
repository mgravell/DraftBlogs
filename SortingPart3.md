Yes, I'm still talking about sorting. I love going *deep* on problems - it isn't enough to just have *a solution* - I like knowing that I have the *best solution that I am capable of*. This isn't always possible in a business setting - deadlines etc - but this is my own time.

In part 1, we introduced the scenario - and discussed how to build a composite unsigned value that we could use to sort multiple properties as a single key.

In part 2, we looked a litle at radix sort, and found that it was a very compelling candidate.

In this episode, we're going to look at how we can significantly improve the performance of radix sort using a range of tools that are directly applicable to a wide range of common scenarios. In particular, I'm going to look at:

- using knowledge of how signed data works to avoid having to transform between them
- performing operations in blocks rather than per value to reduce calls
- using `Span<T>` as a replacemment for `unsafe` code and unmanaged pointers, allowing you to get very high performance even in 100% managed/safe code
- investigating branch removal as a performance optimization of critical loops
- vectorizing critical loops to do the same work with significantly fewer CPU operations
- pratical guidance on parallelizing work to spread the load over available cores

Obviously I can't cover the entire of radix sort for these, so I'm going to focus on one simple part: composing the radix for sorting. To recall, a naive implementation of radix sort requires unsigned keys, so that the data is naturally sortable in their binary representation. Signed integers and floating point numbers don't follow this layout, so in part 1 we introduced some basic tools to change between them:

```
uint Sortable(int value)
{
    // re-base eveything upwards, so anything
    // that was the min-value is now 0, etc
    var val = unchecked((uint)(value - int.MinValue));
    return val;
}
unsafe uint Sortable (float value)
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

These two simple transformation - applied to our target values - will form the central theme of this entry.

# A negative sign of the times

What's faster than performing a fast operation? *Not* performing a fast operation. The way radix sort works is by looking at the sort values `r` bits at a time (commonly 4, 8, 10, but any number is valid) and for that block of bits: counting how many candidates are in each of the `1 << r` possible buckets. So if `r` is 3, we have 8 possible buckets. From that it computes target offsets for each group: if there are 27 with in bucket 0, 12 in bucket 1, 3 in bucket 2, etc - then *when sorted* buckete 0 will start at offset 0, bucket 1 at offset 27, bucket 2 at offset 39, bucket 3 at offset 41 etc - just by accumulating the counts. But this breaks if we have signed numbers.

Why?

First, let's remind ourselves of the various ways that signed and unsigned data can be represented in binary, using a 4 bit number system and integer representations:


Binary  | Unsigned | 2s-comlement | 1s-complement | Sign bit
:---: | :---: | :---: | :---: | :---:
0000 | 0 | 0 | +0 | +0
0001 | 1 | 1 | 1 | 1
0010 | 2 | 2 | 2 | 2
0011 | 3 | 3 | 3 | 3
0100 | 4 | 4 | 4 | 4
0101 | 5 | 5 | 5 | 5
0110 | 6 | 6 | 6 | 6
0111 | 7 | 7 | 7 | 7
1000 | 8 | -8 | -7 | -0
1001 | 9 | -7 | -6 | -1
1010 | 10 | -6 | -5 | -2
1011 | 11 | -5 | -4 | -3
1100 | 12 | -4 | -3 | -4
1101 | 13 | -3 | -2 | -5
1110 | 14 | -2 | -1 | -6
1111 | 15 | -1 | -0 | -7

We're usually most familiar with unsigned and 2s-complement representations, because that is what most modern processors use to represent integers. 1s-complement is where `-x === ~x` - so negate something we simply invert *all the bits*. This works fine but has two zeros, which is one more than we usually need - hence we usually use 2s-complement which simply adds an off-by-one step; this makes zero unambiguous (very useful for `false` as we'll see later) and (perhaps less important) gives us an extra negative value to play with.

The final option is to use a sign bit; to negate a number we flip the most significant bit, so `-x === x ^ 0b1000`. IEEE754 floating point numbers (`float` and `double`) are implemented using a sign bit, which is why floating point numbers have +0 and -0. Due to clever construction, the rest of the value can be treated as naturally/bitwise sortable - even without needing to understand about the "mantissa", "exponent" and "bias". This means that to convert a **negative** `float` (or any other sign-bit number) to a 1s complement representation, we simply flip all the bits except the most significant bit.

---

So: armed with this knowledge, we can see that signed data in 1s-complement or 2s-complement is *almost* "naturally sortable" in binary, but simply: the negative values are sorted in increasing numerical value,but come *after* the positive values (we can happily assert that -0 < +0). This means that we can educate radix sort about 1s-complement and 2s-complement signed data *simply by being clever when processing the **final left-most chunk***: based on `r` and the bit-width, calculate which bit is the most-signficant bit (which indicates sign), and simply *process the negative buckets first* (still in the same order) when calculating the offsets; then calculate the offsets of the non-negative buckets. If we were using the 4-bit system above and `r=4`, we would have 16 buckets, and would calculate offsets in the order (of unsigned buckets) 8-15 then 0-7.

By doing this, we can **completely remove** any need to do any pre-processing when dealing with values like `int`. We could perhaps wish that the IEEE754 committee had preferred 1s-complement so we could skip all of this for `float` too, but a: I think it is fair to assume that there's good technical reasons for the choice (presumably relating to fast negation, and fast access to the mantissa/exponent), and b: it is moot: IEEE754 is implemented in CPU architectures and is here to stay.

We're still left with an issue for `float`, though: we can't use the same trick for values with a sign bit, because the sign bit changes the order *throughout* the data - making grouping impossible. We can make our lives *easier* though: since the algorithm can now cope with 1s-complement and 2s-complement data, we can switch to 1s-complement rather than to fully unsigned, which as discussed above: is pretty easy:

```
unsafe int ToRadix (float value)
{
    const int MSB = 1 << 31;
    int raw = *(int*)(&value);
    // if sign bit set: flip all bits except the MSB
    return (raw & MSB) == 0 ? raw : ~raw | MSB;
}
```

A nice side-effect of this is that it is self-reversing: we can apply the exact same bit operations to convert from 1s-complement back to a sign bit.

# Block operations

We have a large chunk of data, and we want to perform a transformation on each value. So far, we've looked at a per-value transformation function (`Sortable`), but that means the overhead of a call per-value (which may or may not get inlined, depending on the complexity, and how we resolve the method - i.e. does it involve a `virtual` call to a type that isn't reliably known), and makes it hard for us to apply nuclear options to optimize.

So; let's say we have our existing loop:

```
float[] values = ...
int[] radix = ...
for(int i = 0 ; i < values.Length; i++)
{
    radix = someHelper.Sortable(values[i]);
}
```

and we want to retain the ability to swap in per-type implementations of `someHelper.Sortable`; we can significantly reduce the call overhead by performing a block-based transformation. Consider:

```
float[] values = ...
int[] radix = ...
someHelper.ToRadix(values, radix);
...
unsafe void ToRadix(float[] source, int[] destination)
{
    const int MSB = 1 << 31;
    for(int i = 0 ; i < values.Length; i++)
    {
        int raw = *(int*)(&source[i]);
        // if sign bit set: flip all bits except the MSB
        destination[i] = (raw & MSB) == 0 ? raw : ~raw | MSB;
    }
}
```

How much of a speed improvement this makes depends a lot on whether the JIT managed to inline the IL from the original version. It is usually a good win by itself, but more importantly: it is a key stepping stone to further optimizations.

