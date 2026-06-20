---
title: "Go Internals: sync.Mutex and sync/atomic — From Data Race to Starvation Mode"
date: 2026-06-18
draft: false
tags: ["go", "golang", "concurrency", "mutex", "atomic", "cas", "sync", "runtime"]
categories: ["go", "engineering"]
summary: "Mutex and atomic are not two independent tools — they are one stack of abstractions. I dig through the stack from data race to starvation mode: why x++ is three ARM64 instructions, how CAS became a universal primitive (Herlihy 1991), why Mutex is built on top of atomic CAS, and how Russ Cox's barging-vs-handoff debate (issue #13086) gave Go its 1ms starvation threshold. With public benchmarks, the bit layout of Mutex state, and a cheatsheet for when to reach for which."
ShowToc: true
series: ["Go Internals"]
---

When you write `mu.Lock()` in Go, you probably think "lock." When you write `atomic.AddInt64(&x, 1)`, you think "atomic add." Two different tools, two different mental models, two different sections of the standard library documentation. But here is the strange thing: they are the same thing. More precisely, they are one stack of abstractions, and each layer is built on top of the one below.

In this article I will dig through that stack, from the bottom to the top and back. We will start with a broken `counter++`, climb down through CPU instructions, climb back up through CAS (compare-and-swap), atomic operations, futex, and finally arrive at `sync.Mutex` — and discover that it is mostly an optimization layered on top of atomic CAS, plus a fairness mechanism invented to fix a real production bug from 2015.

## The Stack

Before diving in, here is the picture I want you to keep in your head. Five layers, each one built on the previous:

{{< mermaid align="center" >}}
graph TB
    L5["Level 5: When to use what<br>Benchmarks, cheatsheet, pitfalls"]
    L4["Level 4: sync.Mutex<br>state, sema, spinning, starvation"]
    L3["Level 3: Futex<br>userspace fast path + kernel slow path"]
    L2["Level 2: Atomic operations<br>CAS, Load, Store, Add — sync/atomic"]
    L1["Level 1: Hardware<br>LOCK CMPXCHG (x86), LDADD/CAS (ARM LSE)"]
    L0["Level 0: The Problem<br>data race, x++ is 3 instructions"]

    L0 --> L1 --> L2 --> L3 --> L4 --> L5

    style L0 fill:#c62828,color:#fff
    style L1 fill:#ad1457,color:#fff
    style L2 fill:#6a1b9a,color:#fff
    style L3 fill:#4527a0,color:#fff
    style L4 fill:#283593,color:#fff
    style L5 fill:#1565c0,color:#fff
{{< /mermaid >}}

The claim of this article is that you cannot really understand Level 4 (`sync.Mutex`) without understanding Level 2 (atomic), and you cannot understand Level 2 without at least a glimpse of Level 1 (hardware). Conversely, the only reason to care about Level 1 is that it produces the practical recommendations of Level 5.

Let's start at the bottom.

## 1. The Problem: Why `x++` Breaks

Here is the simplest possible data race. A thousand goroutines increment a shared counter:

```go
var counter int64
var wg sync.WaitGroup

func main() {
    for i := 0; i < 1000; i++ {
        wg.Add(1)
        go func() {
            defer wg.Done()
            counter++
        }()
    }
    wg.Wait()
    fmt.Println(counter)
}
```

Run it. The answer is not 1000. It is something like 987, or 993, or 971. Sometimes it is exactly 1000, and that is worse — it means the bug is hiding.

Why? Because `counter++` is not one operation. It looks like one operation in Go source, but the compiler translates it to three. Here is the actual ARM64 assembly Go produces for `counter++`:

```asm
MOVD  main.counter(SB), R0    ; load counter from memory into register R0
ADD   $1, R0, R0              ; add 1 to R0
MOVD  R0, main.counter(SB)    ; store R0 back to memory
```

Three instructions: load, modify, store. Each is atomic on its own, but the sequence is not. Between the load and the store, another goroutine can sneak in.

Here is the timeline when two goroutines collide:

{{< mermaid align="center" >}}
sequenceDiagram
    participant G1 as Goroutine 1
    participant Mem as Memory (counter=5)
    participant G2 as Goroutine 2

    G1->>Mem: MOVD counter → R0 (R0=5)
    Note over G1: about to ADD
    G2->>Mem: MOVD counter → R0 (R0=5)
    G1->>G1: ADD → R0=6
    G1->>Mem: MOVD R0 → counter (counter=6)
    G2->>G2: ADD → R0=6
    G2->>Mem: MOVD R0 → counter (counter=6)
    Note over Mem: One increment lost
{{< /mermaid >}}

Both goroutines read `5`, both compute `6`, both store `6`. We did two increments, the counter went up by one. Multiply by a thousand goroutines and you get 987.

This is a **data race**, and Go's memory model ([go.dev/ref/mem](https://go.dev/ref/mem)) defines it formally. A read-write data race on a memory location `x` consists of a read-like operation `r` and a write-like operation `w` on `x`, at least one of which is non-synchronizing, such that neither happens before the other. Read that twice: the definition is about ordering, not timing. Two operations on the same location, with no synchronization between them, is a race — even if in practice they almost never overlap.

### 1.1. DRF-SC: the promise and the price

The Go memory model makes one big promise, called **DRF-SC**: data-race-free programs execute in a sequentially consistent manner. If your program has no data races, you can reason about it as if all goroutines were multiplexed onto a single processor, taking turns. You do not have to worry about compiler reordering, store buffers, cache effects, or any of the other hardware-level weirdness that plagues C++ programmers.

The price is that you must actually eliminate the data races. Go's race detector (`go test -race`, built on [ThreadSanitizer](https://github.com/google/sanitizers/wiki/ThreadSanitizerGoManual)) will catch them at runtime, but you have to write race-free code in the first place. And to do that, you need synchronization.

The formal model behind DRF-SC is the same one used by C++, Java, JavaScript, Rust, and Swift. It comes from Hans-J. Boehm and Sarita V. Adve's paper "[Foundations of the C++ Concurrency Memory Model](https://dl.acm.org/doi/10.1145/1375581.1375591)" (PLDI 2008). Go aligned with this model in the June 2022 memory model revision, which shipped with Go 1.19.

### 1.2. Happens-before: the relation that makes everything work

The formal definition of data race depends on a relation called **happens-before**. Informally, an operation A happens-before an operation B if you can prove A must have completed before B started. The relation is the transitive closure of two things:

- **Sequenced-before**: within a single goroutine, statements execute in source order. Statement 5 is sequenced before statement 6.
- **Synchronized-before**: across goroutines, certain operations create synchronization edges. `ch <- x` happens-before the corresponding `<-ch`. `mu.Unlock()` happens-before the next `mu.Lock()`. An atomic store happens-before the atomic load that observes the stored value.

If A happens-before B, then anything A wrote to memory is visible to B. If neither happens-before the other, you have a race.

This is why `counter++` is broken: nothing in the program establishes a happens-before relation between the increments of different goroutines. They are unordered, so the read in goroutine A can overlap the write in goroutine B.

