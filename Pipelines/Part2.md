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











Instead, a more "pipelines" way to do this would be to:

- ask the writer for a buffer from the memory pool
- ask the `Encoding` to write *to that buffer*
- tell the pipe that we're done, and to flush the data

(helper methods to do this exist, note)

So what would this look like? Well, getting a buffer is easy enough:

    WritableBuffer wb = writer.Alloc();

This gives us a scratch area - essentially part of a *block* from the memory pool - that we can write to. Subsequent calls to `Alloc` *without flushing* might actually return the same *block*, but with a different offset. We can optionally tell `Alloc` a minimum number of bytes we need: if our data should fit on the end of the current block, then we'll get that - otherwise, it'll ask the memory pool for a new block. Note that if we ask for something larger than the memory pool's block size then an exception can occur, so the *best* approaches are:

- if your data is fairly small, then ask for what you need
- if your data could be large, then just ask for "some" space, and be prepared to handle the different blocks yourself

The `WritableBuffer` has a `Buffer` property that is a `Memory<byte>`:

    Memory<byte> memory = wb.Buffer;