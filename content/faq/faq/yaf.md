+++
title = "Isn't Waxosuit just Yet Another Framework (YAF)?"
weight = 50
+++

In short, no. Waxosuit is an implementation of a set of [standards](https://wascap.io). Firstly, these standards form a foundation for securely signing, verifying, identifying module provenance, and validating capability claim attestations. Secondly, these standards describe the means by which the host runtime and the guest module communicate, which we analagously refer to as "protobufs over Wasm FFI"

Waxosuit's Rust implementation _does_ include a [Guest SDK](https://github.com/waxosuit/wascap-guest), which is a small framework that encapsulates the low-level protocol buffer communication protocol in a way that makes the developer experience of building guest modules easier, simpler, and more ergonomic.