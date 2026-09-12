---
title: "Tricolor GC in Go: How the Marker Keeps Live Objects Alive — and What Happens Without the Write Barrier"
date: 2026-09-12
draft: false
tags: ["go", "golang", "gc", "runtime", "memory", "write-barrier", "internals"]
categories: ["go", "engineering"]
summary: "Tricolor marking is not a garbage collection algorithm so much as a coexistence protocol between the collector and a program that keeps rewriting the object graph mid-traversal. I walk through the invariant, the write barrier (including Go's hybrid barrier from 1.8), allocate-black and mark assist — and hand you an interactive sandbox where you can switch the barrier off and watch the GC free a live object."
ShowToc: true
series: ["Go Internals"]
---

Tricolor marking is usually explained like this: white means not found, grey means queued, black means scanned. All correct, and about equally useless, because the real question never gets asked: why three colors at all, when graph traversal needs exactly two — visited and not visited?

Here is why. The third color exists for one reason only: the program does not stop while the collection runs. This post is about what follows from that. At the end there is an interactive sandbox where you can switch the write barrier off and watch the GC eat a live object.

## Why a third color

A naive collector does mark & sweep in one <abbr title="Stop-the-world — a pause during which the runtime halts every goroutine">STW</abbr> pause: stop the world, walk the graph from the roots, free everything you did not reach. Correct, simple — and the pause is linear in the size of the live heap. With 10 GB of live data that is seconds, not microseconds.

