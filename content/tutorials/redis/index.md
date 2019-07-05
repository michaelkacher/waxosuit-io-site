+++
fragment = "content"
weight = 100

title = "Using the Key-Value Capability"
subtitle = "Learn how to write code using the key-value capability and configure it at runtime"
background = "light"


[sidebar]
  title = "Steps"
  align = "left"
  #sticky = true # Default is false
  content = """
[Pre-requisites](#prereqs)<br/>
[Using the Redis KV Provider](#codeprovider)<br/>
[Sign the Module](#sign)<br/>
[Configuring the Redis Plugin](#redisconfig)<br/>
[Run it](#run)
"""
+++

In this tutorial, you'll be modifying the [first tutorial](/tutorials/firstmod)'s code to make use of a key-value capability provider.


<a name="prereqs"></a>

## What You'll Need to Start
You'll need to have a few things ready before you can do this tutorial:

* **waxosuit** - you will need to either have [compiled](https://github.com/waxosuit/waxosuit) this binary and the default capability provider plugins, or you'll need to run the docker image.
* **nkeys** - you will need the [nkeys](https://github.com/encabulators/nkeys) tool installed (`cargo install nkeys`). This is used to create ed25519 signing keys.
* **wascap** - you will need the [wascap](https://github.com/waxosuit/wascap) tool installed (`cargo install wascap --features "cli"`). This is used to sign and verify WebAssembly modules.
* **redis** - you will need a locally running copy of Redis (either the daemon or the running docker image is fine)
* **First Tutorial** - You will need to have completed the [first tutorial](/tutorials/firstmod)


<a name="codeprovider"></a>

## Using the Key-Value Provider
In this simple example, we're going to be atomically incrementing a value every time our guest module receives an HTTP _POST_ request, and we will return a JSON payload with the current value whenever we get an HTTP _GET_ request. This will illustrate using the [CapabilitiesContext](https://docs.rs/wascap-guest/0.0.3/wascap_guest/struct.CapabilitiesContext.html) struct with the [KeyValueStore](https://docs.rs/wascap-guest/0.0.3/wascap_guest/kv/struct.KeyValueStore.html) trait in the Rust guest SDK.

Open up the **lib.rs** file from the previous solution and set the code to the following:

<script src="https://gist.github.com/autodidaddict/4c14d5dc6bec1c2470beede314aef8f4.js"></script>

Now add the following dependencies to your **Cargo.toml** file:

```session
serde_json = "1.0.39"
serde_derive = "1.0.91"
serde = "1.0.91"
```


<a name="sign"></a>

## Sign the Wasm Module
First make sure your module compiles (you should be able to simply _cargo build_ and produce a **wasm32-unknown-unknown** target). 

Next, use the `wascap` tool and some keys you should have generated in the previous tutorial to sign the module:

```session
$ wascap sign -a ~/waxosuit/iot-example/keys/demo_account.nk -m ~/waxosuit/iot-example/keys/sensorservice.nk -s -k target/wasm32-unknown-unknown/debug/hello_world.wasm ./hello.wasm
Successfully signed ./hello.wasm.
```
Note that the paths to your keys will likely be different. 

We can verify that the module is signed and ready to run in waxosuit:

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
║ K/V Store                                                                  ║
║ HTTP Server                                                                ║
╠════════════════════════════════════════════════════════════════════════════╣
║                                    Tags                                    ║
╠════════════════════════════════════════════════════════════════════════════╣
║ None                                                                       ║
╚════════════════════════════════════════════════════════════════════════════╝
```

<a name="redisconfig"></a>

## Configuring the Redis Plugin
Each capability made available to Waxosuit guest modules is an abstraction. You write your code to use the abstract capabilities of a key-value store, and then at runtime waxosuit will securely bind a specific capability provider to your module.

In the case of this tutorial, we'll be using the capability providers that ship with waxosuit, which includes a **Redis** key-value store. Before continuing, make sure that you're running a Redis server either via docker or as a local system daemon. 

By default, the Redis capability provider plug-in will attempt to connect to the default Redis port on the local host. For more information on the environment variables available to configure Redis connectivity, check out the plugin in the [Waxosuit repository](https://github.com/waxosuit/waxosuit/tree/master/wascap-redis)

<a name="run"></a>

## Running the Wasm Module
Let's start up **waxosuit** and exercise this code. Keep in mind this tutorial code doesn't cover any edge cases (it'll crash if you query before you increment the counter). As an exercise, after you complete this tutorial, you should go back and play with different ways to make that case safe, with or without an empty database.


```session
$ RUST_LOG=info,cranelift_wasm=warn waxosuit hello.wasm --caps /home/kevin/waxosuit/waxosuit/target/debug 
[2019-07-05T15:35:35Z INFO  waxosuit_host::capabilities] Loaded capability: wascap:http_server, provider: Actix-Web HTTP Server
[2019-07-05T15:35:35Z INFO  wascap_httpsrv] Dispatcher received.
[2019-07-05T15:35:35Z INFO  waxosuit] Capability provider wascap:http_server loaded
[2019-07-05T15:35:35Z INFO  actix_server::builder] Starting 4 workers
[2019-07-05T15:35:35Z INFO  actix_server::builder] Starting server on 0.0.0.0:8080
[2019-07-05T15:35:35Z INFO  waxosuit_host::capabilities] Loaded capability: wascap:messaging, provider: NATS Messaging Provider
[2019-07-05T15:35:35Z INFO  waxosuit_host::capabilities] Capability provider for wascap:messaging not claimed by guest module. Unloading.
[2019-07-05T15:35:35Z INFO  waxosuit] Capability provider not loaded: WASCAP contract violation: Unauthorized capability: wascap:messaging
[2019-07-05T15:35:35Z INFO  waxosuit_host::capabilities] Loaded capability: wascap:keyvalue, provider: Redis Key-Value Provider
[2019-07-05T15:35:35Z INFO  wascap_redis] Dispatcher received.
[2019-07-05T15:35:35Z INFO  waxosuit] Capability provider wascap:keyvalue loaded
[2019-07-05T15:35:35Z INFO  waxosuit] Starting Waxosuit for module hello with capability claims - wascap:keyvalue, wascap:http_server
```

With the service up and running, send a **POST** to increment the counter (you'll need to use a new terminal window/tab):

```bash
$ curl -X POST localhost:8080
```

It should appear as though nothing happened, but we can observe the following log output from the **waxosuit** process:

```session
[2019-07-05T15:35:49Z INFO  waxosuit_host::dispatch] Dispatching type.wascap.io/http.Request from wascap:http_server to guest
[2019-07-05T15:35:49Z INFO  waxosuit_host::wasm] Wasm Guest: Invoking handler...
[2019-07-05T15:35:49Z INFO  waxosuit_host::wasm] Guest module invoking host call for wascap:keyvalue
[2019-07-05T15:35:49Z INFO  wascap_redis] Received host call, command - type.wascap.io/keyvalue.AddRequest
[2019-07-05T15:35:49Z INFO  actix_web::middleware::logger] 127.0.0.1:47670 "POST / HTTP/1.1" 200 0 "-" "curl/7.64.0" 0.005200
``` 

Probably the most important piece of information in this log is the mention of the command **type.wascap.io/keyvalue.AddRequest**. This is the protobuf message that is sent from the guest module to the host. This command is handled by the Redis capability provider and returned to the guest module in the form of an **Event**.

Go ahead and send a few more **POST**s to your new service, and then finally query the value with a simple **GET**:

```session
$ curl localhost:8080
{"counter":3}
```

To recap--in this tutorial you added just a few lines of code to a minimal set of scaffolding and you've now got a _secure_, _signed_ WebAssembly module that is capable of handling HTTP requests and interacting with a live Redis database. All of this was done with almost no boilerplate and without violating the Wasm sandbox.

Check out the [other tutorials](/getstarted) where you'll see just how much functionality and power you can get in your services for free, and how writing business logic for cloud services and functions can be both secure and easy.
