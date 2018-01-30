# Sorting myself out, extreme edition

Yes, I'm still talking about sorting. I love going *deep* on problems - it isn't enough to just have *a solution* - I like knowing that I have the *best solution that I am capable of*. This isn't always possible in a business setting - deadlines etc - but this is my own time.

In part 1, we introduced the scenario - and discussed how to build a composite unsigned value that we could use to sort multiple properties as a single key.

In part 2, we looked a litle at radix sort, and found that it was a very compelling candidate.

In this episode, we're going to look at how we can significantly improve the performance of radix sort using a range of tools that are directly applicable to a wide range of common scenarios. In particular, I'm going to look at:

- using knowledge of how signed data works to avoid having to transform between them
- performing operations in blocks rather than per value to reduce calls
- using `Span<T>` as a replacemment for `unsafe` code and unmanaged pointers, allowing you to get very high performance even in 100% managed/safe code
- investigating branch removal as a performance optimization of critical loops
- vectorizing critical loops to do the same work with significantly fewer CPU operations

Hopefully, as a follow up after this one, I'll look at pratical guidance on *parallelizing* this same work to spread the load over available cores.

Key point: the main purpose of these words is **not** to discuss how to implement a radix sort - in fact, we don't even do that. Instead, it uses *one small part* of radix sort as an example problem with which to discuss **much broader** concepts of performance optimization in C# / .NET.

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

[###missing: (baseline) performance data###]

## A negative sign of the times

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

We're usually most familiar with unsigned and 2s-complement representations, because that is what most modern processors use to represent integers. 1s-complement is where `-x === ~x` - i.e. to negate something we simply invert *all the bits*. This works fine but has two zeros, which is one more than we usually need - hence we usually use 2s-complement which simply adds an off-by-one step; this makes zero unambiguous (very useful for `false` as we'll see later) and (perhaps less important) gives us an extra negative value to play with.

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

