# iOS dynamic analysis &amp; reverse engineering

**Independent security researcher.** Field notes from building a dynamic-analysis toolchain for iOS (arm64) — hooking, capturing, and reproducing app behavior on my own jailbroken devices.

Most of these are the kind of problem you only hit once you try to instrument a *real*, hardened app at full speed without freezing it: where the trampoline itself is the bottleneck, where the JIT miscompiles your hook, where the right bytes are one context-switch too far away, where the app kills itself the moment it feels a breakpoint.

## Writeups

### Instrumentation — performance &amp; failure modes
*Why hooks freeze a live app, and what to do about it.*

- **[A Frida Interceptor on a hot function can freeze the app by itself](https://gist.github.com/wenroudeyu-collab/5cfb2105130be01a91d80a7453dc766e)** — the trampoline + lock overhead alone stalls the main thread, independent of what the callback does. A lightweight probe-gate measures the call rate first and skips the layer when the target is calling it too hot.
- **[Enabling many cheap hooks at once can freeze an app — even when every callback is a no-op](https://gist.github.com/wenroudeyu-collab/3168a7adc72178af0649ba445c80f4f7)** — aggregate inline-hook trampoline overhead: each hook is fine in isolation, the sum starves the runloop.
- **[Out-of-process kernel hooks: throughput ceiling, freeze profiles, and a saturation fuse](https://gist.github.com/wenroudeyu-collab/4b4acf733230bae875599519697157d1)** — a control experiment (in-process Frida vs. an out-of-process HW/BRK kernel hook) that disproved my own assumption. The out-of-process hook caps at ~27–330 hits/s and *must* abandon hot functions, so its floodguard keys on saturation **time**, not rate.

### Correctness &amp; capture
*Getting the right bytes, and proving they're right.*

- **[A TinyCC/arm64 codegen bug that corrupts return addresses in CModule hooks](https://gist.github.com/wenroudeyu-collab/69b4dc0df4939978edf29a2b635edb40)** — a wild write from internal-static addressing in JIT'd hook code, chased down to a SIGBUS on `RET` and fixed by moving all mutable state to `extern` + allocated pointers.
- **[Verifying hooks on a stripped native crypto library with a self-built oracle](https://gist.github.com/wenroudeyu-collab/cb3152478194019dc978a0fb1565cead)** — cross-compile the same library, `dlopen` it, and drive known-answer tests, to prove a symbol-less hook reads the right arguments before trusting it on live traffic.
- **[Capturing SQLite result rows from a live app at full speed](https://gist.github.com/wenroudeyu-collab/767fbdedf66ba374a8bc56aa50cc7109)** — a C-only `sqlite3_step` hook that reads typed columns inside the callback, preserves 64-bit precision (raw bytes + BigInt), and never takes the JS lock.
- **[Decrypting an iOS app's HTTP/3 in Wireshark without putting anything inside the app](https://gist.github.com/wenroudeyu-collab/48c9bc126b56a9a1c217c638f09ac596)** — a frida-free QUIC capture: hardware-breakpoint the shared-cache `sendto`/`recvfrom` for the ciphertext and the keylog derivation point for the TLS secrets, then synthesize one pcapng whose embedded Decryption Secrets Block makes Wireshark decrypt it on open.

### Stealth &amp; anti-detection
*Instrumenting an app that's actively looking for you.*

- **[It wasn't the injection: what makes a hardened iOS app self-destruct at launch](https://gist.github.com/wenroudeyu-collab/aa197a3983f3a63387edeb430e9c1251)** — a controlled experiment: spawn the app suspended, scrub the injected dylib from its memory so it runs clean, then toggle a single hardware breakpoint. The breakpoint alone flips survival — so the tell isn't the injection, it's a self-readable `ARM_DEBUG_STATE64`. A one-shot startup scan and a continuous new-thread sweep turn out to need two different evasions (delay the arm; don't arm new threads).

## Focus

Frida internals · CModule / TinyCC codegen · out-of-process kernel hooks (HW/BRK, Mach exception ports) · iOS crypto &amp; TLS instrumentation · capture → analysis → replay pipelines · anti-debug / anti-tamper detection surfaces.

---

*All research is performed on my own devices and on apps I install myself — dynamic analysis and reproduction for defensive / dual-use purposes.*
