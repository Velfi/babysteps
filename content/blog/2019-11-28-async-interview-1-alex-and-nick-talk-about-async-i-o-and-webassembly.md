+++
layout = "post"
title = "Async Interview #1: Alex and Nick talk about async I/O and WebAssembly"
categories = ["Rust", "AsyncInterviews"]
+++

Hello from Iceland! (I'm on vacation.) I've just uploaded [the first
of the Async Interviews][video] to YouTube. It is a conversation with Alex
Crichton ([alexcrichton]) and Nick Fitzgerald ([fitzgen]) about how
WebAssembly and Rust's Async I/O system interact. When you watch it,
you will probably notice two things:

[alexcrichton]: https://github.com/alexcrichton
[fitzgen]: https://github.com/fitzgen

* First, I spent a lot of time looking off to the side! This is
  because I had the joint Dropbox paper document open on my side
  monitor and I forgot how strange that would look. I'll have to
  remember that for the future. =)
* Second, we recorded this on October 3rd[^planning], which was before
  async-await had landed on stable. So at various points we talk about
  async-await being on beta or not yet being stable. Don't be
  confused. =)

[^planning]: Yeah... I've been planning to start these async interviews for a while! Just been busy.
[video]: https://youtu.be/vR0Ry830Hw8

### Video

You can view the [video][video] on YouTube, but it is also embedded
here if that's easier for you.

<center><iframe width="560" height="315" src="https://www.youtube.com/embed/vR0Ry830Hw8" frameborder="0" allow="accelerometer; autoplay; encrypted-media; gyroscope; picture-in-picture" allowfullscreen></iframe></center>

### Rust futures, meet JavaScript promises

The first part of the chat focused on the interaction of Rust futures
with JavaScript's promises. As fitzgen points out early on, on the web
platform, there is no notion of synchronous I/O -- only async[^xhr].
So if you want to execute Rust in the browser, you need to be using
async I/O. Fortunately, the [current tooling makes this pretty easy][rw]!

[^xhr]: We do not speak of [synchronous XHR] on this blog. Or ever.
[synchronous XHR]: https://developer.mozilla.org/en-US/docs/Web/API/XMLHttpRequest/Synchronous_and_Asynchronous_Requests#Synchronous_request
[rw]: https://rustwasm.github.io/wasm-bindgen/reference/js-promises-and-rust-futures.html

