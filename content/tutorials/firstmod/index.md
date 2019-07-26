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

* **waxosuit** - Eventually you'll want this installed, but for now you can just use the docker image.
* **wascap** - you will need the [wascap](https://github.com/waxosuit/wascap) tool installed (`cargo install wascap --features "cli"`). This is used to sign and verify WebAssembly modules.
* **cargo generate** - you will need the [cargo-generate crate](https://crates.io/crates/cargo-generate) installed (`cargo install cargo-generate`) to create a project from the waxosuit-guest-template

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
$ make build
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
In future tutorials and the documentation you'll see how to sign WebAssembly modules. For now, this step is taken care of for you because
the new project template comes with a `.keys` directory, containing some development keys. All you need is `wascap` in your path for the `Makefile`
to work.

Both the **build** and **release** tasks in the **Makefile** will automatically sign your module.

<a name="run"></a>

## Run Your Module in Waxosuit
It's time to run our module!

To do this, we're going to create a docker image from the **waxosuit/waxosuit** image and will combine that with the signed WebAssembly module produced during the build process. To create the docker image, just issue the following command:

```session
$ make docker
```

You'll see the release build get compiled and then you'll see the _wasm_ file get added to the docker image.

Now you can just use docker to run your WebAssembly module: (you'll need to use `docker kill` to terminate the process when you're done)

```session
$ docker run -e RUST_LOG="info,cranelift_wasm=warn" -p 8080:8080 hello-world
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

We set the `RUST_LOG` environment variable so we don't see the spam from the **Cranelift** compiler as it JITs the WebAssembly instructions and so we can see useful information coming from the WebAssembly module, Waxosuit, and its capability providers. You can see in the preceding output that the _Messaging (NATS)_ and _Key-Value (Redis)_ provider plug-ins both get unloaded because the signed Wasm module doesn't have those capability grants.

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

Note that the values for `module` and `issuer` in the above JSON will be the _public keys_ of the account and module that came with the new project template.

To test that your WebAssembly module is functioning the way it should, issue any HTTP request on the **8080** port and it should return a **HTTP 200** status code.

**Congratulations** ! You've compiled, signed, and executed a WebAssembly module designed to run in the cloud. In the next tutorial, you'll start using the capabilities available to your module.
