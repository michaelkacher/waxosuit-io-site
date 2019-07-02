+++
fragment = "content"
weight = 100

title = "Your First Module"
subtitle = "Creating and Running a Waxosuit WebAssembly Module"
background = "light"


[sidebar]
  title = "Steps"
  align = "left"
  #sticky = true # Default is false
  content = """
[Pre-requisites](#prereqs)<br/>
[Create a Module](#create)<br/>
[Sign a Module](#sign)<br/>
[Run it](#run)
"""
+++

In this tutorial, you'll be creating your first WebAssembly module for use with **Waxosuit**. This will consist of three main steps:

* Create and build the WebAssembly Module
* Sign the module
* Run the module in Waxosuit

<a name="prereqs"></a>

## What You'll Need to Start
You'll need to have a few things ready before you can do this tutorial:

* **waxosuit** - you will need to either have [compiled](https://github.com/waxosuit/waxosuit) this binary and the default capability provider plugins, or you'll need to run the docker image.
* **nkeys** - you will need the [nkeys](https://github.com/encabulators/nkeys) tool installed (`cargo install nkeys`). This is used to create ed25519 signing keys.
* **wascap** - you will need the [wascap](https://github.com/waxosuit/wascap) tool installed (`cargo install wascap --features "cli"`). This is used to sign and verify WebAssembly modules.

<a name="create"></a>

## Create and Build a Module
To create a new **Waxosuit module** project in Rust, you can use `cargo generate` to produce a new project from our project template as shown in the following example:

```bash
$ cargo generate --git https://github.com/waxosuit/waxosuit-guest-template
Project Name: HelloWorld
Renaming project called `HelloWorld` to `hello-world`...
Creating project called `hello-world`...
Done! New project created /home/kevin/demo/hello-world
```

In this example we chose `HelloWorld` as the project name, but you can choose anything you like. As soon as this new project is generated, you can go to that project's directory and compile it (already configured to build to the `wasm32-unknown-unknown` target):

```session
$ cd hello-world
$ cargo build
...
... 
 Compiling hello-world v0.1.0 (/home/kevin/demo/hello-world)
    Finished dev [unoptimized + debuginfo] target(s) in 41.23s
```

This new module comes with a default _call handler_ (covered in the next tutorial in the series) that will answer all incoming HTTP requests with a **200/OK**, and it also answers health requests and identity requests by default. You'll see that when we go to run it with the **waxosuit** binary soon.

```bash
$ ls target/wasm32-unknown-unknown/debug/*.wasm
target/wasm32-unknown-unknown/debug/hello_world.wasm
```

<a name="sign"></a>

## Sign a Module
Once you've created your portable WebAssembly binary, you need to sign it with the list of capabilities you want before Waxosuit can run it. You'll need the [nkeys](https://github.com/encabulators/nkeys) tool installed, which places the `nk` binary in your path. Make sure you have at least version _0.0.7_ of this tool installed (you can install with `cargo install nkeys --features "cli"`).

_Note_ - if you already have the Go version (from the NATS github repository) of _nkeys_ installed, you might accidentally run that instead of ours. Make sure that ours shows up first in your path if you have both. Run `nk --version` to see which one you've got.

First, let's generate a primary and seed (private) key for an _account_. An account in this case is just an authorized signer for the JWT that will be embedded in the WebAssembly module we just created.

```bash
$ nk gen account
Public Key: ACTJZQGTL76T4BBNEVZCISJV7F7NVTJJWLQRUOHWGUWHJX7XKMRJ73RL
Seed: SAAE3FRJS7JRFQ625MZ3D6FXKXQQRBYUUUYAZQYHENKZERRXXTSV5RFWOE

Remember that the seed is private, treat it as a secret.
```

The keys generated will be different for you than in the preceding sample. What _will_ be the same is the public key will have an **A** prefix and the seed key will have the **SA** prefix. Copy the private seed key and add it to a file called `account.nk`. A fun side effect of this type of encryption is that you don't need the public key here, we can re-establish it if we have the seed. 

Next, generate a key for the _module_, which establishes the unique identity of your WebAssembly module:

```session
$ nk gen module
Public Key: MDPP5KS2ETC3DSL7PZ6DZVBR6YFBQL32EK3MWITCC36JMWO4QBQOM2WT
Seed: SMAN7NQ7NXBWJI6JIRCG4CWN7K4ASW6EIBQUKNIQUK2JNMVTAASKEK5MC4

Remember that the seed is private, treat it as a secret.
```

Here the key prefix is **M** for the public key and **SM** for the seed key. Copy the seed key to a file called `module.nk`.


With the _account_ and _module_ keys generated and the WebAssembly file created, it's time to sign it. 

In the sample terminal output below, we use the `wascap` binary to grant the _HTTP Server_ capability to the module. Note that you might need to change the source filename if you didn't use "hello world" earlier.

```session
$ wascap sign -s -a ./account.nk -m ./module.nk -t tutorial target/wasm32-unknown-unknown/debug/hello_world.wasm ./hello.wasm
```

Once your module has been signed, you can use the **wascap** CLI to verify the signature and display the registered capabilities:

```session
$ wascap caps hello.wasm 
╔════════════════════════════════════════════════════════════════════════════╗
║                                WASCAP Module                               ║
╠═══════════════╦════════════════════════════════════════════════════════════╣
║ Account       ║   ACNIHZFDBJZD7SC2B6XDMI6UAXSHM5OCPRHHSXXUMGVCGDTM43QJKUTH ║
╠═══════════════╬════════════════════════════════════════════════════════════╣
║ Module        ║   MCWVBKLK2KMIG2SIX7PKICCXPD7UXUAI56NVEIACBIG5ONQMFVNIF2JL ║
╠═══════════════╬════════════════════════════════════════════════════════════╣
║ Expires       ║                                                      never ║
╠═══════════════╬════════════════════════════════════════════════════════════╣
║ Can Be Used   ║                                                immediately ║
╠═══════════════╩════════════════════════════════════════════════════════════╣
║                                Capabilities                                ║
╠════════════════════════════════════════════════════════════════════════════╣
║ HTTP Server                                                                ║
╠════════════════════════════════════════════════════════════════════════════╣
║                                    Tags                                    ║
╠════════════════════════════════════════════════════════════════════════════╣
║ None                                                                       ║
╚════════════════════════════════════════════════════════════════════════════╝
```

**Congratulations!!** - you now have a fully signed and provenance-verifiable WebAssembly module that's ready to run as a Waxosuit guest module!

<a name="run"></a>

## Run Your Module in Waxosuit
It's time to run our module! 

Use the following command (which assumes you have `waxosuit` compiled and in your path, and the capabilities plugins (the `.so` files from the waxosuit compilation output) have been copied to `./waxosuit`) to launch your module. Make sure you use the same file name as the _signed output_ from the previous step as Waxosuit will reject any unsigned WebAssembly module.

```session
$ RUST_LOG=info,cranelift_wasm=warn waxosuit -c ./waxosuit ~/demo/hello-world/hello.wasm 
[2019-06-28T05:46:42Z INFO  waxosuit_host::capabilities] Loaded capability: wascap:http_server, provider: Actix-Web HTTP Server
[2019-06-28T05:46:42Z INFO  wascap_httpsrv] Dispatcher received.
[2019-06-28T05:46:42Z INFO  waxosuit] Capability provider wascap:http_server loaded
[2019-06-28T05:46:42Z INFO  actix_server::builder] Starting 4 workers
[2019-06-28T05:46:42Z INFO  actix_server::builder] Starting server on 0.0.0.0:8080
[2019-06-28T05:46:42Z INFO  waxosuit_host::capabilities] Loaded capability: wascap:messaging, provider: NATS Messaging Provider
[2019-06-28T05:46:42Z INFO  waxosuit_host::capabilities] Capability provider for wascap:messaging not claimed by guest module. Unloading.
[2019-06-28T05:46:42Z INFO  waxosuit] Capability provider not loaded: WASCAP contract violation: Unauthorized capability: wascap:messaging
[2019-06-28T05:46:42Z INFO  waxosuit_host::capabilities] Loaded capability: wascap:keyvalue, provider: Redis Key-Value Provider
[2019-06-28T05:46:42Z INFO  waxosuit_host::capabilities] Capability provider for wascap:keyvalue not claimed by guest module. Unloading.
[2019-06-28T05:46:42Z INFO  waxosuit] Capability provider not loaded: WASCAP contract violation: Unauthorized capability: wascap:keyvalue
[2019-06-28T05:46:42Z INFO  waxosuit] Starting Waxosuit for module hello with capability claims - wascap:http_server
```

We set the `RUST_LOG` environment variable so we don't see the spam from the **Cranelift** compiler as it JITs the WebAssembly instructions and so we can see useful information coming from the WebAssembly module, Waxosuit, and its capability providers. You can see in the preceding output that the _Messaging (NATS)_ and _Key-Value (Redis)_ provider plug-ins both get unloaded because the signed
Wasm module doesn't have those capability grants.

Now that **Waxosuit** is hosting your Wasm file, open another terminal window and you can curl the _identity_ endpoint that the Waxosuit HTTP server provides automatically:

```session
$ curl localhost:8080/id | jq
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100   231  100   231    0     0  57750      0 --:--:-- --:--:-- --:--:-- 77000
{
  "module": "MCWVBKLK2KMIG2SIX7PKICCXPD7UXUAI56NVEIACBIG5ONQMFVNIF2JL",
  "issuer": "ACNIHZFDBJZD7SC2B6XDMI6UAXSHM5OCPRHHSXXUMGVCGDTM43QJKUTH",
  "capabilities": [
    "wascap:keyvalue",
    "wascap:logging",
    "wascap:http_client",
    "wascap:http_server"
  ]
}
```

Note that the values for `module` and `issuer` in the above JSON will be the _public keys_ of the account and module that you generated in a previous step.


To test that your WebAssembly module is functioning the way it should, issue any HTTP request on the **8080** port and it should return a **HTTP 200** status code.

**Congratulations** again! You've compiled, signed, and executed a WebAssembly module designed to run in the cloud. In the next tutorial, you'll start using the capabilities available to your module.
