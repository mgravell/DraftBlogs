# Pipelines Part 3 - Readers

In part 1 we introduced the general aims of pipelines and discussed the abstrat `MemoryPool<byte>` (and the default `MemoryPool` implementation of that). In part 2 we looked at how to create a `Pipe`, and how to write data into the pipe via the `IPipeWriter` API. In this part, we're going to look at how to *consume* data from a pipe.

A `Pipe` exposes an `IPipeReader` in virtually the same way that it exposes the `IPipeWriter` that we looked at previously; just like before, we have two identical options:

    IPipeReader reader = pipe.Reader; // option 1
    IPipeReader reader = pipe; // option 2

The API for *reading* a pipe is in many ways quite different to the API for *writing*, although it still builds upon the block-based approach that comes from the memory-pool.

So; let's get hold of a reader and try reading; firstly, note that because data doesn't arrive usually all at once, we almost always want to *loop* while reading:

```
while (true)
{
    ReadResult readResult = await reader.ReadAsync();
    ReadOnlyBuffer buffer = readResult.Buffer;
    // TODO
}
```

This shows us looping over the input (asynchronously as data becomes available), and obtaining the `ReadResult` and `ReadOnlyBuffer` each time. The `ReadResult` describes the state of the read operation (including telling us whether the writer has been completed, i.e. is there ever going to be any more data?); the `ReadOnlyBuffer` allows us to talk about **multiple chunks** (typically multiple blocks from the memory pool) of data at a single time.

A huge difference between a `Pipe` and a `Stream` is that when you `Read` from a `Stream`, you have consumed the data: it is now your responsibility, your problem. This is massively inconvenient in many cases - especially with "framed" data protocols. When reading from a `Stream` you a very rarely lucky enough to end up reading **exactly** on logical "frame" of data. Usually you end up reading half of a frame, or 2 entire frames and half of the next frame. And now you need to do something with the data that isn't yet usable, either because you don't have enough to do anything *at all*, or because you managed to do something but still had some data left over. This is the "back buffer" problem that I mentioned in part 1.

