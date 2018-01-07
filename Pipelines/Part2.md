# Pipelines Part 2 - Pipes

In part 1 we had a look at what problems attempt to solve and introduced the "memory pool". So now let's see what we can do with it by introducing the "pipe" and comparing it to the familiar `Stream`. Before we can do that we need to define some configuration options (`PipeOptions`) - the most important of which is : the `MemoryPool`:

```
// ensure this gets disposed later
var pool = new MemoryPool();

// we can reuse this between multiple pipes
var options = new PipeOptions(pool);

// create our pipe
var pipe = new Pipe(options);
```

This is somewhat surprising when compared to `Stream`, because `Stream` is an abstract base-type, and you need to instantiate a *specific type* of stream. In pipelines, the `Pipe` is concrete and all the magic happens *at the ends*:

![the pipe with inputs and outputs](pipe.png)

As you can see, a pipe has a very distinct direction, with separate APIs for pushing data into a pipe (`IPipeWriter`) and reading data *from* a pipe (`IPipeReader`); note that the pipe *is itself* the `Input` and the `Output`, so the following two options are identical:

    async Task DoSomethingAsync(IPipeWriter reader) {...}
    //...
    await DoSomethingAsync(pipe.Output); // option 1
    await DoSomethingAsync(pipe); // option 2

 This is quite different from a `Stream` that has very ambiguous semantics for input and output - some stream implementations only support read *or* write; some stream implementations support read *and* write but talking to the same underlying data (`FileStream`, `MemoryStream`) - and some stream implementations support read *and* write, but where read and write refer to completely separate data (`NetworkStream`). This means that if you are dealing with a bidirectional data source, *you simply need two pipes* - one for the input, and one (facing the other way) for the output.

In pipelines, the role of the *pipe* is to negotiate with the *things producing or consuming data*, making use of the memory pool. Essentially, it handles all the buffer and semantics and deals with activating the two ends when needed.

## Writing to a pipe

The first thing we probably want to try is writing data to a pipe. Now, a lazy but bad way of doing this might be the `WriteAsync` extension method that takes a `byte[]`:

    // not great; don't do this
    await pipe.WriteAsync(
        Encoding.ASCII.GetBytes("Hello, world!"));

you might reasonably wonder why this is "bad". Compare back to the list of objectives behind pipelines from part 1, and we've violated two of them:

- we've allocated a trainsient `byte[]` that holds our data
- we've forced it to copy the data from transient buffer to the *actual* buffer that we want to use (the one from the memory pool)

You'd think that writing an arbitrary `string` to an output was a simple enough thing, but it actually requires a surprising number of concepts. So: rather than dive in at the deep end, let's go back a few levels and just write a few bytes (maybe 0x10, 0x20, 0x30 - why not?) to the pipe without a helper method:

First, we ask the writer for some space to write into; in this case, we only want 3 bytes, so we can ask for that:

    WritableBuffer wb = writer.Alloc(3);

A `WritableBuffer` represents the state of an in-progress attempt to write, and essentially provides us with access to part of a *block* obtained from the memory pool (that we looked at in part 1). Usually, we will be writing lots of things at the same time, so by specifying the number of bytes we want to write, the pipe can check whether there is enough space for the 3 bytes at the end of the *current* block, and if so it can allow us to keep appending there; if there were only 2 bytes left (or no active block at all), it can consider the previous block as "done", request a new block from the memory pool, and make the fresh block available to us. An alternative approach is to just use `writer.Alloc()` *without* specifying an amount to ensure; in that case, it is the consumer's job to check how much space is available in the `WritableBuffer`, and only write that much - then request a new one. This is especially important when handling larger quantities of data, an *in particular* when the amount of data that we want to write might be larger than the block size of the memory pool, since `writer.Alloc(hugeSize)` is likely to fail.

Once we have the `WritableBuffer`, we need to write to it. It provides a property that provides the *unused space in the block*:

    Memory<byte> memory = wb.Buffer;

