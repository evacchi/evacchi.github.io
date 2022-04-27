---
title:  'Writing Envoy WASM Filters The Hardest Way (Part 2): LLVM IR'
subtitle: ''
categories: [Envoy, WASM, LLVM]
date:   2022-04-28
---

Welcome to the next installment of the "Writing Envoy WASM Filters The Hardest Way" franchise. In this episode,
our silly hero has decided to put together the lessons learned in the [previous][previous] [episodes][episodes] and define an 
[Envoy WASM filter][envoy-wasm] using only LLVM IR, [a magnetized needle, and a steady hand][xkcd378].

Now that we have figured out [how to write a WASM filter from scratch][episodes],
the jump to LLVM IR is far smaller. We can write a source file `llvm.ll` 
that maps onto the correct WASM ABI pretty easily. 

```c
target triple = "wasm32-unknown-wasi"

declare i32 @putchar(i32) #1

;; prints "hello Envoy\n"
define i32 @main() #0 {
  call i32 (i32) @putchar(i32 104)
  call i32 (i32) @putchar(i32 101)
  call i32 (i32) @putchar(i32 108)
  call i32 (i32) @putchar(i32 108)
  call i32 (i32) @putchar(i32 111)
  call i32 (i32) @putchar(i32 32)
  call i32 (i32) @putchar(i32 69)
  call i32 (i32) @putchar(i32 110)
  call i32 (i32) @putchar(i32 118)
  call i32 (i32) @putchar(i32 111)
  call i32 (i32) @putchar(i32 121)
  call i32 (i32) @putchar(i32 10)
  ret i32 0
}

define void @proxy_abi_version_0_2_0() "wasm-export-name"="proxy_abi_version_0_2_0" {
    ret void
}
define i32 @proxy_on_memory_allocate(i32 %x) "wasm-export-name"="proxy_on_memory_allocate" {
  ret i32 0
}
```

Please notice how we are decorating our functions with `"wasm-export-name"="<exported-name>"`:
this is necessary to ensure that the WASM binary contains the corresponding `(export ...)` section. 

You will also have noticed that this code, as compared to the [WAT version][episodes] is also defining
a `@main()` routine. `@main()` will be exported automatically to [`_start`][_start]: how convenient!

In this [`_start`][_start] routine I am using the `@putchar` function as in 
[my first post in this series][previous], so this time we will print `"hello Envoy\n"` to the console.
However, in order to do that, we have to link our WASM binary to the C standard library provided by the
[WASI][previous] spec. Luckily, we have already learned how to do that.

Assuming you have followed [the setup described in my previous post][previous], 
you can compile your LLVM IR source using `clang`:

```sh
    clang llvm.ll --target=wasm32-unknown-wasi --sysroot=$WASI_SYSROOT -lc \
        $PWD/lib/wasi/libclang_rt.builtins-wasm32.a -o llvm.wasm
```

or, if you know what you are doing, using `llc` and `wasm-ld` directly 
(not recommended, the command line will evolve and change in the future):

```sh
    llc -march=wasm32 -filetype=obj llvm.ll
    wasm-ld -m wasm32 -L$WASI_SYSROOT/lib/wasm32-wasi \
        $WASI_SYSROOT/lib/wasm32-wasi/crt1.o llvm.o -lc \
        $PWD/lib/wasi/libclang_rt.builtins-wasm32.a -o llvm.wasm
```

Now, write your `llvm.yaml` as such:

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
                                filename: "envoy.wasm"
                  - name: envoy.filters.http.router
admin:
  access_log_path: "/dev/null"
  address:
    socket_address:
      address: 0.0.0.0
      port_value: 8001
```

and [load it using `func-e` as described in my last post][episodes]:


```sh
    func-e run -c envoy.yaml &
    curl localhost:18000
    hello world
```

This time you will notice that the console will log:

```
[...][6964812][info][wasm] [source/extensions/common/wasm/context.cc:1172] wasm log: hello Envoy
[...][6964812][info][wasm] [source/extensions/common/wasm/context.cc:1172] wasm log: hello Envoy
[...][6964812][info][wasm] [source/extensions/common/wasm/context.cc:1172] wasm log: hello Envoy
```

Congratulations: your WASM filter is still pretty useless, but at least it's polite!

[previous]: /llvm/wasm/wasi/2022/04/14/compiling-llvm-ir-into-wasm.html
[episodes]: /envoy/wasm/wat/2022/04/23/envoy-wasm-filters-the-hardest-way-using-wat.html
[envoy-wasm]: https://www.envoyproxy.io/docs/envoy/latest/start/sandboxes/wasm-cc
[xkcd378]: https://xkcd.com/378/
[_start]: https://github.com/proxy-wasm/spec/tree/master/abi-versions/vNEXT#_start

