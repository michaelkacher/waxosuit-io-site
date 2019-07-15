+++
fragment = "content"
weight = 100

title = "Publishing to the Wapm Registry"
subtitle = "Learn How to Publish a Signed Module to the Wapm Registry"
background = "light"


[sidebar]
  title = "Steps"
  align = "left"
  #sticky = true # Default is false
  content = """
[Pre-requisites](#prereqs)<br/>
[Creating a new Module](#newmodule)<br/>
[Sign the Module](#sign)<br/>
[Publish to Wapm](#publish)<br/>
[When Should I Publish](#when)
"""
+++

In this tutorial, you'll be creating a new module that echoes the details of an incoming HTTP request back as an HTTP response. Not only is this a helpful diagnostic tool, but we'll show you how to sign this module and then _publish_ it to [wapm.io](https://wapm.io).


<a name="prereqs"></a>

## What You'll Need to Start
You'll need to have a few things ready before you can do this tutorial:

* **waxosuit** - you will need to either have [compiled](https://github.com/waxosuit/waxosuit) this binary and the default capability provider plugins, or you'll need to run the docker image.
* **nkeys** - you will need the [nkeys](https://github.com/encabulators/nkeys) tool installed (`cargo install nkeys --features "cli"`). This is used to create ed25519 signing keys.
* **wascap** - you will need the [wascap](https://github.com/waxosuit/wascap) tool installed (`cargo install wascap --features "cli"`). This is used to sign and verify WebAssembly modules.
* **wapm** - you will need to have installed the [wapm](https://wapm.io/help/install) CLI (required to publish to the registry)



<a name="newmodule"></a>

## Creating a New Module
The instructions here should be familiar if you've already gone through any of the basic tutorials. The first step is to generate your new module:

```
$ cargo generate --git https://github.com/waxosuit/waxosuit-guest-template
```
Choose a suitable name for this project. We selected `EchoExample`, which placed the code in the `echo-example` directory.

Next, add the **serde** dependency to your *Cargo.toml* file:

```
[package]
name = "echo-example"
version = "0.1.0"
authors = ["kevin"]
edition = "2018"

[lib]
crate-type = ["cdylib"]

[dependencies]
wascap-guest = "0.0.3"
serde = { version = "1.0.91", features = ["derive"]}

[profile.release]
# Optimize for small code size
opt-level = "s"
```

Now you can add the simple code that echoes HTTP request details as a JSON response to your _src/lib.rs_ file:

<script src="https://gist.github.com/autodidaddict/996ee9168b8fa029bd272d6678cfa9cc.js"></script>

<a name="sign"></a>

## Sign the Wasm Module
Make sure your module compiles (you should be able to simply _cargo build_ and produce a **wasm32-unknown-unknown** target).

Next, use the `wascap` tool and some keys you may have generated in the previous tutorial(s) to sign the module (or you can use `nk` to create more keys):

```session
$ wascap sign target/wasm32-unknown-unknown/debug/hello_example.wasm -k -s -g -a /path/to/account.nk -m /path/to/module.nk
Successfully signed ./hello_example.wasm.
```
Make sure you replace the placeholders with the actual paths to your keys and Wasm file.

We can verify that the module is signed and ready to run in waxosuit:

```session
$ wascap caps ../echo-example-wapm/echo-example.wasm
╔════════════════════════════════════════════════════════════════════════════╗
║                                WASCAP Module                               ║
╠═══════════════╦════════════════════════════════════════════════════════════╣
║ Account       ║   ABHUOCYPSFAYVQXEVMOSGQKABFF4LUS35EFE3VCEG3BIEJBGI5GUOCUN ║
╠═══════════════╬════════════════════════════════════════════════════════════╣
║ Module        ║   MBY3SLBLMVLQY7KBULCUK3L2GVPPFY5AIWF3P5UBM2OON6NE3HW4JKRJ ║
╠═══════════════╬════════════════════════════════════════════════════════════╣
║ Expires       ║                                                      never ║
╠═══════════════╬════════════════════════════════════════════════════════════╣
║ Can Be Used   ║                                                immediately ║
╠═══════════════╩════════════════════════════════════════════════════════════╣
║                                Capabilities                                ║
╠════════════════════════════════════════════════════════════════════════════╣
║ K/V Store                                                                  ║
║ Messaging                                                                  ║
║ HTTP Server                                                                ║
╠════════════════════════════════════════════════════════════════════════════╣
║                                    Tags                                    ║
╠════════════════════════════════════════════════════════════════════════════╣
║ None                                                                       ║
╚════════════════════════════════════════════════════════════════════════════╝
```

<a name="publish"></a>

## Publishing the Module to the Wapm Registry
Now that we've got a compiled, signed WebAssembly module that we know conforms to **wascap** and runs within **waxosuit**, let's publish it to [wapm.io](https://wapm.io). The first thing we'll need in order to publish it is a wapm module manifest. This is a simple _toml_ file that describes our module. The one I used for the published [echo example](https://wapm.io/package/autodidaddict/echo-example) looks like this:

<script src="https://gist.github.com/autodidaddict/ef8fc8b83a7a5f4340405c6e8d40caef.js"></script>

You'll want to change the values for these properties and make sure that that the **source** property points to the Wasm module you created in the first step. To double-check and make sure everything works the way it should, you can do a _dry-run_:

```
$ wapm publish --dry-run
Successfully published package `autodidaddict/echo-example@0.0.3`
[INFO] Publish succeeded, but package was not published because it was run in dry-run mode
```

And when you're ready to do the final publication, you can just execute `wapm publish`.

<a name="when"></a>

## When Should I Publish To Wapm?
Wapm is a public registry, so at the very least you'll want to be working on something you intend to distribute to the general public to use Wapm. The combination of Wapm and Waxosuit gives you the best of both worlds--you get public distribution of your module, and your module also remains signed and tamper-proof, allowing consumers of your module to make smart (possibly _OPA-powered_) decisions about whether the module should run or if its capability attestations should be honored.

If you wanted to make an application out of one or more WebAssembly modules that run within Waxosuit, you could use wapm.io as your distribution vehicle without sacrificing the security and provenance verification you get with Wascap. 
