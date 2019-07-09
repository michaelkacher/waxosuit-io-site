+++
fragment = "content"
weight = 100

title = "Integrating with OPA"
subtitle = "Execute policy against your Wasm module's embedded JWT"
background = "light"


[sidebar]
  title = "Steps"
  align = "left"
  #sticky = true # Default is false
  content = """
[Pre-requisites](#prereqs)<br/>
[Write a Policy](#policy)<br/>
[Configure OPA Integration](#opaconfig)<br/>
[Run it](#run)
"""
+++

[Open Policy Agent](https://www.openpolicyagent.org/) is a CNCF project that empowers administrators with fine-grained, flexible policy controls. Think of it as a "deferred brain" to which decisions can be delegated by evaluating policies against live, up-to-date raw and massaged data.

**Waxosuit** supports OPA integration by optionally submitting the signed, embedded **JWT** from the Wasm module to OPA as input to a given policy. If you enable this feature, then your OPA policy must return a JSON payload with a single boolean field `allow`. If this value is _true_, then the Wasm module in question will be allowed to run, otherwise the Waxosuit host process will terminate with an appropriate error message.

All you have to do is supply a value for the OPA policy URL environment variable, and **waxosuit** will take care of the rest.


<a name="prereqs"></a>

## What You'll Need to Start
You'll need to have a few things ready before you can do this tutorial:

* **waxosuit** - you will need to either have [compiled](https://github.com/waxosuit/waxosuit) this binary and the default capability provider plugins, or you'll need to run the docker image.
* **nkeys** - you will need the [nkeys](https://github.com/encabulators/nkeys) tool installed (`cargo install nkeys`). This is used to create ed25519 signing keys.
* **wascap** - you will need the [wascap](https://github.com/waxosuit/wascap) tool installed (`cargo install wascap --features "cli"`). This is used to sign and verify WebAssembly modules.
* **OPA** - you will need a locally running copy of the Open Policy Agent or have it running via the `openpolicyagent/opa:latest` docker image.
* **First Tutorial** - You will need to have completed the [first tutorial](/tutorials/firstmod), though virtually any tutorial that produces a signed Wasm file should suffice


<a name="policy"></a>

## Write a Policy
Probably the simplest policy we can write would be one that requires any module to have been issued by a given account. This type of policy might be used to require a specific issuer for _production-bound_ workloads whereas a larger set of issuers might be okay for lower environments.

By convention, the _issuer_ of a Wascap-signed module is the _public key_ of an **account**. For more information on the various types of keys, check out the [nkeys](https://github.com/encabulators/nkeys) crate.

Waxosuit's OPA integration requires two fields in the JSON output:
  
  * **allow** - A boolean indicating whether the module should be allowed to run
  * **cause** - An array of strings with a list of justifications for the allow field (will be an empty array when **allow** is true)

Now let's write a policy that conforms to this interface. If you've done the other tutorials, you should have a valid _account_ public key lying around. You can also just grab an issuer by running `wascap caps (mymodule).wasm`, you'll see the issuer's public key in the output.

Create the following `authz.rego` file and replace the **"A..."** text with the public key of your account.

```
package system

default allow = false

import input

token = {"payload": payload} { io.jwt.decode(input.token, [_, payload, _]) }

allow {
	valid_issuer
}

cause["issuer not valid"] {
	not valid_issuer
}

valid_issuer {
    token.payload.iss == "ACNIHZFDBJZD7SC2B6XDMI6UAXSHM5OCPRHHSXXUMGVCGDTM43QJKUTH"
}

main = res {
    res := {
        "allow": allow,
        "cause": cause
    }
}
```

<a name="opaconfig"></a>

## Configure OPA Integration
Configuring OPA integration is as simple as setting an environment variable that **Waxosuit** can read. This variable points to the full URL of the policy to evaluate.

In the case of this tutorial, we're creating a policy that gets evaluated as the default (root) URL. For more information on how to create a hierarchy of policies, consult the OPA documentation.

Set the `OPA_URL` environment variable to `http://0.0.0.0:8181`. If you're running OPA on a different address, make sure to set the environment variable appropriately.

**NOTE** - In this tutorial setting, it's fine to use an unsecured connection to OPA, but in production you will want, at the very least, TLS enabled. For more information on securing OPA endpoints, check out the [OPA Documentation](https://www.openpolicyagent.org/docs/latest/security/) on the subject.


<a name="run"></a>

## Run It
The first thing you're doing to want to do is start the OPA server. If you want to do this inside docker compose, you can refer to the [IoT example](https://github.com/waxosuit/iot-example) for a sample `compose.yml` file.

You can start OPA from the command line:
```
$ opa run --server --log-level=debug authz.rego
```
You should now have a running instance of OPA with a single root-level policy.

With OPA running and the `OPA_URL` environment variable pointing to OPA, you can start **Waxosuit** with any guest module you like. First, pass it one with the right issuer and you should see the following text somewhere in Waxosuit's console log:

```session
[2019-07-05T16:28:07Z INFO  waxosuit] OPA validation PASSED
```

Next, let's see what happens when the answer from an OPA module is to deny (the `allow` field is _false_). You can make this happen one of two ways:

* Create a new signed Wasm module signed by a different/invalid issuer
* Modify the `authz.rego` file so that the check for valid issuer fails

Now when you run Waxosuit you should see output that looks like the following:
```session
[2019-07-09T18:06:10Z INFO  waxosuit] OPA validation DENIED
Error: Error(WascapViolation("OPA denied this module: issuer not valid"))
```

Waxosuit will then terminate as a result of OPA denying the module, even though the JWT signature is valid and the contents all could be valid. Integrating with OPA gives you the ability to execute these policies to have the final say over which workloads can and cannot be scheduled.