Pipelines solves this by having `ReadAsync` allow you to look at the data *without* considering it "consumed". Instead, you have the opportunity to look at the data, and then *tell the pipe* how much of it you were able to use. That might mean none of it (because you don't have an entire frame), or it might mean all of it, or it might mean some of it. Data we don't consume will be retained by the pipe, and *given to us again next time*. Additionally, you can not only tell it how much you *consumed*, but you can also tell it how much you *looked at*:

- if you *didn't look at* all the data, then when you next `ReadAsync`, it can immediately return control to you, for you to have another look at the data you didn't *actively consume*
- if you *did* look at all the data, it knows that there isn't much point in waking you up again until more data becommes available
- if it can see that you're not looking at all the data but also not consuming anything, then it can interrupt you to prevent you looping forever without advancing
- if it can see that you looked at all the data and the writer is completed (no more data will ever arrive), it can determine that you've got as far as you ever will

We achieve this via the `Advance()` method on the reader; we *must* tell it what we consumed, and we can *optionally* tell it what we looked at.

As a very simple example, we might choose to buffer everything before processing it (note: I'm not suggesting this is a good thinig to do) - essentially `ReadToEnd()`. To do that, we can use:

```
while (true)
{
    ReadResult readResult = await reader.ReadAsync();
    ReadOnlyBuffer buffer = readResult.Buffer;
    if (!readResult.IsCompleted)
    {
        // keep pushing it back; buffer it until the end
        reader.Advance(buffer.Start, buffer.End);
    }
    else
    {
        // TODO
    }
}
```

The `buffer.Start` and `buffer.End` properties are `Position` values - essentially cursors in a sequence of blocks. We can use those `Position` values in our call to `Advance` - the first parameter (`buffer.Start`) tells it "we didn't consume anything past the start"; the second parameter (`buffer.End`) tells it "assume we've looked at it all - don't nudge me again until either more data arrives or the writer is completed".

So; once everything is completed, what can we do? `ReadOnlyBuffer` looks and feels a lot like `Memory<byte>` did. It has a `.Length`, and we can "slice" it to create sub-regions (which as also `ReadOnlyBuffer`). We previously noted that `Memory<byte>` was *indirect* access to memory; well, `ReadOnlyBuffer` is *doubly* indirect - and wraps multiple `Memory<byte>` chunks. A lazy way to process this data would be to linearize it via `ToArray()`:

```
while (true)
{
    ReadResult readResult = await reader.ReadAsync();
    ReadOnlyBuffer buffer = readResult.Buffer;
    if (!readResult.IsCompleted)
    {
        // keep pushing it back; buffer it until the end
        reader.Advance(buffer.Start, buffer.End);
    }
    else
    {
        byte[] arr = buffer.ToArray();
        string message = Encoding.UTF8.GetString(arr);
        Console.WriteLine(message);
        // tell it that we consumed all the data
        reader.Advance(buffer.End);
        break; // we've finished!
    }
}
```

Once again, this *will work*, but represents a failure to properly use the pipelines API: we've forced an additional transient buffer to be created. So what can we do?

Let's assume we want to write a helper extension method (note: our caller will probably be `async`, and we will *probably* want to work with `Span<T>`, so our helper method must be synchronous):

```
public static unsafe string ReadUtf8(
        ref this ReadOnlyBuffer buffer)
{
    if (buffer.IsEmpty) return "";
    // TODO
}
```

allowing our calling code to become:

```
string message = buffer.ReadUtf8();
Console.WriteLine(message);
```

If we're lucky, we'll find that we only *actually* have a single range to consider, which makes a lot of things simpler. For example, we can get a `string` *relatively* easily if we only have a single range, remembering that *depending on what APIs support `Span<T>`*, we might need to use `unsafe` code and get back to an unmanaged pointer:

```
if (buffer.IsSingleSpan)
{
    var span = buffer.First.Span;
    fixed (byte* bytes = &MemoryMarshal.GetReference(span))
    {
        return Encoding.UTF8.GetString(bytes, span.Length);
    }
}
```

Since we're going to want to consume *all* the data in the chunks in one go, we can make use of the fact that `ReadOnlyBuffer` supports `foreach` usage, providing a sequence of `Memory<byte>`, allowing the worst-case multi-buffer version of our method to become:

```
// let's assume that it is small enough to decode
// on the stack; each byte can be *at most* one
// character
int charsSpace = checked((int)buffer.Length);
char* chars = stackalloc char[charsSpace];

var decoder = Encoding.UTF8.GetDecoder();
int charsWritten = 0;

foreach(var chunk in buffer)
{
    var span = chunk.Span;
    fixed (byte* bytes = &MemoryMarshal.GetReference(span))
    {
        int charsAdded = decoder.GetChars(
            bytes, span.Length,
            chars + charsWritten,
            charsSpace, flush: false);

        charsWritten += charsAdded;
        charsSpace -= charsAdded;
    }
}
return new string(chars, 0, charsWritten);
```

More commonly, we want to look at the data in pieces - not all at once, essentially iterating over the various `Span<byte>` that make the multiple chunks in the `ReadOnlyBuffer` consuming only a few bytes at a time. We *could* do this by using `Slice()`, and using the fact that we can `foreach` over a `ReadOnlyBuffer` (which gives us a sequence of `Memory<byte>`), but that's *usually* not the right approach. Instead, we should prefer `BufferReader` (`BufferReader<ReadOnlyBuffer>` now - TBC); this gives us *very efficient* access to the spans inside a `ReadOnlyBuffer` .

A `BufferReader` is a `ref struct` - so stack-only; it *directly* holds a `Span<T>` as a field (something that only a `ref struct` can do), avoiding the "span fetch" cost associted with repeatedly accessing `Memory<T>`. It also avoids constantly re-slicing through the span, and instead provides a `.Index` property that is the next position we should look at in the current `.Span` on the reader. Once we've consumed data, we can call `.Skip(length)` to advance our position in the `BufferReader`, which *might mean it moves to a new span*. We should *always* re-fetch the `.Span` and `.Index` after calling `.Skip()`.

