# Even More Of A Sort Of Problem

So [last time](http://blog.marcgravell.com/2018/01/a-sort-of-problem.html) I talked about a range of ways of performing a sort, ranging from the simple thru to hijacking .NET source code. Very quickly, some folks pointed out that I should have looked at "radix sort", and they're absoltely right - I should have. In fact, in the GPU version of this same code, we do exactly that [via the CUB library](https://nvlabs.github.io/cub/structcub_1_1_device_radix_sort.html).

The great thing is, radix sort is relatively simple, so:

## Attempt 9: radix sort

The key point about radix sort is that it works by grouping the data by groups of bits in the data, using the same "bitwise sortable" idea that we used previously. We've already done the hard work to get a radix compliant representation of our sort data.

We can get a [basic radix sort implementation from wikibooks](https://en.wikibooks.org/wiki/Algorithm_Implementation/Sorting/Radix_sort), but this version has a few things we need to fix:

- this is a single-array version; we want a dual array
- we can use `unsafe` code to get rid of a lot of array range checks (and just: don't be wrong!)
- radix sort needs a workspace the same same as the input values as a scratch area; in the version shown, it allocates this internally, but in "real" code we'll want to manage that externally and pass it in
- we can make `r` (the number of bits to consider at a time) configurable
- the shown code copies the workspace over the real data each cycle, but we can avoid this by simply swapping what we consider "real" and "workspace" each cycle, and copying once at the end if required

I'm not going to try and describe how or why radix sort works (wikipedia covers much of that [here](https://en.wikipedia.org/wiki/Radix_sort), the key thing **that will be relevant in a moment** is: for the group size `r`, it loops through all the data looking `r` bits at a time, to see how many values there are with each possible value for those `r` bits. So if `r=4`, there are 16 possible values over each 4 bits. Once it has that, it iterates a second time, writing the values into the corresponding places for the group it is in.

Once we have an implementation, our code basically consists of preparing the bit-sortable keys just like we did before, then simply invoking the algoritm, passing in our reusable workspace:

```
Helpers.RadixSort(sortKeys, index, keysWorkspace, valuesWorkspace, r);
```

(where `keysWorkspace` and `valuesWorkspace` are scratch areas of the required size, shared between sort cycles).

One consideration here is: what value of `r` (the number of bits to consider at a time) to choose. `4` is a reasonably safe default, but you can experiment with different values for your data to see what works well.

I get:

- r=2: 3800ms
- r=4: 1900ms
- r=8: 1200ms
- r=16: 2113ms

This r=8 is very tempting that is a significant improvement on our previous best.

## Attempt 10: radix sort with parellelization

Remember the "**that will be relevant in a moment**" from a few paragraphs ago? Recall: a key point of radix sort is that for each group of bits (of size `r`), it needs to iterate the entire key-set to count the frequencies of each possible group value. This count operation is something that is embarrasingly parallelizable, since counting chunks can be done independently over the entire data.

To do that, we can create a number of workers, divide the key-space into that many chunks, and tell each worker to perform the counts *for that chunk*. Fire these workers in parallel via `Parallel.Invoke` or similar, and reap the rewards. This creates a slight complexity that we need to *combine* the counts, and there will be thread races. A naive but thread-safe implementation would be to use `Interlocked.Increment` to do all the counts, but that would have severe collision penalties - it is far preferable to count each chunk in complete isolation, and only worry about the combination at the end. At that point, either `lock` or `Interlocked` would be fine, as it is going to happen very minimally. We should also be careful to hoist everything we want into a local, to avoid a lot of `ldarg.0`, `ldfld` overhead:

```
public void Invoke()
{
    var len = Length;
    var mask = Mask;
    var keys = Keys + Offset;
    var shift = Shift;
    int* count = stackalloc int[CountLength];

    // count into a local buffer
    for (int i = 0; i < len; i++)
        count[(*keys++ >> shift) & mask]++;

    // now update the origin data, synchronized
    lock (SyncLock)
    {
        for (int i = 0; i < CountLength; i++)
            Counts[i] += count[i];
    }
}
```

Here we're also using `stackalloc` to do all our counting in the stack space, rather than allocating a count buffer per worker. This is fine, since we'll typically be dealing with values like `r=4` (`CountLength=16`). Even for larger *reasonable* `r`, the stack space is fine. We could very reasonably put an upper bound on `r` of `16` if we wanted to be sure.