Go took a different route back in 1.5: marking runs **concurrently** with the program ([Go 1.5 GC: prioritizing low latency and simplicity](https://go.dev/blog/go15gc)). The world stops twice and briefly — to turn the barrier on and scan the roots, and to terminate marking. Since Go 1.8 those pauses are "usually under 100 microseconds and often as low as 10" ([Go 1.8 release notes](https://go.dev/doc/go1.8#gc)) — and, crucially, they no longer scale with heap size.

But concurrency creates a problem that simply does not exist in an STW collector. While the collector walks the graph, the **mutator** (the term comes from [Dijkstra et al., 1978](https://dl.acm.org/doi/10.1145/359642.359655) — the program that mutates the object graph during traversal) rewrites pointers. And two colors are no longer enough.

Picture this: the collector has fully scanned object `B` — read all its fields, it will never come back to it. At that moment a goroutine runs `B.next = W`, where `W` is a white object that has not been found yet, and simultaneously clears the only other reference to `W`. Formally `W` is alive: it is reachable from the roots through `B`. In practice the collector will never find it — it has already been inside `B`.

That is the classic lost object. Three colors exist to catch it: "scanned" (black) and "found but not scanned" (grey) are fundamentally different states, and the correctness rule is stated in terms of exactly that difference.

<figure class="svg-diagram">
<svg viewBox="0 0 760 250" role="img" aria-label="Three stages of losing a live object: the marker finishes B, the mutator moves the pointer to W behind the black object, the sweep frees the live W">
  <defs>
    <marker id="en-ar" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0 0L10 5L0 10z" fill="#8b90a0"/>
    </marker>
    <marker id="en-ar-a" viewBox="0 0 10 10" refX="9" refY="5" markerWidth="6" markerHeight="6" orient="auto-start-reverse">
      <path d="M0 0L10 5L0 10z" fill="#fbbf24"/>
    </marker>
  </defs>

  <!-- panel 1 -->
  <rect x="8" y="34" width="236" height="200" rx="8" fill="none" stroke="#262a31"/>
  <text x="20" y="24" fill="#8b90a0" font-size="12" font-family="var(--font-mono,'JetBrains Mono'),monospace">1 · marker has scanned B</text>
  <circle cx="70" cy="88" r="22" fill="#e5e7eb" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="70" y="93" fill="#0d0e12" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">B</text>
  <text x="70" y="128" fill="#8b90a0" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">black</text>
  <circle cx="70" cy="180" r="22" fill="none" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="70" y="185" fill="#e5e7eb" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">X</text>
  <circle cx="188" cy="180" r="22" fill="none" stroke="#e5e7eb" stroke-width="1.5" stroke-dasharray="4 3"/>
  <text x="188" y="185" fill="#e5e7eb" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">W</text>
  <text x="188" y="220" fill="#8b90a0" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">white</text>
  <line x1="94" y1="180" x2="162" y2="180" stroke="#8b90a0" stroke-width="1.6" marker-end="url(#en-ar)"/>

  <!-- panel 2 -->
  <rect x="262" y="34" width="236" height="200" rx="8" fill="none" stroke="#fbbf24" stroke-opacity="0.5"/>
  <text x="274" y="24" fill="#fbbf24" font-size="12" font-family="var(--font-mono,'JetBrains Mono'),monospace">2 · mutator: *B = W; X = nil</text>
  <circle cx="324" cy="88" r="22" fill="#e5e7eb" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="324" y="93" fill="#0d0e12" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">B</text>
  <circle cx="324" cy="180" r="22" fill="none" stroke="#8b90a0" stroke-width="1.5" stroke-opacity="0.35"/>
  <text x="324" y="185" fill="#8b90a0" font-size="14" font-weight="700" text-anchor="middle" fill-opacity="0.45" font-family="var(--font-mono,'JetBrains Mono'),monospace">X</text>
  <circle cx="442" cy="180" r="22" fill="none" stroke="#e5e7eb" stroke-width="1.5" stroke-dasharray="4 3"/>
  <text x="442" y="185" fill="#e5e7eb" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">W</text>
  <line x1="336" y1="108" x2="430" y2="158" stroke="#fbbf24" stroke-width="1.8" marker-end="url(#en-ar-a)"/>
  <line x1="348" y1="180" x2="416" y2="180" stroke="#8b90a0" stroke-width="1.4" stroke-opacity="0.3" stroke-dasharray="3 4"/>
  <text x="380" y="136" fill="#fbbf24" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">white behind black</text>

  <!-- panel 3 -->
  <rect x="516" y="34" width="236" height="200" rx="8" fill="none" stroke="#f87171" stroke-opacity="0.5"/>
  <text x="528" y="24" fill="#f87171" font-size="12" font-family="var(--font-mono,'JetBrains Mono'),monospace">3 · sweep</text>
  <circle cx="578" cy="88" r="22" fill="#e5e7eb" stroke="#8b90a0" stroke-width="1.5"/>
  <text x="578" y="93" fill="#0d0e12" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">B</text>
  <circle cx="696" cy="180" r="22" fill="none" stroke="#f87171" stroke-width="1.6" stroke-dasharray="5 4"/>
  <text x="696" y="185" fill="#f87171" font-size="14" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">W</text>
  <text x="696" y="220" fill="#f87171" font-size="10.5" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">freed · still live</text>
  <line x1="590" y1="108" x2="684" y2="158" stroke="#f87171" stroke-width="1.6" stroke-opacity="0.5" stroke-dasharray="4 4"/>
</svg>
<figcaption>The marker never revisits black objects — so a pointer that appears inside a black object after it was scanned has to be caught separately</figcaption>
</figure>

## The invariant and the barrier

The rule that holds the whole construction together comes in two flavors.

**Strong invariant**: a black object never holds a pointer to a white one. **Weak invariant**: a black object may point to a white one, but only if that white object is also reachable through a chain from some grey object. The weak form admits more states and is therefore cheaper to maintain — and it is the one Go relies on.

The invariant has to be enforced at the moment it can be broken, which is when a pointer is written into the heap. Hence the **write barrier** — a small piece of code the compiler injects before every such write. In the textbook (Dijkstra) form it does exactly one thing: if the value being written is white, shade it grey, i.e. push it onto the marker's queue. A white object hidden behind a black one immediately stops being white.

Real Go has used a **hybrid barrier** since 1.8 — a combination of Yuasa's deletion barrier and Dijkstra's insertion barrier ([proposal 17503, Clements & Hudson](https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md)). The pseudocode comes straight from the comment in [`runtime/mbarrier.go`](https://github.com/golang/go/blob/master/src/runtime/mbarrier.go):

```go
writePointer(slot, ptr):
    shade(*slot)
    if current stack is grey:
        shade(ptr)
    *slot = ptr
```

That first line is the difference from the textbook version: it shades the **old** value of the slot, not just the new one. The point is to give the collector a snapshot of the graph as of the start of the cycle — a pointer the mutator tears down still ends up in the queue. This is what let Go drop the final stack re-scan under STW, the nastiest pause in the pre-1.8 design, which grew linearly with the number of goroutines.

The barrier costs something on every pointer write, which is why Go only enables it for the duration of a cycle — outside marking it is off and the code takes the fast path.

Two consequences follow that textbooks tend to mention in passing, and that are exactly what makes the scheme work in practice.

**Objects allocated during a cycle are born black.** Not grey — black. The logic is simple: the object was just created, there is nothing inside it to scan yet, and shading it grey would have the marker forever chasing an allocating goroutine. The price is that such objects are guaranteed to survive to the next cycle even if they are garbage. That is a deliberate trade: floating garbage in exchange for termination.

**Mark assist.** A goroutine that allocates must itself do marking work proportional to what it allocated. Without it, a single allocating goroutine easily outruns the background markers, the heap grows faster than it is scanned, and the cycle never converges. Assist is backpressure built into the allocator; the proportions live in the [pacer](https://github.com/golang/proposal/blob/master/design/44167-gc-pacer-redesign.md), rewritten in Go 1.18.

This is why "the GC is eating CPU" often shows up in practice as "allocating goroutines got slow": the time is spent in assists, not in background workers.

<figure class="svg-diagram">
<svg viewBox="0 0 760 130" role="img" aria-label="Phases of a Go GC cycle: short STW, concurrent mark, short STW, concurrent sweep">
  <rect x="8" y="34" width="96" height="44" rx="6" fill="#f87171" fill-opacity="0.16" stroke="#f87171"/>
  <text x="56" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">STW</text>
  <text x="56" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">barrier on,</text>
  <text x="56" y="110" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">roots</text>

  <rect x="112" y="34" width="308" height="44" rx="6" fill="#4ade80" fill-opacity="0.14" stroke="#4ade80"/>
  <text x="266" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">concurrent mark</text>
  <text x="266" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">grey → black · program keeps running · barrier catches writes · mark assist</text>

  <rect x="428" y="34" width="84" height="44" rx="6" fill="#f87171" fill-opacity="0.16" stroke="#f87171"/>
  <text x="470" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">STW</text>
  <text x="470" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">termination</text>

  <rect x="520" y="34" width="232" height="44" rx="6" fill="#60a5fa" fill-opacity="0.14" stroke="#60a5fa"/>
  <text x="636" y="60" fill="#e5e7eb" font-size="11.5" font-weight="700" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">concurrent sweep</text>
  <text x="636" y="96" fill="#8b90a0" font-size="10" text-anchor="middle" font-family="var(--font-mono,'JetBrains Mono'),monospace">whatever stayed white is garbage</text>

  <text x="8" y="22" fill="#8b90a0" font-size="10.5" font-family="var(--font-mono,'JetBrains Mono'),monospace">block widths are illustrative: STW pauses are tens of microseconds, marking is milliseconds and up</text>
</svg>
<figcaption>One GC cycle in Go: the world stops twice and briefly, all the heavy lifting runs alongside the program</figcaption>
</figure>

One more property worth stating out loud: tricolor marking happily collects **garbage cycles**. An `A → B → A` pair unreachable from the roots stays white and gets freed — no special-case code required. That is the fundamental advantage over reference counting, where a cyclic pair lives forever.

## The sandbox

Below is an interactive model of a single marking cycle. On the left, the root set: goroutine stacks and globals. On the right, the heap: objects with references you can drag, link and unlink.

{{< sandbox src="sandbox/tricolor-gc.html" height="900" title="Tricolor GC in Go — interactive sandbox" link="Open the sandbox in a new tab" caption="Click two objects to link them; click an arrow to drop the pointer" >}}

What to try, in order:

1. **Hit "Step"** and watch the "Grey set" panel. That is the marker's queue. Each step: pop an object, read its pointers, shade white targets grey, turn the object itself black. The cycle ends when the queue is empty.
2. **Switch the mutator on.** Now pointer writes and allocations happen alongside marking. Notice that new objects show up black right away — that is allocate-black — and watch the log for barrier hits.
3. **Switch the write barrier off and press "Experiment: lose an object".** The mutator moves a white object behind a black one and clears the original reference. The marker does not revisit black — and on the sweep phase a live object gets freed. Exactly the bug the barrier prevents.
4. Switch the barrier back on and repeat the experiment. The write shades the white object grey, a `[barrier]` line appears in the log, and the object survives the cycle.
5. Run the first cycle to completion and find the `F ⇄ G` pair — a garbage cycle that gets freed with no special-case code.

One honest caveat about the model: the barrier in the sandbox is the classic Dijkstra insertion barrier, "black got a pointer to white → shade the white grey". Real Go, as noted above, uses the hybrid barrier and also shades the slot's old value. For seeing *why* a barrier is needed at all, the insertion variant is plenty; for reading `mbarrier.go`, it is not.

## What changed with Green Tea

Everything above — the invariant and the barrier — still holds, but the heap traversal itself works differently in recent Go. Go 1.25 shipped the experimental Green Tea collector ([issue #73581](https://github.com/golang/go/issues/73581)), and in Go 1.26 it is [on by default](https://go.dev/doc/go1.26); the old design lives behind `GOEXPERIMENT=nogreenteagc` and is expected to be removed in 1.27.

The idea: the classic marker chases individual pointers scattered across memory, and on small objects that destroys cache locality — every queue step is a near-guaranteed miss. Green Tea works with **pages** rather than individual objects: what turns grey is a span, and the marker processes it whole, sequentially in memory. The quoted win is a 10–40% reduction in GC overhead on real programs, plus roughly another 10% on recent amd64 (Ice Lake / Zen 4 and newer) thanks to vector instructions over the page bitmaps.

What matters for this article: the **unit of marking work** changed, not the protocol. Three colors, the invariant and the write barrier are all still there — it is still concurrent tricolor marking, the queue just holds pages now instead of objects.

## References

- [Dijkstra, Lamport, Martin, Scholten, Steffens. On-the-Fly Garbage Collection: An Exercise in Cooperation, CACM 21(11), 1978](https://dl.acm.org/doi/10.1145/359642.359655) — the original tricolor abstraction
- [Go proposal 17503: Eliminate STW stack re-scanning](https://github.com/golang/proposal/blob/master/design/17503-eliminate-rescan.md) — the hybrid barrier, Go 1.8
- [`runtime/mbarrier.go`](https://github.com/golang/go/blob/master/src/runtime/mbarrier.go) — the comments in this file beat most GC articles
- [A Guide to the Go Garbage Collector](https://go.dev/doc/gc-guide) — the official guide: GOGC, GOMEMLIMIT, latency
- [Go 1.26 release notes](https://go.dev/doc/go1.26) — Green Tea on by default