`Memory<T>` is a very important type for us to understand. It is essentially an value-type (a `readonly struct`) that provides *indirect* access to an arbitrary contiguous block of memory. This is *often* a slice of an array (as it is with the default `MemoryPool` that pipelines provides), but it could also relate to memory accessed via a pointer, or some sub-range of a `string`. A key point here is that unlike an array (`T[]`) or `string` - it doesn't necessarily start *at the start* of the array/string - it is more comparable to the `ArraySegment<T>` that has existed (largely unused) in .NET for a long time. Additionally, you can "slice" within a memory to look at smaller pieces:

    // create a "memory" that refers to the 6 bytes
    // offset 2 from the start of the current memory
    // (essentially "skip 2, take 6")
    Memory<byte> chunk = memory.Slice(2, 6);

As you might expect, it provides a `.Length`, but it *doesn't* have an indexer - meaning: the following doesn't work:

    memory[0] = 0x10; // does not work

The reason that this doesn't work is the word "indirect" in the previous paragraph; what we *actually* want to get hold of is the "span" (`Span<T>`), which is the heavily optimized *actual* access to a chunk of memory. But because of the "indirect", we should consider fetching the span to be *not free*, so we only want to do that once:

```
Span<byte> span = memory.Span;
span[0] = 0x10;
span[1] = 0x20;
span[2] = 0x30;
```

Now; `Span<T>` is a *very* interesting type. As you can see, it behave a lot like an array - including having a `.Length` etc. This type is a *very direct* access to memory, and is highly optimized by the JIT. Because of the semantics of `Span<T>` (and the types of memory it can refer to, which includes `stackalloc` memory and other stack-only data), it is a `ref struct`, which means: it is **only ever** allowed to exist on the stack. This in turn means that we *cannot* have a `Span<T>` field on regular `class` or `struct`, but it also means that we can't have a `Span<T>` local variable in an `async` method, since `async` continuations work by building a state machine that *might* need to end up in an object (on the heap).

This requirement to not have `Span<byte>` in an `async` method is very important in pipelines, since (from part 1 of this blog), `async` is a key goal of pipelines; we'll look at how we avoid this being a problem shortly.

If all we want to write is those 3 bytes, we can finish writing:

```
wb.Advance(3);
wb.Commit();
```

The `Advance(3)` tells the buffer that we have *actually* written 3 bytes - if we access `wb.Buffer` again, we'll find that the `Memory<byte>` we get back is now 3 shorter (and it  now starts 3 bytes further into the block).

The `Commit()` tells the pipe that we're finished with that `WritableBuffer`.

---

Marc suggests: this feels backwards; sorry, but it does.

Alternative API suggestion and thoughts:

- a writable buffer only ever accesses a single span
- it will be used in sync code, since the caller needs a span
- there's lots of layers you need to step through currently: `.Buffer.Span`
- the `Advance()` feels heavy - it needs another `.Buffer.Span` to be useful
- `Advance` goes back to the pipe, but... without good reason

Suggestion:

- the current API feels heavy and awkward, with unnecessary steps that actively harm performance (lots of slicing and `.Memory.Span` usage)
- in most real uses, you will be using `Alloc` repeatedly to write additively to the same block; this might be for multiple independent values via extension methods on `IOutput`, or it might be a single large value such as encoding a large string, with it looping over multiple `Alloc`s
- remove `Advance()` completely; there's no benefit to it - any *useful* usage of it requires you to keep slicing a span; let the *consumer* keep track of the offset and just hold a `Span<byte>` once
- move to `Commit(3)` which acts like `Advance(3)` + `Commit()` does currently
- remove `WritableBuffer` *completely*
  - have `Alloc()` *simply return the span*
  - have `Commit(3)` *on the `IOutput` / `IPipeWriter`
- alternative to that:
  - make `WritableBuffer` a `ref struct` and have it hold a `Span<byte>`, not a `Memory<byte>`
  - have `Commit(3)` on the `WritableBuffer`
- only when `Commit()` is called does the pipe have any involvement; I know the point of this currently is to move the write head, but that's simply an implementation detail - it isn't actually *necessary* to do so until we actually have something useful we can do with it

example usage:

```
var span = writer.Alloc(3);
span[0] = 0x10;
span[1] = 0x20;
span[2] = 0x30;
writer.Commit(3);
```

Since the exchange type is intended to be `IOutput`, this makes perfect sense to me, and is *so much cleaner*.

The second suggested alternative would be:

```
var wb = writer.Alloc(3);
var span = wb.Span;
span[0] = 0x10;
span[1] = 0x20;
span[2] = 0x30;
wb.Commit(3);
```

(end of Marc suggests)

---