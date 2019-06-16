+++
title = "Is Waxosuit Compatible with Kubernetes?"
weight = 30
+++

**Yes!** Waxosuit can host a WebAssembly module anywhere the small **waxosuit** process can run. Waxosuit is designed from the ground up to be cloud native and to work seamlessly from inside a _Docker_ image. You can use Kubernetes to create a _deployment_ that executes the Waxosuit binary and loads a WebAssembly module either pulled from a container-mounted volume or from a **Gantry** (Under Development) repository.