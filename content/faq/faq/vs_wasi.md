+++
title = "How is Waxosuit different than WASI?"
weight = 15
+++

[WASI](https://wasi.dev/), **WebAssembly System Interface** is a standard that essentially takes low-level, _libc_ type calls
and replaces them with WebAssembly _imports_. The types of system calls that are currently covered by WASI are things like file descriptors, sockets, file paths, environment variables, and a handful of others. This means that anything "compiled for WASI" will rely on a _host runtime_ to satisfy these system calls. [Wasmer](https://wasmer.io) and several other host runtimes are "WASI compliant".

**Waxosuit** uses _wasmer_ as a host runtime, but it does _not_ allow WASI calls or any other system calls. In short, **Waxosuit** does not and will not violate the WebAssembly sandbox. WASI-compiled WebAssembly modules have _direct, unfettered system access_, and one of Waxosuit's main tenets is providing a secure, capabilities-driven sandbox in whic WebAssembly modules run.