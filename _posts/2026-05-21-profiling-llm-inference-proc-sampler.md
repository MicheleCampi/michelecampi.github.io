---
layout: post
title: "Profiling LLM inference: what your /proc sampler isn't telling you"
date: 2026-05-21
categories: observability systems-engineering llm-inference
tags: profiling llm inference nvml gpu rust linux process-tree inferscope
---

Most LLM inference profilers that read `/proc/<pid>` to track
memory and CPU usage are quietly monitoring the wrong process.
The number they show is correct — it just describes a transient
shell wrapper, not the long-lived worker doing the actual
inference. The output looks plausible, the GPU panel says the
GPU is hot, and nothing breaks. The data just doesn't mean what
the chart label says it means.

This is a class of bug, not a one-off. I ran into it while
validating `inferscope v0.2` on an NVIDIA RTX L4, and the
shape of the failure is general enough that it likely shows up
in any pipeline that captures a PID via `$!` after a bash
background launch. This post walks through what I observed,
why it happens at the kernel level, and what a defensible fix
looks like at the tooling layer.

## The signal that didn't match the reality

`inferscope` is a Rust profiler I'm building for LLM inference
engines: it drives an engine over its OpenAI-compatible HTTP
API, captures per-token timing, and (in v0.2) also samples GPU
resources via NVML. The CPU-side sampler reads
`/proc/<pid>/status` and `/proc/<pid>/stat` every 50 ms for the
duration of the probe, mirroring the
[ADR](https://github.com/MicheleCampi/inferscope/blob/main/docs/adr/003-sysmon-scope-and-correlation.md)
that defines the sampling cadence.

For the v0.2 validation I rented an RTX L4 on RunPod, brought
up a `llama-server` (llama.cpp) instance serving Qwen 2.5 0.5B
Q4 on the GPU, and ran `inferscope --gpu` against it. The first
output looked like this:

    Probe summary
      Tokens generated      80
      Generation rate       375.7 tokens/s

    Process resource usage (6 samples)
      RSS                peak 2 MiB  mean 2 MiB
      CPU utilization    mean 0%
      Threads            1 throughout

    GPU resource usage (6 samples)
      VRAM               peak 1.25 GiB
      SM utilization     peak 83%  mean 41%

The GPU panel reports a workload doing real work: 1.25 GiB of
VRAM resident, SM utilisation peaking at 83%, sustained
throughput of 375 tokens per second. The process panel
describes a process barely alive: 2 MiB of RSS, zero CPU
activity, one thread.

These two views cannot both be true of the same process. A
program serving an LLM at 375 tokens/s has to do *some* host
work — tokenisation, request handling, HTTP serialisation,
memory management around the KV cache. It will not fit in 2 MiB
of RSS or run in a single idle thread. So either the GPU panel
is wrong, or the process panel is monitoring something other
than the inference worker.

A quick check confirmed the second:

    $ nvidia-smi
    NVIDIA L4 ... 1282 MiB / 23034 MiB ... 0% util  <-- idle
                  ... 1.25 GiB ...                  <-- matches inferscope GPU
    ... no other processes on the GPU ...

The GPU side was accurate. The process side was not the inference
worker.

## What the kernel was actually telling us

The way I had launched the server matched the pattern most
tutorials suggest:

    ./build/bin/llama-server -m model.gguf --port 8080 \
      --n-gpu-layers 99 > /tmp/llama-server.log 2>&1 &
    SERVER_PID=$!

That `$!` is the source of the bug. Bash's `$!` returns the PID
of the most recent background job, which under output
redirection is *not* the binary you launched. Bash creates a
subshell to handle the redirection setup, that subshell becomes
the background job, and only then does the subshell `exec` into
the real binary — or in some cases, `fork+exec`, leaving the
subshell around as a parent that exists only to maintain job
control.

In the Qwen 2.5 0.5B run, the difference between the two PIDs
was directly visible in `/proc`:

    $ cat /proc/2252/status | grep -E 'Name|VmRSS|Threads'
    Name:   bash
    VmRSS:    2144 kB
    Threads:    1

    $ cat /proc/2254/status | grep -E 'Name|VmRSS|Threads'
    Name:   llama-server
    VmRSS:  523804 kB
    Threads:    52

    $ cat /proc/2254/status | grep PPid
    PPid:   2252

PID 2252, the value `$!` had handed me, was `bash`. PID 2254
was the actual `llama-server`, with the bash wrapper as its
parent. Both processes were running; both were valid `/proc`
entries; the sampler was faithfully reading `/proc/2252/status`
and reporting exactly what was there.

This is the failure mode I want to be explicit about: **the
sampler was not broken, the data was not wrong, the contract
was not violated**. The PID I supplied was a real PID of a
real, currently-running process. The tooling can't tell that
the user intended a different PID. From the sampler's point of
view, asking it to monitor PID 2252 and getting back "2 MiB,
0% CPU, 1 thread" is the correct answer to the question that
was asked.

This is why the bug is interesting. It can't be fixed by being
more careful with the sampler. The sampler is already correct.

## Why this shows up in more pipelines than just mine

If you supply a PID to your profiling tooling using `$!` after
a background launch with output redirection, you are likely
seeing a similar pattern. The shape varies — some shells, some
configurations, and some binaries do an in-place `exec` that
keeps the PID consistent, so the bug stays hidden until you
move environments. Some inference servers (vLLM, Ollama,
llama.cpp built differently) fork additional workers, so even a
correct top-level PID misses the workers that hold the model
weights. Some Docker entrypoints add another wrapper layer.

The general statement: in a pipeline where you launch an
inference server in the background, capture its PID
programmatically, and hand that PID to anything that samples
`/proc`, you should verify at least once on real hardware that
the PID actually points at the worker doing the inference. The
right check is two lines:

    $ cat /proc/$SERVER_PID/status | grep -E 'VmRSS|Threads'
    VmRSS:    523804 kB
    Threads:  52

A real LLM inference worker has VmRSS at least in the hundreds
of MiB (the model weights and runtime live there even with GPU
offload) and threads well above one. Anything else is a wrapper
or an idle process and should not be the PID you're sampling.

## The fix, in two pieces

There are two reasonable places to handle this kind of failure:
the documentation and the runtime.

The documentation fix is the easy half. The
[runbook](https://github.com/MicheleCampi/inferscope/blob/main/docs/runbooks/runpod-gpu-validation.md)
now tells the user not to use `$!`, and gives them the
alternative:

    SERVER_PID=$(pgrep -x llama-server | head -1)

`pgrep -x` matches on the exact process name rather than on the
command line, so it returns the long-lived worker rather than a
wrapper or a `grep` of the command itself. The runbook also
includes the sanity check above as a pre-flight step before
invoking `inferscope`.

The runtime fix is the more interesting half. After the
probe completes, `inferscope` now runs a single pessimistic
heuristic against the collected timeline. If *every* sample
shows RSS below 10 MiB *and* exactly one thread *and* zero CPU
jiffies (user plus system), it emits a stderr warning:

    inferscope: warning: monitored PID looks idle across all 6
    samples (RSS < 10 MiB, 1 thread, 0 CPU jiffies). The --pid
    argument may point to a wrapper shell rather than the
    actual workload. Verify with:
    cat /proc/<pid>/status | grep -E 'VmRSS|Threads'.

Each of the three conditions on its own would be a poor signal.
A small daemon can legitimately use less than 10 MiB; a
single-threaded program is normal; a momentarily idle process
will register 0 jiffies in a 50 ms window. Requiring all three
to hold across every sample in the timeline is what keeps the
false-positive rate down. A real inference worker never matches
all three for the entire duration of a 200-ms probe run; an
idle bash wrapper matches all three trivially.

The warning is informational only. It does not abort the run
or change the report — the user still sees the GPU section,
the timing section, and the (wrong) process section. The point
is not to refuse to produce output, but to make sure the user
can't accept the misleading output without being told it might
be misleading.

## Empirical validation, A/B

The two runs below were taken back-to-back on the same RTX L4
pod, against the same llama-server instance, with the same
prompt and `--max-tokens` value. The only difference is the
PID supplied to `--pid`.

PID 2252 (the bash wrapper, the wrong PID, the case the warning
should catch):

    inferscope: warning: monitored PID looks idle across all 6
    samples (RSS < 10 MiB, 1 thread, 0 CPU jiffies). ...

    Process resource usage (6 samples)
      RSS                peak 2 MiB  mean 2 MiB
      CPU utilization    mean 0%
      Threads            1 throughout

    GPU resource usage (6 samples)
      VRAM               peak 1.25 GiB
      SM utilization     peak 91%  mean 45%

PID 2254 (the real `llama-server`, the right PID, no warning):

    Process resource usage (7 samples)
      RSS                peak 511 MiB  mean 510 MiB
      CPU utilization    mean 73%
      Threads            52 throughout

    GPU resource usage (7 samples)
      VRAM               peak 1.25 GiB
      SM utilization     peak 81%  mean 23%

The GPU panels match — slightly different distributions because
each run captures a different slice of work, but the same
ballpark (1.25 GiB resident, peak utilisation in the 80–90%
range). The process panels are now what you would expect from
a real inference server: half a gigabyte resident for the model
weights and runtime, 52 threads from the worker pool and HTTP
handlers, sustained CPU utilisation during the request because
even GPU-offloaded inference does meaningful host-side work.

The warning fires for the wrong PID and stays silent for the
right one. The validation cost was about $0.40 of RunPod credit
across two sessions on the L4 ($0.39/hr).

## What this means beyond inferscope

Three things are worth taking away even if you never use
inferscope.

The first is that `$!` is the wrong PID-capture pattern when
the launch uses background plus output redirection. The right
one is `pgrep -x <binary-name>` after a short sleep to let the
worker stabilise. This applies to llama.cpp, vLLM, Ollama,
TGI, and any other server you launch this way from a shell
script.

The second is that "the process panel and the GPU panel
disagree" is a high-signal symptom. If your profiling
dashboard shows the GPU is hot and the process is idle, the
process being monitored is not the one talking to the GPU.
This is true even when the profiling tooling is correct —
particularly when the profiling tooling is correct, because a
correct tool will faithfully report the state of whatever
process you pointed it at.

The third is more general. Validation on real hardware
surfaces a class of bugs that no amount of unit testing in a
clean room will catch. The `inferscope` test suite (109 tests
green with `--features gpu-nvidia` enabled) passed in CI from
day one. The PID bug was not visible until the tool was
running against a real LLM workload on a real GPU. The
inexpensive fix — half a day of RunPod time, two commits, and
a runbook entry — was only possible because the validation
budget existed. If your tooling has no real-hardware
validation step in its release process, every release is on
the honour system.

## What I'd do differently next time

If I were starting the v0.2 NVIDIA path from scratch, I would
not add a process-tree-aware sampler in v0.2. That is the
correct long-term fix — the sampler should aggregate
`/proc/<pid>/task/*/stat` for the supplied PID plus its
descendants — but it is meaningful design work and has its
own correctness gotchas (cgroup boundaries, namespace
crossings, descendant PID enumeration races). The
pessimistic-AND warning is the right thing for v0.2 because
it is small, defensible, and informational. The full process-
tree sampler is on the v0.3 list.

I would also write the validation runbook earlier — before
the first real-hardware test, not after. The runbook I wrote
post-hoc captures all the steps I had to figure out the
first time; if it had existed up front, the PID bug would
have been documented in the runbook before it ever fired in
inferscope output.

## Roadmap

v0.3 adds AMD GPU sampling via `amd-smi`, the successor
tool AMD is positioning as the long-term replacement for
`rocm-smi`. `rocm-smi`'s JSON output is documented as
unstable across ROCm upgrades, so investing parser work on
it now would mean refactoring it within months;
[ADR-005](https://github.com/MicheleCampi/inferscope/blob/main/docs/adr/005-gpu-resource-sampling.md)
records that decision in full.

v0.4 and beyond is engine-internal instrumentation:
phase-level attribution of time and memory inside the
inference engine itself (KV cache management, attention,
sampling), which is where black-box profiling stops being
useful and engine-aware tooling has to take over.

inferscope is Apache-2.0,
[on GitHub](https://github.com/MicheleCampi/inferscope), and
v0.2.0 is the current release.
