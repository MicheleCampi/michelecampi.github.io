---
layout: post
title: "I built an eBPF profiler for vLLM cold start. Here is what it can and cannot see."
date: 2026-06-15
categories: observability systems-engineering llm-inference
tags: rust ebpf llm vllm profiling observability gpu cold-start aya
---

> **Correction, 2026-10-15.** This post's original central claim does not hold. The traced `read` calls move 0.53 GiB while the weights are 13.5 GiB: the weights do not pass through the four traced syscalls, so the kernel-side times below are time spent in those syscalls, not disk I/O, and the post cannot say how much of cold start is the disk. No capture recorded "eighteen seconds"; the load time recorded for a phase A capture of this model is 11.73 s. The 1.5x scaling compares a 32B on an A100 with a 7B on an A10. The tool, the tables as time inside the traced calls, and the single 1.29 s launch stand. The sections that depended on it were rewritten on 2026-10-15; the corrected findings are in the [probe's README](https://github.com/MicheleCampi/vllm-coldstart-probe#readme).

A vLLM server loading Mistral-7B (FP16, 13.5 GiB of weights) has to read those weights, set up the GPU and warm the engine before it answers the first request. The obvious assumption — the one I started with — is that most of that time is reading the weights off the disk.

I built an eBPF probe to check, and it could not answer that question: the weights do not pass through the syscalls it traces. This post is about the tool, what it does measure — time inside four syscalls and four CUDA driver calls, on one clock — and why the first version of this post drew the wrong conclusion from it.

## The question: how much of cold start is the disk?

Cold start is the time between "process starts" and "server can serve a token." It matters more than it used to. Scale-to-zero inference, autoscaling under bursty load, spot-instance preemption recovery — all of them pay the cold-start cost every time a replica comes up, and on a GPU you are paying by the second for hardware that is doing no useful work during that window. On the probe's captures, constructing the engine for a 7B model on an A10 with a cold page cache took 11.7 to 19.0 s in eager mode ([CAPTURES.md](https://github.com/MicheleCampi/vllm-coldstart-probe/blob/main/analysis/CAPTURES.md)); at $1.29/hr that is not much money on its own, but multiply it by every scale event across a fleet and it becomes a line item.

The intuitive model of cold start is I/O-bound: you have to move 13.5 GiB of weights from disk into GPU memory, so the cost is the bandwidth of that path. If that were true, the fix would be faster disks, prefetching, or keeping weights in page cache. So the first question worth answering precisely is: how much of cold start is actually I/O?

To answer it I needed something that could see, on one clock, both the kernel-side I/O (the syscalls a loader uses to open and read files) and the userspace/GPU work (the CUDA calls setting up the device). I had neither tool on hand. So I built one.

## Why /proc sampling and OTel weren't enough here

I have [written before](https://michelecampi.github.io/observability/systems-engineering/llm-inference/2026/05/21/profiling-llm-inference-proc-sampler.html) about sampling `/proc/<pid>` to profile a running inference server, and [before that](https://michelecampi.github.io/observability/systems-engineering/2026/05/17/otel-tracing-compute-bound-services.html) about why OpenTelemetry spans show you nothing when the work is compute-bound and happening below your instrumentation. Both tools share a boundary: they stop at the process. `/proc` sampling tells you RSS and CPU and thread count at some interval; OTel tells you about spans you chose to instrument. Cold start lives below that boundary. The interesting work is individual syscalls (microsecond-scale, far faster than any sampler) and calls into the CUDA driver (which no Python-level tracer sees). A 50 ms sampling loop averages right over the thing you want to see.

eBPF is the right instrument for this. You attach a small program to a kernel tracepoint or to a function in a userspace library; it runs on every hit, in kernel context, and pushes an event into a ring buffer that userspace drains. No sampling interval — you see every call. The cost is per-event, not per-interval, so it scales with how busy the thing is rather than how long you watch it.

## What the probe attaches to

The tool — [vllm-coldstart-probe](https://github.com/MicheleCampi/vllm-coldstart-probe), Rust, Apache-licensed, built on [aya](https://aya-rs.dev/) — attaches two families of probe.

On the kernel side, four syscall tracepoints, each traced on both enter and exit so I can compute how long each call spent in the kernel: `openat`, `read`, `mmap`, `close`. These are the syscalls I expected a model loader to use to get its weights off disk; the captures show the weights take another path.

On the userspace side, four entry points in the CUDA driver library `libcuda.so`, each traced with a uprobe on entry and a uretprobe on return, so I get the duration spent inside each driver call: `cuInit`, `cuModuleLoadData`, `cuMemAlloc_v2`, `cuLaunchKernel`. (`cuMemAlloc_v2` is the real exported ABI symbol; the unsuffixed name you write in C is a header macro.)

Both families write the same fixed-layout event into one ring buffer:

```rust
#[repr(C)]
pub struct SyscallEvent {
    timestamp_ns: u64,
    pid: u32,
    tid: u32,
    syscall_nr: u32, // real syscall number, or >= 1000 for a uprobe id
    kind: u8,        // 0 = enter, 1 = exit
    ret: i64,
}
```

Reusing one event type for both keeps the userspace side simple: one ring buffer, one reader, one JSONL format. The `syscall_nr` field doubles as a discriminator — anything `>= 1000` is a libcuda call, and the analysis splits on that boundary. The kernel-side functions are generated by a small macro so adding a syscall or a CUDA symbol is one line, not a copy-pasted function body:

```rust
macro_rules! define_uprobe {
    ($short:ident, $event_id:expr) => {
        ::paste::paste! {
            #[uprobe]
            pub fn [<probe_ $short>](_ctx: ProbeContext) -> u32 {
                emit_uprobe($event_id)
            }
            #[uretprobe]
            pub fn [<probe_ $short _ret>](_ctx: RetProbeContext) -> u32 {
                emit_uprobe_ret($event_id)
            }
        }
    };
}
```

There were two non-obvious fights to get here. A kernel-side PID filter using a BPF map kept getting dead-code-eliminated by the Rust toolchain regardless of the usual workarounds, so filtering happens in userspace on the drained stream instead. And `#[uretprobe]` takes a `RetProbeContext`, a different type from the `#[uprobe]` entry side's `ProbeContext` — a one-line fix once the compiler tells you, but the kind of thing that isn't in the examples.

## The capture: Mistral-7B on an A10, cache dropped

The capture host is a Lambda Labs A10 (24 GB, Ampere), Ubuntu 22.04, kernel 6.8. RunPod, which I'd used for earlier work, turns out not to grant the capabilities eBPF needs in its consumer pods — no `CAP_BPF`, no `CAP_PERFMON`, no `/sys/kernel/tracing` — so a full VM was required. The probe is a 4.3 MB statically-linked musl binary, scp'd onto the box and run under sudo.

Two things had to be right for the numbers to mean anything.

First, the page cache. If you load the model once to download it and then measure the second load, the 14 GB is already in RAM and `read` returns in microseconds from cache — you measure nothing about disk. So I drop the page cache (`echo 3 > /proc/sys/vm/drop_caches`) immediately before the measured run, forcing real SSD reads.

Second — and this was the surprise that cost me a run — the PID. vLLM's v1 architecture does the actual model load and CUDA work in a separate `EngineCore` subprocess, spawned after my probe has already attached. My first capture filtered on the launcher PID and recorded **zero** libcuda events, despite all four uprobes attaching cleanly. The CUDA calls were all coming from a child process I wasn't watching. The fix was a `--pid 0` mode that captures every process and filters to the target's process tree afterward — the robust way to handle a target that forks its real workers after you attach.

## What the syscalls show, and what they don't

With the cache dropped and the whole process tree captured, here is the time spent inside the four traced syscalls:

| syscall | calls  | total time | p50    | max   |
|---------|--------|-----------|--------|-------|
| read    | 23,137 | 1037 ms   | 1.1 us | 44 ms |
| openat  | 19,454 |  154 ms   | 2.6 us | 1.4 ms|
| mmap    |  1,846 |   16 ms   | 4.8 us | 1.0 ms|
| close   | 16,803 |   16 ms   | 0.8 us | 0.3 ms|

Total time in these syscalls: about 1.22 seconds, with `read` at 1037 ms. That is not the weights coming off the SSD. These `read`s move 0.53 GiB; Mistral-7B's weights are 13.5 GiB. The weights reach memory some other way, through a path these four syscalls do not see, so this table is syscall time, not disk I/O, and it cannot say what share of cold start the disk takes. The first version of this post read it as the disk and computed 7% of an eighteen-second load; the load time of this capture was not kept, and no capture recorded eighteen seconds.

One detail worth noting: 1,846 `mmap` calls against 23,137 `read` calls. The probe times the `mmap` calls but not the page faults that fill a mapping, so a mapped file is read without any traced syscall doing the reading. Whether that is how the weights arrive is the probe's next question, not something this capture shows.

So this capture does not say how much of cold start is the disk. It says that reading the weights is not paid inside these four syscalls.

## The CUDA calls: many, mostly instant

The other half of the probe is the CUDA driver. vLLM makes a lot of driver calls during cold start. Here is the libcuda side, durations measured entry-to-return:

| function         | calls | total time | p50    | max     |
|------------------|-------|-----------|--------|---------|
| cuLaunchKernel   |   947 | 1337 ms   | 4.2 us | 1287 ms |
| cuInit           |     6 |  123 ms   | 5 us   |  123 ms |
| cuMemAlloc_v2    |   179 |   34 ms   | 106 us |    2 ms |
| cuModuleLoadData |     1 |    1 ms   | 780 us |    1 ms |

`cuLaunchKernel` is called 947 times. The total is 1337 ms. So the kernels are the cost, right?

Look at the p50: 4.2 microseconds. The median kernel launch is instant. 947 launches at 4 us each is about 4 ms. The 1337 ms total is not spread across the launches — it is in the tail.

## One kernel launch, and the gaps

The `max` column is the tell. One single `cuLaunchKernel` took **1287 ms** — 1.29 seconds in one call, against a median of 4 microseconds. So I went looking at what that one call was, in the capture data.

It is not the first launch. It is the 516th of 947, about 7.2 seconds into the warmup, with 515 fast launches before it. During its 1.29 seconds the process issues almost no syscalls — two `read`s, nothing else — and the one `cuModuleLoadData` of the whole run happens almost two seconds *after* it, not before. Of my first three guesses — disk I/O, compilation writing a kernel cache, a module load feeding this launch — the data rules out the last two: no file is opened while it runs, and the module load comes after it. The disk it cannot rule out, because page faults on mapped files are not traced.

What it most likely is: a synchronization point. CUDA kernel launches are asynchronous — `cuLaunchKernel` returns as soon as the kernel is queued, and the GPU works through the queue behind it. The 515 fast launches before this one queued work that the GPU was still chewing on. At launch 516 something forces the host to wait — a full queue, an allocation that needs prior work to finish, an implicit sync — and this single call pays the bill for all the accumulated GPU compute at once. The 1.29 seconds is not spent inside the launch; it is spent waiting for the GPU, accounted to the call that happened to block.

I can't prove that from this capture alone — I'm not tracing `cuStreamSynchronize` or `cudaDeviceSynchronize`, which is exactly what would confirm it. What the data does prove is narrower: this cost is not the first launch and not compilation writing to disk. `cuInit` is cleaner: one 123 ms call plus five no-op repeats, the driver initialising once.

Now add it up. Time in the four syscalls: 1.22 s. Time in the four libcuda calls: 1.49 s. Together, about 2.7 seconds. The load time of this capture was not kept, so there is no honest share of the total to give; the first version of this post divided by eighteen seconds that no capture recorded.

The rest is in the gaps — the wall-clock time that elapses *between* the traced calls, where the process is neither in a syscall I'm watching nor inside a libcuda function I'm watching. That time includes reading the weights, by whatever path they take, as well as GPU compute that has already been launched and is running asynchronously, CUDA runtime and framework work above the driver, and plain Python: importing torch, building the model object, moving tensors. The probe sees the boundaries of that work, not its interior, and cannot split it between these.


## Does it hold as the model grows?

One model is an anecdote. The obvious next test is how the probe's numbers move with model size. So I ran the same probe against Qwen2.5-Instruct in AWQ at three sizes — 7B, 14B, 32B — same family, same quantization, only the parameter count changing. (The 7B and 14B fit on the A10; the 32B needed an A100 40 GB.) Keeping quantization fixed matters: AWQ changes the kernel count dramatically versus FP16 — the 7B-AWQ issues 3,475 `cuLaunchKernel` calls where Mistral-7B in FP16 issued 947 — so mixing precisions would confound size with quantization. Holding it constant isolates size.

The load times:

| model    | params | load time | syscall time | cuLaunchKernel calls | cuMemAlloc calls |
|----------|--------|-----------|-----------|----------------------|------------------|
| 7B-AWQ   | 7B     | 18.97 s   | ~1.86 s   | 3,475                | 161              |
| 14B-AWQ  | 14B    | 25.02 s   | ~2.27 s   | 6,547                | 372              |
| 32B-AWQ  | 32B    | 28.36 s   | ~1.93 s   | 8,691                | 418              |

On the same A10, doubling the parameters (7B to 14B) makes the load 1.32x longer. The 32B ran on an A100, so its 1.50x against the 7B, with 4.6x the parameters, mixes model size with a different GPU; it is not a clean scaling point.

Three things hold across the sweep, and one is worth being careful about.

`cuInit` and `cuModuleLoadData` are fixed costs — they do not move with model size (cuInit is ~80–140 ms at every size, the variation is run-to-run noise). The driver does not care how big your model is.

Syscall time does not grow with size either: around 2 s at all three sizes, and the traced `read`s move 0.54 GiB at every size — not the weights, as in the single-model capture. So this says nothing about whether the larger models' weights take longer to read.

The *number* of operations does scale with size: `cuLaunchKernel` count rises monotonically 3,475 → 6,547 → 8,691, `cuMemAlloc` 161 → 372 → 418. More model, more kernels and more allocations — but sub-linearly, not proportionally. (The 14B capture dropped 1,493 events, so its counts are lower bounds.)

The careful part: the *time* inside those launches does not scale cleanly. Total `cuLaunchKernel` time was 1.77 s / 3.41 s / 2.81 s — the 14B spent more total launch time than the larger 32B. That is the same effect from the single-model section: the launch time is dominated by a few synchronization outliers, and where the sync tax lands is noisy run-to-run, not a function of model size. So the honest statement is precise: the *count* of operations scales sub-linearly with size; the *time inside the launches* is dominated by a few long calls (the longest is 93%, 57% and 43% of it at 7B, 14B and 32B), and where those land is noisy. What the probe cannot say is how load time splits between those calls and reading the weights.

## What this means for cold start

What holds across every model I tried is narrower than the first version of this post claimed. Driver init is one fixed cost, flat across sizes. Kernel launches are many and almost all instant; their time sits in a few long calls that most likely wait on the GPU. Whether making cold start faster is a storage problem these captures cannot say, because the weights take a path the probe does not trace.

So the first thing to measure next is the path the weights take: page faults on mapped files, or whatever else moves those 13.5 GiB. Until then, the levers worth naming are the ones that do not depend on the answer: keeping a replica warm rather than paying cold start at all, or persisting a warmed state. CUDA graphs, which I named here as a lever on warmup, made the load 3.2x slower with 79x the kernel launches in the probe's phase D; [a later post]({% post_url 2026-07-26-cuda-graphs-tradeoff %}) measures that trade-off. I used `enforce_eager=True` to keep the trace readable.

## What the probe can't see yet

The honest limit: everything outside those 2.7 seconds is in the gaps, and "weights, GPU compute, runtime, and Python" is a bucket, not an attribution. Narrowing it means more probe points — page faults on mapped files first, then more libcuda entry points, the CUDA runtime layer, uprobes into libtorch — to push the boundary of what's measured further into that gap. Each one converts some of the "between the calls" time into named, timed work. That is the next round.

There is also a structural ceiling, and the 1.29 s launch is the clearest example of it. A uretprobe times wall-clock duration inside a driver call, but a `cuLaunchKernel` returns when the kernel is queued, not when it finishes on the GPU. Most launches return in microseconds because the queueing is all that happens on the host; the one that blocked for 1.29 s did so because something made the host wait for the device. To attribute that wait properly — to say how much is queued compute versus an explicit sync — you have to correlate host-side launches with device-side completion, which needs CUPTI or NVML on the same clock. That is exactly the boundary my earlier `/proc`-plus-NVML profiler worked at, from the other side.

Which is the through-line. The [process-level profiler](https://github.com/MicheleCampi/inferscope) stops at the process boundary and looks down; this one starts at the kernel and driver and looks up. Cold start is one of the places where you need both, because the cost is split across exactly the seam where most tools stop. The probe is on [GitHub](https://github.com/MicheleCampi/vllm-coldstart-probe), Apache-licensed, with the analysis script and CAPTURES.md behind the numbers above. If you run inference cold starts at scale and have measured them differently — or found the device-side cost somewhere I haven't looked — I'd be interested to hear about it.