To fix the race, we need to add synchronization. There are two main paths:

1. Make the increment itself atomic — a single operation that cannot be interrupted.
2. Wrap the increment in a mutex, so only one goroutine can execute it at a time.

These are Levels 2 and 4 in our stack. Let's take them in order.

## 2. The First Path: Atomic Operations

The cleanest fix for `counter++` is to make it atomic:

```go
var counter int64
// ...
atomic.AddInt64(&counter, 1)
```

`atomic.AddInt64` performs the load, add, and store as a single indivisible operation. No other goroutine can observe an intermediate state, no other goroutine can sneak in between the load and the store, because there is no "between."

How is that possible? Because underneath, `atomic.AddInt64` compiles to a single CPU instruction.

### 2.1. CAS: the universal primitive

The fundamental building block of all atomic operations is **CAS** — Compare-And-Swap. Its signature, in pseudocode:

```
CAS(addr, expected, new) → bool
    if *addr == expected:
        *addr = new
        return true
    else:
        return false
```

The whole thing is atomic: the comparison and the store happen as one indivisible step. If `*addr` equals `expected`, the new value is written and CAS returns true. Otherwise, nothing is written and CAS returns false.

CAS is special. In 1991, Maurice Herlihy published a paper called "[Wait-Free Synchronization](https://cs.brown.edu/people/mph/Herlihy91/p124-herlihy.pdf)" in ACM Transactions on Programming Languages and Systems. He classified synchronization primitives by their **consensus number** — the maximum number of threads for which the primitive can solve the consensus problem (all threads agree on a single value). The hierarchy is brutal:

| Primitive | Consensus number |
|-----------|------------------|
| Read/write registers | 1 |
| Test-and-set | 2 |
| Fetch-and-add | 2 |
| Queue, stack | 2 |
| **CAS** | **∞** |

CAS has consensus number ∞. That makes it a **universal primitive**: any wait-free or lock-free data structure can be built on top of CAS. The others cannot. Test-and-set can solve consensus for 2 threads, fetch-and-add for 2, but only CAS scales to arbitrarily many. This is why every modern architecture provides CAS in hardware, and why Go builds everything else on top of it.

### 2.2. CAS-loop: how Add is actually implemented

You might wonder: if CAS is the universal primitive, how do you build `Add` out of it? You cannot CAS-and-add in one shot, because CAS only writes a specific new value, not "old plus one." The answer is a loop:

```go
func addInt64(addr *int64, delta int64) int64 {
    for {
        old := atomic.LoadInt64(addr)
        new := old + delta
        if atomic.CompareAndSwapInt64(addr, old, new) {
            return new
        }
        // CAS failed — someone else wrote between our load and our CAS.
        // Loop, re-read the current value, try again.
    }
}
```

This is called a **CAS-loop** or **compare-and-swap retry loop**. It loads the current value, computes the new value, attempts to CAS. If the CAS fails, that means another goroutine modified the memory between our load and our CAS, so we retry with the new current value.

{{< mermaid align="center" >}}
flowchart TD
    Start([Add delta to addr]) --> Load["old = *addr"]
    Load --> Compute["new = old + delta"]
    Compute --> CAS{"CAS(addr, old, new)"}
    CAS -->|success| Done([done])
    CAS -->|failure — someone wrote| Load

    style Done fill:#2e7d32,color:#fff
    style CAS fill:#1565c0,color:#fff
{{< /mermaid >}}

This is what makes CAS universal. Out of CAS, you can build Add, Subtract, Exchange, Min, Max, any operation you want — all of them become atomic by retrying until CAS succeeds.

There is a price. Under heavy contention (many goroutines all trying to update the same address), CAS-loops can waste CPU. Every failed CAS is a wasted attempt, and the loop spins until it wins. We will come back to this in section 4.

### 2.3. Hardware: LOCK CMPXCHG and ARM LSE

When you write `atomic.CompareAndSwapInt64`, the Go compiler emits a single CPU instruction. On x86, that instruction is `LOCK CMPXCHG`:

```asm
lock cmpxchg [addr], new
```

The `LOCK` prefix is important. In early x86 CPUs, `LOCK` literally asserted the `LOCK#` hardware pin, which froze the entire memory bus for the duration of the instruction. That was catastrophically expensive. In modern x86 CPUs (Pentium Pro and later), `LOCK` is much smarter: if the target address fits in a single cache line, the CPU uses **cache line locking** — it invalidates that one cache line on all other cores and performs the operation without touching the bus. Only if the operation spans two cache lines (which should not happen with proper alignment) does it fall back to the old bus lock.

On ARM64, things evolved in two stages. Pre-LSE (Large System Extensions, pre-ARMv8.1), CAS was emulated with a pair of instructions called `LDXR`/`STXR` (exclusive load and exclusive store). The load tagged the cache line for monitoring; if no other core wrote to it before the store, the store succeeded. Otherwise it failed and you retried — exactly the CAS-loop, but in hardware.

ARMv8.1 added **LSE** (Large System Extensions), which introduced single-instruction atomics: `CAS`, `LDADD`, `LDCLR`, `STSET`, and friends. These are faster because there is no in-CPU retry loop — the operation completes in one go. Apple Silicon (M1 and later) and AWS Graviton 3 and 4 all support LSE, which is why atomic-heavy code on these chips is so much faster than the pre-LSE era.

The key takeaway: every `atomic.X` call in Go is one CPU instruction. There is no function call, no loop (unless CAS fails), no syscall. Just one instruction. This is what makes atomics fast.

### 2.4. Go's sync/atomic package

The `sync/atomic` package offers two styles of API.

**Functions** (the original form, present since Go 1):

```go
var x int64
atomic.LoadInt64(&x)
atomic.StoreInt64(&x, 42)
atomic.AddInt64(&x, 1)
atomic.SwapInt64(&x, 99)
atomic.CompareAndSwapInt64(&x, expected, new)
```

Plus equivalents for `int32`, `uint32`, `uint64`, `uintptr`, `unsafe.Pointer`, and the special-case `atomic.Value` for arbitrary types.

**Types** (introduced in Go 1.19, August 2022):

```go
var x atomic.Int64
x.Load()
x.Store(42)
x.Add(1)
x.Swap(99)
x.CompareAndSwap(expected, new)

var p atomic.Pointer[Config]   // generic, type-safe
var f atomic.Bool
var u atomic.Uint64
var flags atomic.Uint32
```

The types are newer and nicer. They wrap the same underlying CPU instructions, but with three advantages:

1. **No more `&x`** — you call methods on the value directly, which makes ownership clearer.
2. **No alignment headaches** — see the next section.
3. **Type safety for pointers** — `atomic.Pointer[T]` is generic, no more `unsafe.Pointer` casting.

