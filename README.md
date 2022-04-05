# `tracing-wasm`

Leverage performance profiling with your browser tools with the [tracing crate](https://crates.io/crates/tracing).

[![Crates.io][crates-badge]][crates-url]
[![Documentation][docs-badge]][docs-url]
[![MIT licensed][mit-badge]][mit-url]
[![APACHE licensed][apache-2-badge]][apache-2-url]

[crates-badge]: https://img.shields.io/crates/v/tracing-wasm.svg
[crates-url]: https://crates.io/crates/tracing-wasm
[docs-badge]: https://docs.rs/tracing-wasm/badge.svg
[docs-url]: https://docs.rs/tracing-wasm
[mit-badge]: https://img.shields.io/badge/license-MIT-blue.svg
[mit-url]: LICENSE-MIT
[apache-2-badge]: https://img.shields.io/badge/license-APACHE%202.0-blue.svg
[apache-2-url]: LICENSE-APACHE

![Screenshot of performance reported using the `tracing-wasm` Subscriber](./2020-07-10-devtools-demo-screenshot.png)

Note: `tracing_wasm` uses the global JavaScript `console` and `performance` objects. It will not work in environments where one or both of these are not available, such as Node.js or Cloudflare Workers.

## Reason for this fork

Using regular [span
tracing](https://docs.rs/tracing/latest/tracing/span/index.html) (either
explicitly or via the [instrument
attribute](https://docs.rs/tracing/latest/tracing/attr.instrument.html)) often
works, but at least with Chrome 100, non-deterministically fails. The failure
symptom is that rather than getting a sensible set of spans, like this:

![Expected output](./expected.png)

one or more spans starts correctly, but never terminates, extending to the end
of the sample set:

![Bad output](./bad.png)

Exporting the JSON from the Chrome dev tools reveal that the bad spans are the
usual `blink.user_timing` entries that have a valid `"ph":"b"` begin entry, but
no corresponding `"ph": "e"` end entry. ([See
spec](https://docs.google.com/document/d/1CvAClvFfyA5R-PhYUmn5OOQtYMH4h6I0nSsKchNAySU/preview).)

I don't know why this is. But a brute-force way to fix this seems to be to use
the variant of
[performance.measure()](https://developer.mozilla.org/en-US/docs/Web/API/Performance/measure)
that uses two explicit marks, rather than a single mark for the start of the
span. This fork is a minimal implementation of that idea. This does result in
extra "ticks" but otherwise seamlessly fixes the problem for me:

![Resulting output](./result.png)

## Usage

For the simplest out of the box set-up, you can simply set `tracing_wasm` as your default tracing Subscriber in wasm_bindgen(start)

We have this declared in our `./src/lib.rs`

```rust
#[wasm_bindgen(start)]
pub fn start() -> Result<(), JsValue> {
    // print pretty errors in wasm https://github.com/rustwasm/console_error_panic_hook
    // This is not needed for tracing_wasm to work, but it is a common tool for getting proper error line numbers for panics.
    console_error_panic_hook::set_once();

    // Add this line:
    tracing_wasm::set_as_global_default();

    Ok(())
}
```
