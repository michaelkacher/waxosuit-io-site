+++
title = "Can I Use Waxosuit to Build Knative Functions?"
weight = 40
+++

**Yes!** A **[Knative](https://cloud.google.com/knative/)** function is really just a service that launches, exposes an HTTP endpoint, and reacts to incoming web requests. Knative manages scaling this service to zero (no running copies) or up to however many copies you want to run. Waxosuit might have a slightly higher _cold start_ time (at least in its current infantile development stage), but once warmed up will execute at native speeds and can handle whatever workloads you throw at it.

In order for your Wasm modules to work with **[Knative](https://cloud.google.com/knative/)**, they _must_ be granted the **wascap:http_server** capability when signed.