The [fetch example] from the [wasm-bindgen] site kind of shows off
some of what is possible. The example shows a Rust function that
downloads content from the web by invoking the [fetch] function. To
start, it shows you can [write a Rust `async fn` that gets exported to
JavaScript](https://github.com/rustwasm/wasm-bindgen/blob/df34cf843eca7478e3879562670e52c889e32fdf/examples/fetch/src/lib.rs#L35-L36):

[fetch]: https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API
[fetch example]: https://github.com/rustwasm/wasm-bindgen/blob/df34cf843eca7478e3879562670e52c889e32fdf/examples/fetch/src/lib.rs
[wasm-bindgen]: https://github.com/rustwasm/wasm-bindgen

```rust
#[wasm_bindgen]
pub async fn run() -> Result<JsValue, JsValue> {
    ...
}
```

When [users invoke this `run` function from
JS](https://github.com/rustwasm/wasm-bindgen/blob/df34cf843eca7478e3879562670e52c889e32fdf/examples/fetch/index.js#L5),
it acts just like a JavaScript asynchronous function, which means that
it returns a JS promise. This is possible because wasm-bindgen
includes the ability to interconvert between JS and Rust promises. You can see
this at play also within the `run` function, which [invokes `fetch` and then
converts the result into a `Rust` future](https://github.com/rustwasm/wasm-bindgen/blob/df34cf843eca7478e3879562670e52c889e32fdf/examples/fetch/src/lib.rs#L50-L51):

```rust
let window = web_sys::window().unwrap();
let resp_value = JsFuture::from(window.fetch_with_request(&request)).await?;
```

Note the `JsFuture::from` call in particular, which converts the
JavaScript promise into a Rust future that can then be awaited.  (You
can go the other way using [`future_to_promise`], which converts a
Rust future into a JavaScript promise.)

[`future_to_promise`]: https://docs.rs/wasm-bindgen-futures/0.4.1/wasm_bindgen_futures/fn.future_to_promise.html

As you can see from this example, a lot of the basic tooling for
interoperating with JavaScript futures exists, but it's also still at
a fairly low-level. The [web-sys] crate used in the [fetch example]
exports all the basic APIs, but does so in an untyped fashion, which
means that using them from Rust can be error-prone. The [gloo] crate
is an attempt to build a more idiomatic layer atop, but that is still
fairly close to the JS APIs. Crates like [surf] offer a higher-level
alternative. [surf] is an interface for fetching things off the web
(think [`libcurl`]), and it [can be compiled] to use a number of
backends, including the JS `fetch` API (which only works when
compiling to webassembly, of course).

[`libcurl`]: https://curl.haxx.se/libcurl/
[can be compiled]: https://docs.rs/surf/1.0.3/surf/#features

[web-sys]: https://crates.io/crates/web-sys
[gloo]: https://crates.io/crates/gloo
[surf]: https://github.com/http-rs/surf

If you're interested to learn more, here are some links to get you started:

* [wasm-bindgen-futures API docs](https://docs.rs/wasm-bindgen-futures/0.4.1/wasm_bindgen_futures/)
* [JS Promises and Rust futures from the wasm-bindgen reference](https://rustwasm.github.io/wasm-bindgen/reference/js-promises-and-rust-futures.html)
* the [wasm-bindgen] repository

### WebAssembly outside the browser

Next we discussed what it mean to use Async I/O *outside* of the
browser. This part of the space is much less developed. When you are
running outside the browser, there aren't standard APIs like [fetch]
to build off of. This is where [WASI] comes in. It is an effort to
build up a standardized set of APIs that WebAssembly apps can run
against which will work both inside and outside the browser. (To learn
more about [WASI], I recommend Lin Clark's excellent [Mozilla Hacks
blog
post](https://hacks.mozilla.org/2019/03/standardizing-wasi-a-webassembly-system-interface/).)

[WASI]: https://wasi.dev/

At this point, the conversation turned much more speculative. WASI
doesn't yet include any asynchronous APIs, but it seems clear that
they will be needed. Alex and Nick felt that some of the things we've
learned in the Rust Async I/O effort would likely inform the design
going forward. For example, WASI might export something vaguely epoll
or mio-like, and let the languages supply the higher-level APIs that
build on that. Still, there has also been talk of including direct
support in WASI for protocols like HTTP, which might necessitate a
different sort of interface altogether.

One of the interesting things we discussed is that WASI is very
*capability driven*. Typically in programming languages, we aim to
expose a small set of powerful, flexible primitives that can be used
to do almost anything -- but this has downsides, too. Offering
higher-level options means you can build a more effective sandbox. For
example, it might be useful to have some way to hand-off an open HTTP
socket to a WASM[^WASM] program, which would mean that it can read and write
through that connection, but that it cannot necessarily create new
HTTP connections to other servers, or speak other protocols.

[^WASM]: [According to the spec], "WASM" is a contraction and hence should not be capitalized. I however maintain that it plainly fits [the definition of an acronym]. Moreover, if it were a contraction, it would be written w'asm', and nobody wants that. So I'm going with WASM. Sue me.

[the definition of an acronym]: https://www.collinsdictionary.com/dictionary/english/acronym
[According to the spec]: https://webassembly.github.io/threads/intro/introduction.html#wasm

### What does this mean for Rust async?

In the last few minutes, we turned our discussion to Rust async
itself. What developments in Rust async would be most helpful to WASM?
Since there are still so many unknowns, especially when it comes to
WASI, this is a bit hard to say for sure, but Nick and Alex had a few
things to say:

* First, in terms of JS interop, building more crates like [surf],
  which can be configured to target web APIs, so that people can write
  Rust programs that work equally well on the web or running natively.
* Second, to support that effort, better support for async fn in
  traits (and perhaps other language primitives). The basic idea is
  that, on the web, async fn is just the only way to do I/O -- this
  means we need it to work seamlessly across all of Rust's language
  features. (As usual, I'll refer folks to dtolnay's [async-trait]
  crate, which is presently the best way to write crates that use
  async fn, for a [host of reasons].)
* Finally, support for async streams (and async generator functions),
  so as to interoperate with their JavaScript equivalents.
  
[async-trait]: https://github.com/dtolnay/async-trait
[host of reasons]: http://smallcultfollowing.com/babysteps/blog/2019/10/26/async-fn-in-traits-are-hard/

### Comments?

There is a [thread on the Rust users forum][thread] for questions and
discussion.

[thread]: https://users.rust-lang.org/t/async-interviews/35167

### Footnotes