[###missing: performance data###]

## Blocking ourselves out

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

[###missing: performance data###]

## Safely spanning the performance chasm

You'll notice that in the code above I've made use of `unsafe` code. There are a few things that make `unsafe` appealing, but one of the things it does exceptionally well is allow us to reintrepret chunks of data as other types, which is what this line does:

```
int raw = *(int*)(&source[i]);
```

Actually, there are some methods on `BitConverter` [to do exactly this](https://github.com/dotnet/coreclr/blob/master/src/mscorlib/shared/System/BitConverter.cs#L462-L472), but only the 64-bit (`double`/`long`) versions exist in the ".NET Framework" (".NET Core" has both 32-bit and 64-bit) - and that only helps us with this single example, rather than the general case.

For example, if we are talking in pointers, we can tell the compiler to treat a `float*` as though it were an `int*`. One way we *might* be tempted to rewrite our `ToRadix` method could be to move this coercison earlier:

```
unsafe void ToRadix(float[] source, int[] destination)
{
    const int MSB = 1 << 31;
    fixed(flat* fPtr = source)
    {
        int* ptr = (int*)fPtr;
        for(int i = 0 ; i < values.Length; i++)
        {
            int raw = *ptr++;
            destination[i] = (raw & MSB) == 0 ? raw : ~raw | MSB;
        }
    }
}
```

Now we're naturally reading values out *as* `int`, rather than performing any reinterpretation per value. This is *useful*, but it requires us to use `unsafe` code (always a great way to get hurt), and it doesn't work with generics - you **cannot** use `T*` for some `<T>`, even with the `where T : struct` constraint.

I've spoken more than a few times about `Span<T>`; quite simply: it rocks. To recap, `Span<T>` (and it's heap-friendly cousin, `Memory<T>`) is a general purpose, efficient, and versatile representation of contiguous memory - which includes things like arrays (`float[]`), but more exotic things too.

One of the most powerful (but simple) features of `Span<T>` is that it allows us to do type coercison *in fully safe managed code*. For example, instead of a `float[]`, let's say that we have a `Span<float>`. We can reinterpet that as `int` *very simply*:

    Span<float> values = ...
    var asIntegers = values.NonPortableCast<float, int>();

Note: this API is likely to change - by the time it hits general availablity, it'll probably be:

    var asIntegers = values.Cast<int>();

What this does is:

- look at the sizes of the original and target type
- calculate how many of the target type fit into the original data
- round down (so we never go out of range)
- hand us back a span of the target type, of that (possibly reduced) length

Since `float` and `int` are the same size, we'll find that `asIntegers` has the same length as `values`.

What is *especially* powerful here is that this trick *works with generics*. It does something that `unsafe` code **will not do for us**. Note that a *lot of love* has been shown to `Span<T>` in the runtime and JIT - essentially all of the same tricks that make `T[]` array performance largely indistinguishable from pointer `T*` performance.

This means we could simplify a lot of things by converting from our generic `T` *even earlier* (so we only do it once for the entire scope of our radix code), and having our radix converter just talk in terms of the raw bits (usually: `uint`).

```
// our original data
float[] values = ...
// recall that radix sort needs some extra space to work in
float[] workspace = ... 
// implicit <float> and implicit conversion to Span<float>
RadixSort32(values, workspace); 

...
RadixSort32<TSource>(Span<T> typedSource, Span<T> typedWorkspace)
    where T : struct
{
    // note that the JIT can inline this *completely* and remove impossible
    //  code paths, because for struct T, the JIT is per-T
    if (Unsafe.SizeOf<T>() != Unsafe.SizeOf<uint>()) .. throw an error

    var source = typedSource.NonPortableCast<T, uint>();
    var workspace = typedWorkspace.NonPortableCast<T, uint>();

    // convert the radix if needed (into the workspace)
    var converter = GetConverter<T>();
    converter?.ToRadix(source, workspace);

    // ... more radix sort details not shown
}
...
// our float converter
public void ToRadix(Span<uint> values, Span<uint> destination)
{
    const uint MSB = 1U << 31;
    for(int i = 0 ; i < values.Length; i++)
    {
        uint raw = values[i];
        destination[i] = (raw & MSB) == 0 ? raw : ~raw | MSB;
    }
}
```

The code is getting simpler, while retaining performance *and* becoming more generic-friendly; and we haven't needed to use a single `unsafe`. You'll have to excuse me an `Unsafe.SizeOf<T>()` - despite the name, this isn't *really* an "unsafe" operation in the usual sense - this is simply a wrapper to the `sizeof` IL instruction that is *perfectly well defined* for all `T` that are usable in generics. It just isn't directly available in safe C#.

[###missing: performance data###]

## Taking up tree surgery: prune those branches

Something that gnaws at my soul in where we've got to is that it includes a branch - an `if` test, essentially - in the inner part of the loop. Actually, there's two and they're both hidden. The first is in the `for` loop, but the one I'm talking about here is the one hidden in the ternary conditional operation, `a ? b : c`. CPUs are very clever about branching, with branch prediction and other fancy things - but it can still stall the instruction pipeline, especially if the prediction is wrong. If only there was a way to rewrite that operation to not need a branch. I'm sure you can see where this is going.

Branching: bad. Bit operations: good. A common trick we can use to remove branches is to obtain a bit-mask that is either all 0s (000...000) or all 1s (111...111) - so: 0 and -1 in 2s-complement terms. There are various ways we can do that (although it also depends on the actual value of `true` in your target system, which is a surprisingly complex question). Obviously one way to do that would be:

    // -1 if negative, 0 otherwise
    var mask = (raw & MSB) == 0 ? 0 : ~0;

but that just *adds* another branch. If we were using C, we could use the knowledge that the equality test returns an *integer* of either 0 or 1, and just negate that to get 0 or -1:

    // -1 if negative, 0 otherwise
    var mask = -((raw & MSB) == 0);

But no such trick is available in C#. What we *can* do, though, is use knowledge of *arithmetic shift*. Left-shift (`<<`) is simple; we shift our bits `n` places to the left, filling in with 0s on the right. So binary `11001101 << 3` becomes `01101000` (we lose the `110` from the left).

 Right-shift is more subtle, as there are two of them: logical and arithmetic, which are essentially unsigned and signed. The *logical* shift (used with `uint` etc) moves our bits  `n` places to the right, filling in with 0s on the left, so `11001101 >> 3` gives `00011001` (we lose the `101` from the right). The *arithmetic* shift behaves differently depending on whether the most significant bit (which tells us the sign of the value) is `0` or `1`. If it is `0`, it behaves exactly like the logical shift; however, if it is `1`, it fills in with 1s on the left; so `11001101 >> 3` gives `11111001`. Using this, we can use `>> 31` (or `>> 63` for 64-bit data) to create a mask that matches the sign of the original data:

    // -1 if negative, 0 otherwise
    var mask = (uint)(((int)raw) >> 31);

Don't worry about the extra conversions: as long as we're in an `unchecked` context (which we are by default), they simply don't exist. All they do is tell the compiler which shift operation to emit. If you're curious, you can [see this here](https://sharplab.io/#v2:EYLgtghgzgLgpgJwDQxASwDYB8ACAmAAgGUALNAMxgFgAoAb1oKYJwEYA2AgVzQDsYCAcTgwAstADWACh78CANwgYucAJSNmDGsx04A7ARl8YqqWeOrFytQQB8tggGZWqgNwamAX1qegA===), but in IL terms, this is just:

    (push the value onto the stack)
    ldc.i4.s 31 // push 31 onto the stack
    shr         // arithmetic right shift

In IL terms, there's really no difference between signed and unsigned integers, *other* than whether the compiler emites the signed or unsigned opcodes for operations. In this case we've told it to emit [`shr`](https://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.shr(v=vs.110).aspx) - the signed/arithmetic opcode, instead of [`shr.un`](https://msdn.microsoft.com/en-us/library/system.reflection.emit.opcodes.shr_un(v=vs.110).aspx) - the unsigned/logical opcode.

OK; so we've got a mask that is either all zeros or all ones. Now we need to use it to avoid a branch, but how? Consider:

```
var condition = // all zeros or all ones
var result = (condition & trueValue) | (~condition & falseValue);
```

If `condition` is all zeros, then the `condition & trueValue` gives us zero; the `~condition` becomes all ones, and therefore `~condition & falseValue` gives us `falseValue`. When we "or" (`|`) those together, we get `falseValue`.

Likewise, if `condition` is all ones, then `condition & trueValue` gives us `trueValue`; the `~condition` becomes all zeros, and therefore `~condition & falseValue` gives us zero. When we "or" (`|`) those together, we get `trueValue`.

So our branchless operation becomes:

```
public void ToRadix(Span<uint> values, Span<uint> destination)
{
    const uint MSB = 1U << 31;
    for(int i = 0 ; i < values.Length; i++)
    {
        uint raw = values[i];
        var ifNeg = (uint)(((int)raw) >> 31);
        destination[i] =
            (ifNeg & (~raw | MSB)) // true
            | (~ifNeg & raw);      // false
    }
}
```

This might look more complicated, but it is **very** CPU-friendly: it pipelines very well, and doesn't involve any branches for it to worry about. Doing a few extra bit operations is *nothing* to a CPU - especially if they can be pipelined. Long instruction pipelines are actually a *good* thing to a CPU - compared to a branch or something that might involve a cache miss, at least.

[###missing: performance data###]

## Why do one thing at a time?

OK, so we've got a nice branchless version, and the world looks great. We've made significant improvements. But we can still get *much better*. At the moment we're processing each value one at a time, but as it happens, this is a perfect scenario for *vectorization* via [SIMD ("Single instruction, multiple data")](https://en.wikipedia.org/wiki/SIMD).

We have a pipelineable operation without any branches; if we can execute it for *one* value, we can probably execute it for *multiple* values **at the same time** - magic, right? Many modern CPUs include support for performing basic operations like the above on multiple values at a time, using super-wide registers. Right now we're using 32-bit values, but most current CPUs will have support for AVX (mostly: 128-bit) or AVX2 (mostly: 256-bit) operations. If you're on a very expensive server, you might have more (AVX512). But let's assume AVX2: that means we can handle 8 32-bit values at a time. That means 1/8th of the main operations, and also 1/8th of the `if` branches hidden in the `for` loop.

Some languages have automatic vectorization during compilation; C# doesn't have that, and neither (AFAIK) for the JIT. But, we still have access to a range of vectorized operations (with support for the more exotic intrinsics being added soon). Until recently, one of the most awkward things about working with vectorization has been *loading the values*. This might sound silly, but it is surprisingly difficult to pull the values in efficiently when you don't know how wide the vector registers are on the target CPU. Fortunately, our amazing new friend `Span<T>` jumps to the rescue here - making it almost embarrassingly easy!

First, let's look at what the shell loop might look like, without actually doing the real work in the middle:

```
public void ToRadix(Span<uint> values, Span<uint> destination)
{
    const uint MSB = 1U << 31;

    int i = 0;
    if (Vector.IsHardwareAccelerated)
    {                               
        var vSource = source.NonPortableCast<uint, Vector<uint>>();
        var vDest = destination.NonPortableCast<uint, Vector<uint>>();
        
        for (int j = 0; j < vSource.Length; j++)
        {
            var vec = vSource[j];
            vDest[j] = // TODO
        }
        // change our root offset for the remainder of the values
        i = vSource.Length * Vector<uint>.Count;
    }
    for( ; i < values.Length; i++)
    {
        uint raw = values[i];
        var ifNeg = (uint)(((int)raw) >> 31);
        destination[i] =
            (ifNeg & (~raw | MSB)) // true
            | (~ifNeg & raw);      // false
    }
}
```

First, look at the bottom of the code; here we see that our *regular branchless* code still persists. This is for two reasons:

- the target CPU *might not be capable of vectorization*
- our input data might not be a nice multiple of the register-width, so we might need to process a final few items the old way

Note that we've changed the `for` loop so that it doesn't reset the position of `i` - we don't *necessarily* start at `0`.

Now look at the `if (Vector.IsHardwareAccelerated)`; this checks that suitable vectorization support is available. Note that the JIT can optimize this check away completely (and remove all of the inner code if it won't be reached). If we *do* have support, we cast the span from a `Span<uint>` to a `Span<Vector<uint>>`. Note that `Vector<T>` is recognized by the JIT, and will be *reshaped* by the JIT to match the size of the available vectorization support on the running computer. That means that when using `Vector<T>` we don't need to worry about whether the target computer has SSE vs AVX vs AVX2 etc - or what the available widths are; simply: "give me what you can", and the JIT worries about the details.

We can now loop over the *vectors* available in the cast spans - loading an entire vector at a time simply using our familiar: `var vec = vSource[j];`. This is a *huge* difference to what loading vectors used to look like. We then do some operation (not shown) on `vec`, and assign the result *again as an entire vector* to `vDest[j]`. On my machine with AVX2 support, `vec` is block of 8 32-bit values.

Next, we need to think about that `// TODO` - what are we actually going to *do* here? If you've already re-written your inner logic to be branchless, there's actually a very good chance that it will be a like-for-like translation of your branchless code. In fact, it turns out that the ternary conditional scenario we're looking at here is *so common* that there are vectorized operations *precisely to do it*; the "conditional select" vectorized CPU instruction can essentially be stated as:

    // result conditionalSelect(condition, left, right)
    result = (condition & left) | (~condition & right);

Where `condition` is *usually* either all-zeros or all-ones (but it doesn't have to be; if you want to pull different bits from each value, you can do that too).

This intrinsic is exposed directly on `Vector`, so our missing code becomes simply:

    var vMSB = new Vector<uint>(MSB);
    var vNOMSB = ~MSB;
    for (int j = 0; j < vSource.Length; j++)
    {
        var vec = vSource[j];
        vDest[j] = Vector.ConditionalSelect(
            condition: Vector.GreaterThan(vec, vNOMSB),
            left: ~vec | vMSB, // when true
            right: vec // when false
        );
    }

Note that I've pre-loaded a vector with the MSB value (which creates a vector with that value in every cell), and I've switched to using a `>` test instead of a bit test and shift. Partly, this is because the vectorized eqaulity / inequality operations *expect* this kind of usage, and very kindly return `-1` as their true value - using the result to directly feed "conditional select".

[###missing: performance data###]

A reasonable range of common operations are available on [`Vector`](https://msdn.microsoft.com/en-us/library/system.numerics.vector(v=vs.111).aspx) and [`Vector<T>`](https://msdn.microsoft.com/en-us/library/dn858385(v=vs.111).aspx). If you need exotic operations like "gather", you might need to wait until [`System.Runtime.Intrinsics`](https://github.com/dotnet/corefx/blob/master/src/System.Runtime.Intrinsics/ref/System.Runtime.Intrinsics.cs) lands. One key difference here is that `Vector<T>` exposes the *common intersection* of operations that might be available (with different widths) against *different* CPU instruction sets, where as `System.Runtime.Intrinsics` aims to expose the *underlying* intrinsics - giving access to the full range of instructions, but forcing you to code specifically to a chosen instruction set (or possibly having two implementations - one for AVX and one for AVX2). This is simply because there isn't a uniform API surface between generatons and vendors - it isn't simply that you get the same operations with different widths: you get different operations too. So you'd typically be checking `Aes.IsSupported`, `Avx2.IsSupported`, etc. Being realistic: `Vector<T>` is what we have *today*, and it worked damned well.

## Summary

We've looked at a range of advanced techniques for improving performance of critical loops of C# code, including (to repeat the list from the start):

- using knowledge of how signed data works to avoid having to transform between them
- performing operations in blocks rather than per value to reduce calls
- using `Span<T>` as a replacemment for `unsafe` code and unmanaged pointers, allowing you to get very high performance even in 100% managed/safe code
- investigating branch removal as a performance optimization of critical loops
- vectorizing critical loops to do the same work with significantly fewer CPU operations

And we've seen *dramatic* improvements to the performance. Hopefully, some or all of these techniques will be applicable to your own code. Either way, I hope it has been an interesting diversion.

Next time: practical parallelization