---
title:  'Writing Envoy WASM Filters The Hardest Way: Using WAT'
subtitle: ''
categories: [Envoy, WASM, WAT]
date:   2022-04-23
---

If you hadn't noticed, I have started [looking into WebAssembly and WASI](https://evacchi.github.io/llvm/wasm/wasi/2022/04/14/compiling-llvm-ir-into-wasm.html). 

Last time I was looking into the LLVM toolchain. In the last few days, I have been reading about [Envoy and its WASM filters][envoy-wasm]. While there are a number of tutorials on how to use the [Proxy WASM SDK][proxy-sdk] using higher-level languages such as C++, Go, Rust, Zing and AssemblyScript, I could not find any resource on how to write a WASM extension from scratch using the canonical text format WAT and the lower-level tool [`wat2wasm`][wabt]. 

To be fair, the [proxy-wasm ABI spec][proxy-wasm-spec] is very well-written and it can be easily used as a reference. However, for [a newcomer like me](/assets/envoy/ihavenoideadog.jpeg), it is not clear by trying examples that employ higher-level language SDKs what is the minimal amount of code that has to be provided in a WASM filter in order to successfully initialize and boot Envoy.

So here is my attempt. From trial and error, it looks like Envoy will happily boot up if you implement at least two functions:

- `proxy_abi_version_0_2_0`
- `proxy_on_memory_allocate`

This will make for a pretty lousy Envoy filter, but it's a good way to get started.

Create `tiny.wat`:

```clj
(module
    (func $proxy_abi_version_0_2_0 
    nop)
    (func $proxy_on_memory_allocate (param $memory_size i32) (result i32) 
    i32.const 0 ;; do not allocate any memory
    return)
    (export "proxy_abi_version_0_2_0" (func $proxy_abi_version_0_2_0))
    (export "proxy_on_memory_allocate" (func $proxy_on_memory_allocate))
)
```

then build `tiny.wasm`

```
wat2wasm tiny.wat
```

and then paste this into a `tiny.yaml` that [I lifted pretty much verbatim][tetrate-yaml] from [Tetrate's excellent workshop][tetrate-workshop].

```yaml
static_resources:
  listeners:
    - name: main
      address:
        socket_address:
          address: 0.0.0.0
          port_value: 18000
      filter_chains:
        - filters:
            - name: envoy.http_connection_manager
              typed_config:
                "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                stat_prefix: ingress_http
                codec_type: auto
                route_config:
                  name: local_route
                  virtual_hosts:
                    - name: local_service
                      domains:
                        - "*"
                      routes:
                        - match:
                            prefix: "/"
                          direct_response:
                            status: 200
                            body:
                              inline_string: "hello world\n"
                http_filters:
                  - name: envoy.filters.http.wasm
                    typed_config:
                      "@type": type.googleapis.com/udpa.type.v1.TypedStruct
                      type_url: type.googleapis.com/envoy.extensions.filters.http.wasm.v3.Wasm
                      value:
                        config:
                          vm_config:
                            vm_id: "my_vm"
                            runtime: "envoy.wasm.runtime.v8"
                            code:
                              local:
                                filename: "tiny.wasm"
                  - name: envoy.filters.http.router
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```

Then start your Envoy proxy using `func-e` (again, courtesy of [Tetrate's Workshop][tetrate-workshop]. No, seriously, watch it).

```
$ func-e run -c envoy.yaml &
$ curl localhost:18000
hello world
```

Congratulations, your extensions does nothing. No really, the `"hello world\n"` is configured in `tiny.yaml`. So, yeah, all your extension does is loading without crashing the VM. Aren't you proud?

Don't you believe me? Why, let's try implementing [another callback in the spec][proxy-wasm-spec]; for instance `proxy_on_vm_start`. This function takes two parameters (which we are not going to use) and returns an `i32` representing a boolean value: `0` means `false`, i.e. a failure to initialize.

```clj
(module
    (func $proxy_abi_version_0_2_0 
    nop)
    (func $proxy_on_memory_allocate (param $memory_size i32) (result i32)
    i32.const 0 ;; do not allocate any memory
    return)
    (func $proxy_on_vm_start (param $root_context_id i32) ($vm_configuration_size i32) (result i32)
    i32.const 0
    return)

    (export "proxy_abi_version_0_2_0" (func $proxy_abi_version_0_2_0))
    (export "proxy_on_memory_allocate" (func $proxy_on_memory_allocate))
    (export "proxy_on_vm_start" (func $proxy_on_vm_start))
)
```

Let's rebuild the `wat` file with `wasm2wat` and reload using `func-e` and you'll see that your Envoy proxy indeed will fail to start while attempting to load your WASM filter:

```
[2022-04-23 17:17:55.382][6666463][error][wasm] [source/extensions/common/wasm/wasm.cc:109] Wasm VM failed Failed to start base Wasm
[2022-04-23 17:17:55.383][6666463][critical][wasm] [source/extensions/common/wasm/wasm.cc:471] Plugin configured to fail closed failed to load
[2022-04-23 17:17:55.383][6666463][critical][main] [source/server/server.cc:117] error initializing configuration 'tiny.yaml': Unable to create Wasm HTTP filter
[2022-04-23 17:17:55.384][6666463][info][main] [source/server/server.cc:925] exiting
Unable to create Wasm HTTP filter
error: envoy exited with status: 1
```

but turn the boolean result into a `1`:

```clj
    ...
    (func $proxy_on_vm_start (param $root_context_id i32) ($vm_configuration_size i32) (result i32)
    i32.const 0
    return)
    ...
```

and rejoyce! Your WASM filter is back to load successfully, although it still cannot do anything useful. Well, we all have to start somewhere.

[envoy-wasm]: https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/wasm-cc
[proxy-sdk]: https://github.com/proxy-wasm/spec
[wabt]: https://github.com/WebAssembly/wabt
[proxy-wasm-spec]: https://github.com/proxy-wasm/spec/tree/master/abi-versions/vNEXT
[tetrate-yaml]: https://github.com/tetratelabs/proxy-wasm-go-sdk/blob/main/examples/helloworld/envoy.yaml
[tetrate-workshop]: https://www.youtube.com/watch?v=KbbZBMYI58Y