The Go 1.19 release notes ([go.dev/doc/go1.19](https://go.dev/doc/go1.19)) describe these types as a direct consequence of the memory model revision: "Along with the memory model update, Go 1.19 introduces new types in the `sync/atomic` package that make it easier to use atomic values, such as atomic.Int64 and atomic.Pointer[T]." Before 1.19, on 32-bit platforms (think `GOARCH=386`, `GOARCH=arm`), an `int64` had to be 8-byte aligned for atomic operations to work — otherwise `atomic.LoadInt64` would panic at runtime. You had to manually arrange struct fields or add padding. The new `atomic.Int64` type guarantees alignment automatically, which kills a whole class of bugs.

### 2.5. Memory ordering: Go only has one mode

This is one of the biggest differences between Go and C++/Rust. C++11 introduced six memory orderings: `memory_order_relaxed`, `memory_order_consume`, `memory_order_acquire`, `memory_order_release`, `memory_order_acq_rel`, `memory_order_seq_cst`. Rust inherits the same six. The weaker orderings (`relaxed`, `acquire`, `release`) allow the compiler and CPU to reorder operations more aggressively, which can buy performance at the cost of more careful reasoning.

Go has one. **Sequentially consistent**, always. Every atomic operation in Go is a full memory barrier. There is no `atomic.LoadAcquire`, no `atomic.StoreRelease`. As [Russ Cox wrote](https://research.swtch.com/gomm) when revising the memory model for Go 1.19: Go deliberately does not provide the relaxed orderings, because they are too easy to misuse, and the performance gain is usually small.

This is the same trade-off as in Level 1: simpler model, less performance, fewer bugs. If you are coming from C++ and looking for `acquire/release` in Go, stop. Use `sync/atomic` as-is, accept the full barrier cost, and your code will be correct by default.

One subtle but important consequence: atomic operations synchronize not just the atomic variable, but everything that happened before the atomic write in the same goroutine. This is the foundation of the **publication pattern**:

```go
type Config struct {
    Hosts []string
    TTL   time.Duration
    // ... many fields
}

var configPtr atomic.Pointer[Config]

// Writer goroutine (applies config update)
func updateConfig(c *Config) {
    // All writes to c.Hosts, c.TTL, etc. happen-before this Store.
    configPtr.Store(c)
}

// Reader goroutine (millions of calls)
func getConfig() *Config {
    // Load synchronizes with the corresponding Store.
    // The returned *Config is fully initialized — no partial reads.
    return configPtr.Load()
}
```

The fields of `Config` are not atomic. They are plain `[]string`, plain `time.Duration`. But because the writer fully constructs `c` before calling `Store`, and the reader's `Load` synchronizes with that `Store`, the reader sees a fully initialized struct. No locks, no copying on the read path, no contention. This is the canonical way to do read-heavy configuration in Go.

## 3. The Second Path: Mutex, Built on Top of Atomic

Now we can finally explain `sync.Mutex`. The short version: it is mostly an optimization layered on top of atomic CAS, plus a fairness mechanism. Let's see exactly how.

### 3.1. The Mutex struct

Here is the actual definition from Go 1.24 ([source](https://cs.opensource.google/go/go/+/refs/tags/go1.24.0:src/internal/sync/mutex.go)):

```go
type Mutex struct {
    state int32
    sema  uint32
}
```

Two fields. Eight bytes total. `state` is a packed bitfield carrying four pieces of information. `sema` is a semaphore that the runtime uses to park and wake goroutines.

The `state` field is laid out as follows:

```
┌───────────────────────────────────────────────────────────────┐
│                          state (int32)                          │
├───────┬────────┬─────────────┬─────────────────────────────────┤
│ bit 0 │ bit 1  │   bit 2     │  bits 3 .. 31                   │
│       │        │             │                                 │
│Locked │ Woken  │ Starving    │  Waiter count (29 bits, max ~536M)│
└───────┴────────┴─────────────┴─────────────────────────────────┘
    1      2         4                   8, 16, 32, ...
```

The constants in the source:

```go
const (
    mutexLocked      = 1 << iota // 1
    mutexWoken                   // 2
    mutexStarving                // 4
    mutexWaiterShift = iota      // 3

    starvationThresholdNs = 1e6  // 1 millisecond
)
```

Four things encoded in 32 bits:

- **Locked** (bit 0): is the mutex currently held? 1 = yes.
- **Woken** (bit 1): has a waiter been woken up and is now trying to acquire? This is an optimization hint to `Unlock` — it knows not to wake another waiter because one is already awake.
- **Starving** (bit 2): is the mutex in starvation mode? See section 3.6.
- **Waiter count** (bits 3-31): how many goroutines are currently parked waiting for this mutex. 29 bits, so up to about 536 million waiters. You will never hit that.

The `sema` field has no internal structure. It is an opaque token that the runtime's semaphore code uses to manage a queue of parked goroutines.

### 3.2. Fast path: one CAS, no syscall

The hot path through `Lock()` is short:

```go
func (m *Mutex) Lock() {
    // Fast path: grab unlocked mutex.
    if atomic.CompareAndSwapInt32(&m.state, 0, mutexLocked) {
        // ... race detector bookkeeping ...
        return
    }
    // Slow path (outlined so that the fast path can be inlined)
    m.lockSlow()
}
```

That is it. One CAS. If the mutex is unlocked (state == 0), CAS it to `mutexLocked` (state == 1) and return. If the CAS fails, fall through to `lockSlow`.

The fast path is inlined. You can verify this yourself:

```bash
$ go build -gcflags="-m" main.go
./main.go:13:12: inlining call to sync.(*Mutex).Lock
./main.go:15:14: inlining call to sync.(*Mutex).Unlock
```

Inlining matters here. It means that in the uncontended case, calling `mu.Lock()` does not even incur a function call overhead — the CAS instruction is planted directly at the call site. This is why uncontended `mu.Lock()` is roughly 20-60 nanoseconds: it is essentially one CAS instruction plus a few cycles of bookkeeping.

If you remember nothing else from this article, remember this: **an uncontended mutex lock is one CAS instruction.** That is the same instruction `atomic.CompareAndSwapInt32` uses. The cost difference between uncontended mutex and atomic comes down to a few extra cycles of bookkeeping, not a qualitative difference.

### 3.3. Slow path: spinning

When the fast path CAS fails, the mutex is already locked, and we enter `lockSlow`. This is where things get interesting.

The first thing `lockSlow` tries is **spinning**: actively waiting in a tight CPU loop, hoping the current holder releases the lock soon. The rationale is simple — if the holder is going to release the lock in the next few hundred nanoseconds, it is much cheaper to spin and grab it immediately than to park the goroutine and wake it up later (a goroutine park/unpark cycle through the runtime costs on the order of hundreds of nanoseconds to microseconds).

Here is the relevant slice of `lockSlow`:

```go
func (m *Mutex) lockSlow() {
    var waitStartTime int64
    starving := false
    awoke := false
    iter := 0
    old := m.state
    for {
        // Don't spin in starvation mode, ownership is handed off to
        // waiters so we won't be able to acquire the mutex anyway.
        if old&(mutexLocked|mutexStarving) == mutexLocked && runtime_canSpin(iter) {
            // Active spinning makes sense.
            // Set the woken flag to inform Unlock that we are about to need it.
            if !awoke && old&mutexWoken == 0 && old>>mutexWaiterShift != 0 &&
                atomic.CompareAndSwapInt32(&m.state, old, old|mutexWoken) {
                awoke = true
            }
            runtime_doSpin()
            iter++
            old = m.state
            continue
        }
        // ... try to acquire or queue ourselves ...
    }
}
```

The function `runtime_doSpin` ultimately calls `runtime.procyield(cycles)`, which on ARM64 expands to this assembly:

```asm
TEXT runtime·procyield(SB),NOSPLIT,$0-0
    MOVWU cycles+0(FP), R0
again:
    YIELD
    SUBW  $1, R0
    CBNZ  R0, again
    RET
```

A tight loop that issues the `YIELD` instruction (a hint to the CPU that this is a spin-wait loop — the core can temporarily release execution resources to its hyperthreaded sibling, or reduce its own priority) and counts down from the cycle count. Go calls `procyield(30)`, so 30 YIELDs per spin iteration.

`runtime_canSpin(iter)` caps the spinning: at most 4 spin iterations, and only if `GOMAXPROCS > 1` (spinning on a single-core machine is pure waste — there is nothing else to run, so you might as well park), and there is more than one runnable goroutine (so the runtime has something else to do if our spin fails). All told, the worst case is 4 spins × 30 YIELDs = 120 YIELDs before giving up. That is around a few hundred nanoseconds of wall-clock time — short enough that you barely notice, long enough that a holder finishing a small critical section is likely to release the lock in that window.

After spinning, the goroutine builds a new `state` value (marking itself as a waiter if necessary, possibly marking itself starving if it has been waiting too long), CASes it into place, and if it still cannot acquire the lock, falls through to the slowest path of all: parking.

### 3.4. The slowest path: futex and parking

When spinning fails, the goroutine needs to actually sleep until the lock is released. This is where **futex** comes in.

Futex — short for "fast userspace mutex" — is a Linux syscall ([futex(2)](https://man7.org/linux/man-pages/man2/futex.2.html)) introduced in Linux 2.6 (2003) by Hubertus Franke, Matthew Kirkwood, Ingo Molnar, and Ulrich Drepper (yes, the same Drepper who wrote "[What Every Programmer Should Know About Memory](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)"). The original paper, "[Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux](https://www.kernel.org/doc/ols/2002/ols2002-pages-479-494.pdf)" (OLS 2002), is the canonical reference.

The idea is brilliant in its simplicity. Most of the time, a lock is uncontended or only briefly contended. In those cases, you can acquire and release it entirely in userspace using CAS, with no kernel involvement. Only when you actually need to sleep (because the lock is held for a long time) do you call into the kernel. The kernel's only job is to maintain a queue of waiters and wake them up at the right moment.

The two essential futex operations:

- `FUTEX_WAIT(addr, expected)`: check that `*addr == expected`. If so, park the calling thread on the queue associated with `addr` and sleep. If not, return immediately. The check-and-park is atomic — there is no race where the value changes between the check and the sleep.
- `FUTEX_WAKE(addr, n)`: wake up at most `n` threads parked on the queue associated with `addr`.

Go's runtime builds its own semaphore system on top of futex (on Linux) or the equivalent syscall on other operating systems (`umtx` on FreeBSD, `futex` on OpenBSD, etc.). The function `runtime_SemacquireMutex(&m.sema, ...)` ultimately parks the goroutine; `runtime_Semrelease(&m.sema, ...)` wakes one up.

{{< mermaid align="center" >}}
graph LR
    subgraph US["Userspace"]
        CAS["CAS(state, 0, locked)"]
        Spin["Spin: procyield(30) × 4"]
        Slow["lockSlow: build new state"]
    end

    subgraph K["Kernel"]
        Wait["FUTEX_WAIT(sema)"]
        Queue["Waiter queue"]
        Wake["FUTEX_WAKE(sema)"]
    end

    Lock["mu.Lock()"] --> CAS
    CAS -->|success| Done["done (no syscall)"]
    CAS -->|fail — locked| Spin
    Spin -->|got it| Done
    Spin -->|still locked| Slow
    Slow --> Wait
    Wait --> Queue
    Queue -.->|parked, sleeping| SleepZzz["..."]
    Wake -.->|from another g's Unlock| Queue
    Queue -->|woken| CAS

    style CAS fill:#2e7d32,color:#fff
    style Done fill:#2e7d32,color:#fff
    style Wait fill:#c62828,color:#fff
    style Wake fill:#c62828,color:#fff
{{< /mermaid >}}

The cost difference between these paths is enormous. An uncontended CAS is one instruction, a few nanoseconds. A futex round-trip (park and wake) is two syscalls plus context switches, on the order of microseconds. That is a 1000x difference. This is why Mutex tries so hard to stay in userspace: spinning, optimistic CAS, all of it is designed to avoid hitting the kernel.

#### The GMP model: why parking a goroutine is not parking a thread

There is a subtlety here that is easy to miss. Futex in Linux operates on **kernel threads** — `FUTEX_WAIT` puts the calling thread to sleep. But Go uses an **M:N scheduler**: many goroutines (M) are multiplexed onto fewer kernel threads (N, by default equal to the number of CPUs). Before looking at the parking mechanics, here are the components of the Go scheduler — the **GMP model**:

{{< mermaid align="center" >}}
graph TB
    subgraph Gs["G — goroutines (thousands)"]
        G1["G1 running"]
        G2["G2 runnable"]
        G3["G3 runnable"]
        Gn["... Gn"]
    end

    subgraph Ps["P — logical processors (GOMAXPROCS)"]
        P1["P1<br>local runq: [G2, G3]"]
        P2["P2<br>local runq: [G4, G5]"]
    end

    subgraph Ms["M — kernel threads (count ≤ GOMAXPROCS + blocked)"]
        M1["M1 ← P1"]
        M2["M2 ← P2"]
    end

    subgraph SYS["OS kernel"]
        SCHED["Linux scheduler<br>manages M"]
        FUTEX["futex syscall<br>parks M"]
    end

    G1 -.->|runs on| M1
    G2 -.->|queued on| P1
    G3 -.->|queued on| P1
    M1 ===>|bound to| P1
    M2 ===>|bound to| P2
    M1 -.->|syscalls| SCHED
    M2 -.->|syscalls| SCHED
    SCHED --- FUTEX

    style G1 fill:#2e7d32,color:#fff
    style P1 fill:#1565c0,color:#fff
    style P2 fill:#1565c0,color:#fff
    style M1 fill:#c62828,color:#fff
    style M2 fill:#c62828,color:#fff
    style FUTEX fill:#6a1b9a,color:#fff
{{< /mermaid >}}

Three letters:

- **G** (Goroutine) — a user-space goroutine. Lightweight, 2KB starting stack, grows on demand. Hundreds of thousands can coexist.
- **M** (Machine) — a kernel thread. Real, heavy, with its own multi-KB stack. Count is bounded — usually number of CPUs plus a few spares for threads blocked in syscalls.
- **P** (Processor) — a logical processor. Holds a local queue of runnable goroutines (`runq`) and the context to execute Go code. Count of P = `GOMAXPROCS`. For M to run Go code, it needs a P.

When G1 on M1/P1 calls `mu.Lock()` and the mutex is already held, the runtime walks this decision tree:

{{< mermaid align="center" >}}
flowchart TD
    Start([G1 calls mu.Lock<br>mutex already held]) --> Mark["Runtime:<br>G1 → _Gwaiting state<br>remove from P1's runq<br>add to mutex sema queue"]
    Mark --> Check{"Are there other<br>runnable G on P1?"}
    Check -->|yes — e.g. G2, G3| Switch["M1 switches to G2<br>~200ns, no syscall<br>P1 stays acquired"]
    Check -->|no| Global{"Any G in<br>global runq?"}
    Global -->|yes| Take["M1 takes G from globalq<br>or steals from another P"]
    Global -->|no| ParkM["M1 releases P1<br>another M can grab it<br>M1 calls FUTEX_WAIT<br>kernel thread sleeps"]
    Switch --> Work["G2 executes"]
    Take --> Work
    ParkM --> Kernel["... kernel keeps M1 parked<br>in futex queue"]
    Kernel -.->|"somewhere else:<br>Gx releases the mutex"| Wake
    Wake["runtime_Semrelease<br>G1 → _Grunnable<br>placed on some P's runq"]
    Wake --> Rerun["G1 back in queue<br>waits for its M/P<br>and resumes"]

    style Start fill:#c62828,color:#fff
    style Switch fill:#2e7d32,color:#fff
    style Take fill:#2e7d32,color:#fff
    style ParkM fill:#ad1457,color:#fff
    style Wake fill:#1565c0,color:#fff
{{< /mermaid >}}

The crucial check is the first one: "are there other runnable G on P1?". In a typical Go program the answer is almost always **yes** — there are a few more goroutines in the local queue. So M1 does not sleep; it just switches to G2. This context switch happens entirely in user-space (hundreds of nanoseconds), without entering the kernel.

Only if the local queue is empty **and** the global queue is empty **and** nothing can be stolen from other Ps — only then M1 calls `FUTEX_WAIT` and sleeps at the kernel level. That is the optimization that separates Go from C++/Java: instead of parking a heavy kernel thread (with an OS context switch, on the order of microseconds), we park a lightweight goroutine and reuse the kernel thread for other work.

| Platform | What gets parked under contention | Cost | Scale |
|----------|----------------------------------|------|-------|
| **C++ `std::mutex`** | Kernel thread (via futex) | ~1-5 µs (syscall + ctx switch) | ≤ thousands of threads |
| **Java `synchronized`** | Kernel thread (via JVM monitor) | ~1-5 µs | ≤ thousands of threads |
| **Go `sync.Mutex`** | Goroutine (in user-space) | ~100-200 ns | hundreds of thousands of goroutines |

This is why Go can have hundreds of thousands of goroutines waiting on mutexes without consuming hundreds of thousands of kernel threads — that would be a memory disaster (each kernel thread has at least a few KB of stack) and a scheduler disaster (the OS would have to traverse a huge thread queue).

The cost of "parking a goroutine" in Go is not the cost of a syscall, but the cost of a user-space context switch, on the order of a hundred nanoseconds. The `FUTEX_WAIT` syscall only happens when the kernel thread genuinely has no other work. This is another reason why `mu.Lock()` under contention in Go is often faster than the equivalent in C++/Java — we pay only for what we actually use.

For more on the Go scheduler, see Dmitry Vyukov's "[Scalable Go Scheduler Design Doc](https://docs.google.com/document/u/0/d/1TTj4T2JO42uD5ID9e89oa0sLKhJYD0Y_kqxDv3I3XMw/mobilebasic)" (May 2012, still the basis of runtime/sched).

### 3.5. Unlock: symmetric, with a handoff surprise

`Unlock` looks symmetric to `Lock`, but with one twist:

```go
func (m *Mutex) Unlock() {
    // ... race detector bookkeeping ...
    // Fast path: drop lock bit.
    new := atomic.AddInt32(&m.state, -mutexLocked)
    if new != 0 {
        // Outlined slow path to allow inlining the fast path.
        m.unlockSlow(new)
    }
}
```

The fast path is `atomic.AddInt32(&m.state, -mutexLocked)` — one atomic subtract instruction, clearing the locked bit. If the resulting state is zero (no waiters, no flags), we are done — no syscall, no wake-up. This is why uncontended unlock, like uncontended lock, is just a few nanoseconds.

If there are waiters, `unlockSlow` runs, and its behavior depends on whether the mutex is in starvation mode:

```go
func (m *Mutex) unlockSlow(new int32) {
    if (new+mutexLocked)&mutexLocked == 0 {
        fatal("sync: unlock of unlocked mutex")
    }
    if new&mutexStarving == 0 {
        // Normal mode.
        old := new
        for {
            // If there are no waiters, or someone else already woke one,
            // or someone grabbed the lock, we don't need to wake anyone.
            if old>>mutexWaiterShift == 0 ||
                old&(mutexLocked|mutexWoken|mutexStarving) != 0 {
                return
            }
            // Grab the right to wake someone.
            new = (old - 1<<mutexWaiterShift) | mutexWoken
            if atomic.CompareAndSwapInt32(&m.state, old, new) {
                runtime_Semrelease(&m.sema, false, 2)
                return
            }
            old = m.state
        }
    } else {
        // Starving mode: hand off mutex ownership directly
        // to the next waiter, and yield our time slice so that
        // the next waiter can start to run immediately.
        // Note: mutexLocked is not set, the waiter will set it after wakeup.
        runtime_Semrelease(&m.sema, true, 2)
    }
}
```

Look at the last argument to `runtime_Semrelease`: `false` in normal mode, `true` in starvation mode. That boolean is the **handoff flag**. When `true`, the runtime directly transfers ownership of the mutex to the woken waiter — the waiter wakes up already owning the lock, with the locked bit pre-set. When `false`, the woken waiter has to compete for the lock like everyone else.

This single boolean is the entire point of starvation mode. To understand why it matters, we need to look at the bug it was invented to fix.

### 3.6. Starvation mode: the bug behind issue #13086

On October 28, 2015, [Russ Cox opened Go issue #13086](https://github.com/golang/go/issues/13086) with the title "runtime: fall back to fair locks after repeated sleep-acquire failures." The opening line was blunt: "Go's locks make no guarantee of fairness."

The bug report described a simple two-goroutine program. Goroutine 1 holds the lock almost all the time, releasing it for only 100 microseconds at a stretch. Goroutine 2 wants the lock only briefly, every 100 microseconds. The naive expectation is that goroutine 2 should get the lock once in a while, maybe within a second or two.

The reality, on Russ's Linux workstation, was that goroutine 2 took **100 to 600 seconds** to acquire the lock even once. Not milliseconds. Seconds. Minutes. Ten minutes for a single lock acquisition.

Russ's analysis identified the problem precisely. When goroutine 1 calls `Unlock`, it marks the lock unlocked and tells the runtime "wake up goroutine 2." But goroutine 2 does not run immediately. Goroutine 1 keeps running, goes around its loop, calls `Lock` again — and because the lock is now unlocked and goroutine 1 is already on-CPU, goroutine 1 grabs it back. By the time goroutine 2 actually gets scheduled and tries to acquire, goroutine 1 has the lock again. The pattern repeats. Millions of times.

Russ named the problem **barging**. The alternative, where `Unlock` keeps the lock locked and explicitly transfers ownership to the woken waiter, he called **handoff**. In Doug Lea's earlier work on `java.util.concurrent` ([AQS paper, 2003](http://gee.cs.oswego.eu/dl/papers/aqs.pdf)), barging was found to improve throughput. Lea's measurements were on operating system threads with the Linux 2.4 NPTL scheduler, and his argument was that barging avoids bad OS scheduling decisions: if the OS is slow to schedule the woken thread, leaving the lock unlocked lets another thread do useful work in the meantime.

But Go does not use OS threads directly. It uses goroutines and its own user-space scheduler, where a goroutine switch costs tens of nanoseconds, not the microseconds of a thread context switch. The trade-off that justified barging in Java did not necessarily apply to Go.

Russ proposed a hybrid: stay in barging mode (faster) most of the time, but fall back to handoff when severe unfairness is detected. The detection mechanism: a waiter that gets woken up but finds the lock unavailable (because some other goroutine barged in) increments a counter. After enough consecutive failures, the mutex switches to handoff mode.

{{< mermaid align="center" >}}
graph LR
    subgraph Normal["Normal mode (default)"]
        N1["Unlock: mark unlocked"] --> N2["Wake waiter W"]
        N2 --> N3["Continue running"]
        N3 --> N4["Another g calls Lock"]
        N4 --> N5["Barge! Acquire lock"]
        N2 -.->|W scheduled too late| NLost["W wakes, finds lock taken"]
        NLost --> N1
    end

    subgraph Starvation["Starvation mode (after 1ms wait)"]
        S1["Unlock: keep locked"] --> S2["Hand off to next waiter W"]
        S2 --> S3["Yield time slice"]
        S3 --> S4["W wakes owning the lock"]
        S4 --> S5["W runs critical section"]
        S5 --> S1
    end

    Normal -.->|"waiter waited > 1ms"| Starvation
    Starvation -.->|"last waiter OR waited < 1ms"| Normal

    style Normal fill:#1565c015
    style Starvation fill:#c6282815
    style N5 fill:#c62828,color:#fff
    style S4 fill:#2e7d32,color:#fff
{{< /mermaid >}}

The implementation that landed in Go 1.9 (August 2017), authored by Dmitry Vyukov, was slightly different in detail but the same in spirit. Instead of counting failures, it tracks **wall-clock wait time**. If a goroutine has been waiting longer than `starvationThresholdNs = 1e6` (1 millisecond) to acquire the lock, it sets the `mutexStarving` bit. In starvation mode:

- New arrivals do not try to acquire the lock. They go straight to the back of the waiter queue.
- `Unlock` hands off ownership directly (the `true` we saw in `runtime_Semrelease(&m.sema, true, 2)`), and yields the time slice so the woken waiter runs immediately.
- Spinning is disabled — it would be useless since the lock is being handed off, not barged.

The mutex exits starvation mode when either of two conditions is met:

1. The current waiter is the **last** one in the queue (waiter count would drop to 0 after this acquisition), or
2. The current waiter has waited **less than 1ms** in this round.

This is a self-correcting mechanism. When contention drops, the mutex naturally falls back to fast barging mode. When a pathological workload causes starvation, it switches to handoff mode just long enough to drain the queue fairly.

The impact, measured by Russ on his `lockskew` benchmark, was a 500000x speedup in the pathological case — from 100+ seconds per acquisition down to 162 microseconds. And on the common-case throughput benchmark (random number generation under heavy contention), performance actually improved slightly (1-12% faster), contradicting the Java-era wisdom that handoffs always hurt throughput. Goroutine scheduling is cheap enough that the handoff cost is invisible against the rest of the work.

This is why your Go code almost never starves: there is a 1ms threshold, sitting quietly inside `sync.Mutex`, defending you against a class of bugs that took real production systems down in 2015.

### 3.7. The rules: don't copy, don't reenter

Two operational rules that catch people out:

**Never copy a Mutex.** This:

```go
type Server struct {
    mu sync.Mutex
    // ...
}

func (s Server) Handle(req Request) {  // BUG: receiver is by value
    s.mu.Lock()
    defer s.mu.Unlock()
    // ...
}
```

…compiles, but is a bug. The method receiver is a copy of `Server`, which means `s.mu` is a copy of the original mutex — a fresh, unlocked mutex with no connection to the original's `state` or `sema`. Two goroutines calling `Handle` on the same `Server` will each lock their own private copy, providing zero mutual exclusion. `go vet` catches this:

```
$ go vet
./server.go:12:20: Handle passes lock by value: Server contains sync.Mutex
```

The fix: make the receiver a pointer (`func (s *Server) Handle(...)`), or embed `*sync.Mutex` instead of `sync.Mutex`.

The same applies to types that embed a Mutex — `sync.WaitGroup`, `sync.RWMutex`, anything containing one. The general rule: a value containing a Mutex must not be copied after first use.

**Mutexes are not reentrant.** Calling `Lock` twice from the same goroutine deadlocks:

```go
func (s *Server) Outer() {
    s.mu.Lock()
    defer s.mu.Unlock()
    s.Inner()  // BUG: Inner calls s.mu.Lock again — deadlock
}

func (s *Server) Inner() {
    s.mu.Lock()
    defer s.mu.Unlock()
    // ...
}
```

The second `Lock` blocks forever waiting for the first `Unlock`, which will never come because the goroutine is stuck in `Lock`. There is no recursive mutex in Go. The language designers have repeatedly rejected proposals for one, on the grounds that reentrancy encourages sloppy thinking: if a function takes a lock it already holds, that is usually a sign that the locking discipline is unclear, and papering over it with a recursive mutex hides the real problem.

The fix is structural: split `Inner` into two functions, one that assumes the lock is held and one that takes it. Or document the locking discipline explicitly. Or restructure so the same goroutine never needs to acquire the same lock twice.

## 4. When to Use What

You now understand the stack. The remaining question is practical: given a piece of code, do you reach for atomic, or for mutex, or for something else? Let's work through it.

### 4.1. Public benchmarks: the numbers to remember

Most of the published Go mutex-vs-atomic benchmarks cluster around the same numbers. From [thecodinggopher's benchmarks](https://thecodinggopher.substack.com/p/mutex-or-atomics-choosing-the-right), [goperf.dev](https://goperf.dev/01-common-patterns/atomic-ops/), and others, on modern x86 hardware:

| Operation | Uncontended | Under contention |
|-----------|-------------|------------------|
| `atomic.AddInt64(&x, 1)` | 4-8 ns | tens of ns to µs (CAS retries) |
| `atomic.LoadInt64(&x)` | 1-2 ns | 1-2 ns (read-only) |
| `mu.Lock(); mu.Unlock()` | 20-60 ns | hundreds of ns to µs |
| `mu.Lock(); work; mu.Unlock()` (short cs) | ~30 ns + work | scales with contention |
| CAS loop under heavy contention | — | microseconds of CPU burn |

The 5-10x gap between uncontended atomic and uncontended mutex shows up reliably. It comes from the bookkeeping we saw in `Lock`: even the fast path has to set the woken flag, check waiters, manage the state field, not just CAS the value.

Under contention, the picture inverts in a subtle way. A CAS loop under heavy contention can burn hundreds of nanoseconds to microseconds of CPU per operation, because every failed CAS is wasted work. A mutex, by parking the loser, lets the winner proceed without being repeatedly interrupted. For sustained contention, mutex is often faster than naive atomic.

### 4.2. When atomic is right

Use atomic when the operation is on a single memory location and you can express it as one of the atomic primitives.

| Use case | Recommendation |
|----------|----------------|
| Counter (requests/sec, errors/sec) | `atomic.Int64.Add(1)` |
| Done/started flag | `atomic.Bool.Store(true)` / `.Load()` |
| Single-word state (idle/active/stopped) | `atomic.Int32` or `atomic.Uint32` |
| Pointer to immutable snapshot | `atomic.Pointer[T]` |
| Sequence number | `atomic.Uint64.Add(1)` |

If your hot path is a counter being incremented from many goroutines, atomic is 5-10x faster than a mutex and just as correct. Do not put a mutex around a counter "for safety" — it is slower for no benefit.

The trickier case is when you want a more complex atomic operation, like "atomically update two counters together." That is not directly expressible as a single atomic primitive. You have two options:

1. Pack both values into a single `uint64` using bit shifts, and CAS the combined value.
2. Use a mutex.

Packing is faster but limits you to 64 bits total. Mutex is simpler and works for any size. Most of the time, mutex is the right answer here.

### 4.3. When mutex is right

Use mutex when the operation spans multiple memory locations, or when the operation needs to maintain an invariant across multiple fields.

The classic example: updating a map.

```go
type Cache struct {
    mu    sync.Mutex
    items map[string]*Entry
    size  int
}

func (c *Cache) Put(k string, e *Entry) {
    c.mu.Lock()
    defer c.mu.Unlock()
    if _, ok := c.items[k]; !ok {
        c.size++
    }
    c.items[k] = e
}
```

You cannot atomically update `items` and `size` together. They are separate fields. If you tried to use atomic for each, another goroutine could observe `size` updated but `items` not, or vice versa. The mutex guarantees that the entire `Put` operation is atomic with respect to other goroutines: they either see the state before, or the state after, never a mix.

This generalizes: **any operation that needs to read-modify-write multiple fields while maintaining an invariant across them requires a mutex.** Atomics do not help here, because they only synchronize a single word.

### 4.4. atomic.Value vs RWMutex for read-heavy data

For read-heavy data — configuration, feature flags, lookup tables — there are two common patterns:

**Pattern A: atomic.Pointer[T] (copy-on-write).**

```go
var config atomic.Pointer[Config]

func Get() *Config { return config.Load() }

func Update(c *Config) { config.Store(c) }
```

Reads are one atomic load — a few nanoseconds, no contention ever. Writes require building a complete new `*Config` and storing it atomically; the old pointer is garbage-collected once the last reader finishes with it.

**Pattern B: sync.RWMutex.**

```go
var (
    mu     sync.RWMutex
    config *Config
)

func Get() *Config {
    mu.RLock()
    defer mu.RUnlock()
    return config
}

func Update(c *Config) {
    mu.Lock()
    defer mu.Unlock()
    config = c
}
```

Reads take a read lock, which is cheaper than a write lock but not free (it still does an atomic add on the reader count, and writes have to wait for all readers to release).

Which is faster? It depends on the read/write ratio:

- **Millions of reads per second, a few writes per minute**: atomic wins decisively. Reads are 5-10x cheaper, and there is no write-side contention to worry about.
- **Many writes per second, with the data being large**: mutex may win, because building a complete new copy on every write is expensive.
- **You need to mutate the data in place (not replace it wholesale)**: mutex, no question. atomic.Value/Pointer only works for wholesale replacement.

For most configuration-style use cases, `atomic.Pointer[T]` is the right answer.

### 4.5. False sharing: when independent atomics fight

Here is a subtle performance bug. Suppose you have two counters that are updated independently from different goroutines:

```go
type Stats struct {
    Requests int64
    Errors   int64
}

var stats Stats
// goroutine A: atomic.AddInt64(&stats.Requests, 1)
// goroutine B: atomic.AddInt64(&stats.Errors, 1)
```

Looks fine. The two counters are independent. They should scale linearly across cores.

They do not. On a modern CPU, `Stats` is 16 bytes, which fits comfortably in one **cache line** (typically 64 bytes). When goroutine A on core 0 does an atomic add to `Requests`, the CPU invalidates the entire cache line on all other cores. When goroutine B on core 1 does an atomic add to `Errors`, it has to reload the cache line, do the add, and invalidate it on core 0. The two goroutines ping-pong the cache line back and forth, even though they touch different bytes.

This is **false sharing**, and it can slow down parallel code by 5-10x.

The fix is **padding**: insert unused bytes between the hot fields so they land on different cache lines.

```go
type Stats struct {
    Requests int64
    _        [56]byte  // pad to 64 bytes
    Errors   int64
}
```

Now `Requests` and `Errors` are guaranteed to be on different cache lines (assuming 64-byte lines, which is the case on essentially all modern x86 and ARM cores), and the two goroutines can update them independently without invalidating each other's cache.

This is a Level 1 (hardware) concern, but it shows up in Level 5 (practice). Most Go code never needs to worry about it. But if you are writing a high-throughput metrics collector or a hot lock-free data structure, false sharing is one of the first things to look for.

### 4.6. The double-checked locking trap

Here is a classic concurrency bug, in a singleton initializer:

```go
var (
    instance *Config
    mu       sync.Mutex
)

func GetConfig() *Config {
    if instance == nil {            // BUG: non-atomic read
        mu.Lock()
        defer mu.Unlock()
        if instance == nil {
            instance = loadConfig() // BUG: non-atomic write
        }
    }
    return instance
}
```

The "double-check" is the two `instance == nil` tests. The idea is to avoid taking the mutex on the common path (after initialization), while still being thread-safe during initialization.

This is broken in two ways:

1. The first `instance == nil` is a non-atomic read. The Go memory model does not guarantee it observes the write done under the mutex. The compiler and CPU are free to reorder, cache, or otherwise misbehave.
2. Even if the read worked, `instance = loadConfig()` is a non-atomic write. Another goroutine doing the first check might observe a partially constructed `*Config`.

The right answer in Go is `sync.Once`:

```go
var (
    instance *Config
    once     sync.Once
)

func GetConfig() *Config {
    once.Do(func() {
        instance = loadConfig()
    })
    return instance
}
```

`sync.Once` uses atomic operations internally to make the fast path (after initialization) lock-free, and it correctly handles all the memory ordering issues. It is the canonical answer to "lazy initialization in a concurrent context."

If for some reason you cannot use `sync.Once`, the correct hand-rolled version uses atomic load and store:

```go
var instanceP atomic.Pointer[Config]

func GetConfig() *Config {
    if c := instanceP.Load(); c != nil {
        return c
    }
    // ... take mutex, double-check with Load, build, Store ...
}
```

The atomic `Load` and `Store` provide the memory ordering guarantees that plain reads and writes do not.

### 4.7. Cheatsheet

Here is the one-page summary. When you are staring at a piece of code and asking "atomic or mutex?", run through this table.

| Scenario | Recommendation | Why |
|----------|----------------|-----|
| Counter (`requests++`) | `atomic.Int64.Add(1)` | 5-10x faster than mutex, just as correct |
| Boolean flag (`stopped`) | `atomic.Bool` | One-word state, no invariant to protect |
| Single-word state machine | `atomic.Int32` or `atomic.Uint32` | Encode states as small integers |
| Pointer to immutable snapshot | `atomic.Pointer[T]` | Lock-free reads, atomic replacement |
| Update map + counter together | `sync.Mutex` | Multi-field invariant requires critical section |
| Update slice in place | `sync.Mutex` | Slice header + backing array, multi-word |
| Read-heavy config (rare updates) | `atomic.Pointer[T]` or `sync.RWMutex` | `atomic.Pointer` if you can replace wholesale; `RWMutex` if you mutate in place |
| Per-goroutine state | No synchronization | If only one goroutine touches it, no race |
| Lazy singleton init | `sync.Once` | Hand-rolled double-checked locking is wrong |
| Hot path with two independent counters | `atomic.Int64` × 2 with padding | False sharing if struct-packed |

If you remember nothing else from this article, remember this: **single-word state — atomic. Multi-field invariant — mutex. Anything else — measure and think.**

## 5. Conclusion

We have traveled the whole stack. From `counter++` being three ARM64 instructions, through CAS as the universal primitive (Herlihy 1991), through `LOCK CMPXCHG` and ARM LSE, through Go's `sync/atomic` package with its single sequentially-consistent memory ordering, through futex as the bridge between userspace and kernel, through `sync.Mutex`'s packed `state` field with its spinning and starvation mode, and finally to the practical recommendations.

The two main takeaways:

1. **Mutex and atomic are one stack, not two tools.** Mutex is built on atomic CAS, with a fairness layer on top. Uncontended `mu.Lock()` is one CAS instruction — barely more expensive than `atomic.CompareAndSwapInt32`. The cost difference shows up only under contention, and that is where mutex's parking (instead of busy-spinning) actually wins.
2. **Starvation mode is not theoretical.** It exists because real production code was hitting 100-second lock acquisition times in 2015 (issue #13086). The 1ms threshold inside `sync.Mutex` is silently defending your code against that pathology, every day.

If you want to dig deeper, the references below are the canonical reading list. The Go source files ([`sync/mutex.go`](https://cs.opensource.google/go/go/+/refs/tags/go1.24.0:src/internal/sync/mutex.go), [`runtime/sema.go`](https://cs.opensource.google/go/go/+/refs/tags/go1.24.0:src/runtime/sema.go)) are readable, well-commented, and not as scary as they look. Russ Cox's [memory model revision notes](https://research.swtch.com/gomm) and the [issue #13086 discussion](https://github.com/golang/go/issues/13086) are the two pieces of historical context that explain why the code looks the way it does.

The next article in the Go Internals series will look at `sync.Pool` — which builds on `sync/atomic` and per-P local storage to give Go one of the cheapest object allocation strategies of any mainstream language. Stay tuned.

### References

- Go Memory Model reference — [go.dev/ref/mem](https://go.dev/ref/mem)
- Go 1.19 release notes (atomic types, memory model revision) — [go.dev/doc/go1.19](https://go.dev/doc/go1.19)
- Russ Cox, "Go memory model" revision notes — [research.swtch.com/gomm](https://research.swtch.com/gomm)
- Go issue #13086 — "runtime: fall back to fair locks after repeated sleep-acquire failures" — [github.com/golang/go/issues/13086](https://github.com/golang/go/issues/13086)
- Hans-J. Boehm, Sarita V. Adve, "Foundations of the C++ Concurrency Memory Model," PLDI 2008 — [dl.acm.org/doi/10.1145/1375581.1375591](https://dl.acm.org/doi/10.1145/1375581.1375591)
- Maurice Herlihy, "Wait-Free Synchronization," ACM TOPLAS 13(1), 1991 — [cs.brown.edu/people/mph/Herlihy91/p124-herlihy.pdf](https://cs.brown.edu/people/mph/Herlihy91/p124-herlihy.pdf)
- Doug Lea, "The java.util.concurrent Synchronizer Framework" (AQS paper), 2003 — [gee.cs.oswego.edu/dl/papers/aqs.pdf](http://gee.cs.oswego.edu/dl/papers/aqs.pdf)
- Hubertus Franke, Matthew Kirkwood, Ingo Molnar, Ulrich Drepper, "Fuss, Futexes and Furwocks: Fast Userlevel Locking in Linux," OLS 2002 — [kernel.org/doc/ols/2002/ols2002-pages-479-494.pdf](https://www.kernel.org/doc/ols/2002/ols2002-pages-479-494.pdf)
- Linux futex(2) man page — [man7.org/linux/man-pages/man2/futex.2.html](https://man7.org/linux/man-pages/man2/futex.2.html)
- Ulrich Drepper, "What Every Programmer Should Know About Memory," 2007 — [people.freebsd.org/~lstewart/articles/cpumemory.pdf](https://people.freebsd.org/~lstewart/articles/cpumemory.pdf)
- Go sync/mutex.go source (Go 1.24) — [cs.opensource.google/go/go/+/refs/tags/go1.24.0:src/internal/sync/mutex.go](https://cs.opensource.google/go/go/+/refs/tags/go1.24.0:src/internal/sync/mutex.go)
- VictoriaMetrics blog, "Go sync.Mutex: Normal and Starvation Mode" — [victoriametrics.com/blog/go-sync-mutex](https://victoriametrics.com/blog/go-sync-mutex/)
- goperf.dev, "Atomic Operations and Synchronization Primitives" — [goperf.dev/01-common-patterns/atomic-ops](https://goperf.dev/01-common-patterns/atomic-ops/)
- "Mutex or Atomics? Choosing the Right Tool in Go" — [thecodinggopher.substack.com/p/mutex-or-atomics-choosing-the-right](https://thecodinggopher.substack.com/p/mutex-or-atomics-choosing-the-right)
