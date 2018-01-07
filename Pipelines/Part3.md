# Pipelines Part 3 - Readers

In part 1 we introduced the general aims of pipelines and discussed the abstrat `MemoryPool<byte>` (and the default `MemoryPool` implementation of that). In part 2 we looked at how to create a `Pipe`, and how to write data into the pipe via the `IPipeWriter` API. In this part, we're going to look at how to *consume* data from a pipe.

A `Pipe` exposes an `IPipeReader` in virtually the same way that it exposes the `IPipeWriter` that we looked at previously; just like before, we have two identical options:

    IPipeReader reader = pipe.Reader; // option 1
    IPipeReader reader = pipe; // option 2

The API for *reading* a pipe is in many ways quite different to the API 