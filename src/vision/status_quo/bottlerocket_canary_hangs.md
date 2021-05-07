# 😱 Status quo stories: Bottlerocket Canary Hangs

## 🚧 Warning: Draft status 🚧

This is a draft "status quo" story submitted as part of the brainstorming period. It is derived from real-life experiences of actual Rust users and is meant to reflect some of the challenges that Async Rust programmers face today.

If you would like to expand on this story, or adjust the answers to the FAQ, feel free to open a PR making edits (but keep in mind that, as they reflect peoples' experiences, status quo stories [cannot be wrong], only inaccurate). Alternatively, you may wish to [add your own status quo story][htvsq]!

## The story

[Bottlerocket] is a Linux-based open-source operating system that is purpose-built by Amazon Web Services for running containers.
We use our own program [pubsys] to ensure that Bottlerocket update repositories are healthy.

### The TL;DR

Although the first two of these points are about tokio, they are really about Rust async since tokio serves as the de-facto `std::runtime` for Rust.

- It is confusing and dangerous that multiple tokio runtimes can panic or hang at program runtime.
- It is challenging that using multiple major versions of tokio (which is allowed by cargo) can fail at runtime.
- It is unfortunate that we need a 3rd party runtime in order to `block_on` a future, even if we are not trying to write async code.

### Multiple Tokio Runtimes

We use our own [tough] library to read and write TUF repositories.
This library was created before async became widespread and [reqwest] changed its main interface to `async`.
When reqwest switched to async, we used the `reqwest::blocking` feature instead of re-writing tough to be an async interface.
(Maybe we [should](https://github.com/awslabs/tough/issues/213) make tough an async interface, but we haven't yet.)
In order to provide a non-async runtime, reqwest creates a tokio runtime so that it can await futures.

In pubsys we created some parallel downloading logic while using the above libraries.
Without realizing the danger, we created a tokio runtime in pubsys and used futures/await to do this parallelization.
Surprisingly, in retrospect, this worked... until it didn't.

Recently we we realized that our repository verification alarm was hanging.
The root cause was having multiple tokio runtimes, though we don't know what change exposed the issue.
The fix was to eliminate the need for a tokio runtime in the pubsys code path by doing the parallel downloads in a different way
(first with [threads] for a quick fix, then with a [thread pool]).

### Multiple Tokio Major Versions

Updating to tokio v1 was a challenge for Bottlerocket because:
- Having two major versions of the tokio runtime can/will cause problems.
- Cargo does not understand this and allows multiple major versions of tokio.

Ultimately our strategy for this in Bottlerocket was to ensure that only one version of tokio exists in our Cargo.lock.
This requirement delayed our ability to upgrade to tokio v1 and caused us to use a beta version of actix-web.

### Not Easy to Block-On

If we are writing a procedural program, and it is perfectly fine to block, then encountering an async function is problematic.

```rust
fn my_blocking_program() {
    blocking_function_1();
    blocking_function_2();

    // uh oh, now what?
    async_function_1().await
}
```

Uh oh.
Now I need to decide what third-party runtime to use.
Should I create that runtime around main, or should I create it and clean it up around this one function call?
Put differently, should I bubble up async throughout my program even though my program is blocking by nature?

If I use tokio, and get it wrong (foot-guns described above), my program may hang or panic at runtime.

A nicer experience would be:

```rust
fn my_blocking_program() {
    blocking_function_1();
    blocking_function_2();

    std::thread::block_on({
        async_function_1()
    })
}
```

<!-- links -->

[Bottlerocket]: https://github.com/bottlerocket-os/bottlerocket
[pubsys]: https://github.com/bottlerocket-os/bottlerocket/tree/develop/tools/pubsys
[tough]: https://github.com/awslabs/tough/
[reqwest]: https://github.com/seanmonstar/reqwest
[threads]: https://github.com/bottlerocket-os/bottlerocket/pull/1521/files#diff-7546c95d0732614af12f62ff8c072f8c1061f82945c714daf1dd2962c42921ffL47
[thread pool]: https://github.com/bottlerocket-os/bottlerocket/pull/1564/files

## 🤔 Frequently Asked Questions

*Here are some standard FAQ to get you started. Feel free to add more!*

### **What are the morals of the story?**

When you use a Rust async runtime, which is unavoidable these days, you *really* need to know what you're doing.

### **What are the sources for this story?**

See the links embedded in the story itself (mostly at the top).

### **Why did you choose *Bottlerocket* to tell this story?**

Bottlerocket is a real-life project that experienced these real-life challenges!
TODO - pick a character and re-write to fit the mold?

### **How would this story have played out differently for the other characters?**
TODO - figure this out if it is required.

[character]: ../characters.md
[status quo stories]: ./status_quo.md
[Alan]: ../characters/alan.md
[Grace]: ../characters/grace.md
[Niklaus]: ../characters/niklaus.md
[Barbara]: ../characters/barbara.md
[htvsq]: ../how_to_vision/status_quo.md
[cannot be wrong]: ../how_to_vision/comment.md#comment-to-understand-or-improve-not-to-negate-or-dissuade
