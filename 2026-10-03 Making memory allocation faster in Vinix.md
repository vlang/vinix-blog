Memory allocation in Vinix has become faster at two levels: inside the kernel,
and in the libc allocator used by applications. In our latest matched x86-64
QEMU runs, Vinix finished ahead of macOS Catalina on all six user-space and
syscall workloads, in both repeats and when every recorded sample was pooled.

The work began with the kernel's zeroing and poisoning loops, then moved through
mapping costs, bounded allocator reuse, and the ordinary `malloc`/`free` path.
Here are the before-and-after measurements, including comparisons with an actual
Catalina guest running XNU. They describe these emulated workloads, rather than
native hardware or every allocation an application might make.

## First, the kernel allocator

The kernel benchmark calls Vinix's kernel `malloc`/`free` and XNU's exported
`kern_os_malloc`/`kern_os_free` from a diagnostic kernel extension. Both sides
must supply zeroed memory, and the untimed warmup checks every requested byte.
Timed samples write and verify the payload endpoints.

Our first changes concentrated on filling memory. We capture the zeroing and
poisoning word counts before entering the loops, so the generated code does not
keep reloading aliased global page geometry on every store. Freeing a validated,
16-byte-aligned slab slot writes whole 64-bit words instead of going through a
generic byte fill. Zero-on-allocation and `0xaa` poisoning on free stay enabled,
along with poison checks, locks, reference counts and slot publication ordering.

These are **pooled median serialized TSC ticks per allocation/free pair**, with
15 samples per cell from three cohorts. Lower is better. The ticks include QEMU
execution and host scheduling; they are not native CPU-cycle measurements.

| Kernel workload | Original Vinix | Optimized Vinix | Catalina XNU | Vinix time reduction | Optimized Vinix/XNU |
| --- | ---: | ---: | ---: | ---: | ---: |
| 64 B reuse | 3,844.790 | 3,032.250 | 13,532.760 | 21.13% | 0.224× |
| 256 live objects, mixed slab classes | 18,763.184 | 10,634.521 | 27,638.265 | 43.32% | 0.385× |
| 256 KiB heap allocation | 1,756,093.750 | 805,585.938 | 8,818,304.688 | 54.13% | 0.091× |

The phases use 100,000, 12,288 and 128 pairs per sample respectively. The mixed
phase covers all fourteen Vinix slab classes, from 16 to 2,048 bytes, in 48
batches of 256 objects. We kept all samples and outliers.

The original Vinix kernel's pooled medians were already below XNU's in these
workloads. The optimization still removed 21–54% of its measured cost. There is
an execution-context difference: Vinix runs the sampler on the boot CPU before
its scheduler starts, while Catalina runs the loaded extension with its
scheduler and interrupts active. Vinix uses fresh boots; Catalina uses three
loads of the extension in one boot. Those differences and the wide raw ranges
matter when interpreting the ratios.

The [kernel report, raw ranges and per-cohort tables](https://github.com/vlang/vinix/blob/c16eb3f85a303f2b690b6ca351d3d4fef80e6d36/tests/alloc-bench/results/2026-10-02/README.md)
and the [shared C sampler](https://github.com/vlang/vinix/blob/c16eb3f85a303f2b690b6ca351d3d4fef80e6d36/kernel/c/heap_benchmark.c)
preserve the exact workload and build provenance.

## Measuring what applications pay

An application's `malloc` call usually goes through libc before it reaches the
kernel. Vinix uses musl's mallocng allocator; Catalina uses Apple's libmalloc.
The `malloc` rows below compare those allocators and their interaction with the
OS. The `mmap` rows include XNU or Vinix mapping, fault and teardown work, and the
pipe row includes system calls and kernel objects. The direct XNU kernel-heap
comparison is the table above.

We also had to fix the clock before trusting these timings. Vinix's old x86
monotonic clock counted received PIT interrupts and could lose elapsed time
under TCG or long kernel operations. It now reads a free-running hardware
counter. The original user-space captures remain archived, but their
nanosecond figures cannot serve as a reliable before-and-after baseline. The
serialized-TSC kernel measurements above are independent of that bug.

For the application comparison, the **before** table is the last complete
corrected-clock campaign before our final malloc optimization, called V5. It
already includes the preceding kernel and bounded-retention work. The **after**
table is the complete V6 campaign. These are separate historical campaigns,
with a fresh Catalina reference in each; they do not isolate the causal effect
of one code change. The kernel is identical between V5 and V6.

All values are **nanoseconds per operation pair**, pooled from all 14 raw
samples per workload and guest. A pair means allocate/free, map/unmap, or
create/close. `Vinix/Catalina` below one means Vinix took less time. Displayed
figures are rounded; validation uses the raw observations.

### Before the final malloc optimization: V5

| Workload | Vinix ns/pair | Catalina ns/pair | Vinix/Catalina |
| --- | ---: | ---: | ---: |
| 64 B malloc/free | 550.070 | 473.913 | 1.161× |
| Mixed malloc, batches of 64 | 491.208 | 838.058 | 0.586× |
| 256 KiB malloc/touch/free | 1,954.100 | 15,955.000 | 0.122× |
| 4 KiB anonymous mmap/munmap | 15,099.650 | 25,475.250 | 0.593× |
| 256 KiB mmap/touch/munmap | 862,624.450 | 1,668,026.400 | 0.517× |
| Pipe create/close | 28,820.900 | 49,255.200 | 0.585× |

V5 still missed the target: hot 64-byte allocation was slower in the second
cohort and in pooled samples. The other five workloads passed both cohorts.
We preserved the [complete V5 comparison](https://github.com/vlang/vinix/blob/c16eb3f85a303f2b690b6ca351d3d4fef80e6d36/tests/alloc-bench/results/2026-10-03-userspace-v5/README.md),
including the slower capture, and continued working on the ordinary allocation
path.

### After the final malloc optimization: V6

| Workload | Vinix ns/pair | Catalina ns/pair | Vinix/Catalina |
| --- | ---: | ---: | ---: |
| 64 B malloc/free | 228.850 | 387.653 | 0.590× |
| Mixed malloc, batches of 64 | 313.803 | 777.990 | 0.403× |
| 256 KiB malloc/touch/free | 1,537.300 | 18,458.800 | 0.083× |
| 4 KiB anonymous mmap/munmap | 11,432.800 | 27,426.500 | 0.417× |
| 256 KiB mmap/touch/munmap | 669,947.750 | 1,924,765.850 | 0.348× |
| Pipe create/close | 29,889.350 | 70,205.850 | 0.426× |

All six workloads now pass in **both individual cohorts and pooled results**.
The final hot-allocation comparison is about 229 ns/pair against Catalina's
388 ns/pair when pooled. The other pooled ratios range from 0.083 to 0.426.
The pipe median rose slightly from V5 to V6 even though its final Catalina
comparison passes; this is one reason to keep each campaign's reference
alongside its Vinix results and avoid calling every change a measured speedup.

The touched workloads write and verify one byte per 4 KiB page. They do not
read and write the entire 256 KiB. The mixed user workload holds up to 64
objects across sizes from 16 bytes to 16 KiB, then frees them in reverse order.

## What changed between the kernel and malloc

A faster kernel heap does not remove the cost of repeatedly creating and
tearing down mappings. We added bounded reuse to musl: eligible ordinary
classes can keep one free group with at most 128 KiB of backing, and five
direct-map buckets have upper bounds totaling 3,968 KiB. A conservative bound
including nested groups keeps the additional retained backing below 16 MiB;
live allocations and upstream fragmentation are outside that bound. `malloc_trim`
can release the retained storage, and dirty reuse still has to satisfy `calloc`'s
zeroing contract.

Earlier kernel work also moved private anonymous x86 mappings to first-touch
allocation, deferred a pipe's data buffer until its first nonempty write, and
reduced page-table teardown work. Empty tables must still be detached and
completed TLB invalidation must precede physical reclamation. Mapping and pipe
creation include those policies as well as allocator speed.

The final change gives ordinary single-thread allocation a short path. When
there is an active size class with an available slot, and both the thread state
and actual malloc lock are zero, `malloc` can consume that slot directly. A sole
group can also refill from freed slots that are already active. Requests that
need mapping, page activation, group traversal or lock handling go through the
original allocator body. The path works across ordinary size classes; it does
not recognize the benchmark or special-case its 64-byte request.

Freeing now reuses the index and stride already computed while validating the
allocation's metadata. It avoids repeating that work while preserving the
alignment, secret, bounds, live-slot and redzone checks before publishing a
freed slot. Slot rotation and address cycling remain. Direct mappings also
receive stronger shape checks before stride use.

Independent review checked the final source, 24 static/PIC allocator objects
and twelve corresponding static-archive members. On x86 the warm malloc wrapper
saves two registers and needs no stack frame; on ARM it also needs no stack
frame. The new path reduces repeated bookkeeping and calls while keeping the existing
checks and fallback behavior.

The [allocator change](https://github.com/vlang/vinix/commit/ba930704134ba6ea1d44bc2b19e9e72b3b5b1cb3)
and [final musl patch](https://github.com/vlang/vinix/blob/c16eb3f85a303f2b690b6ca351d3d4fef80e6d36/build-support/musl/malloc-retain.patch)
show the implementation.

## Reproducing the comparison

The final protocol was fixed before its artifacts and timings: fresh captures
in **Vinix, Catalina, Catalina, Vinix** order. Each workload has a full warmup
and seven measured samples. Hot and mixed malloc use 200,000 pairs per sample;
the other four workloads use 10,000. Every one of the 168 raw samples remains
in the result, including outliers. The acceptance rule required all six medians
to be no slower in each cohort and when pooling all samples.

The common setup is QEMU 11.1.1, x86-64, `q35,vmport=off`, 4 GiB RAM and
`tcg,thread=single,tb-size=1024`, with
`Penryn,kvm=on,vendor=GenuineIntel,+ssse3,+sse4.2,+popcnt`. The kernel sampler
uses one vCPU; the user-space sampler uses two. Firmware, disks and OS
peripherals differ, and other sessions continued using the host.

The macOS guest is **Catalina 10.15.7, Darwin 19.6.0, XNU 6153.141.2.2~1**.
This is a comparison with that installation, not a measurement of current
macOS. The newer XNU source tree available locally is not the guest kernel.

The user-space C benchmark is compiled with genuine GNU GCC in both guests:
14.2.0 on Vinix and
14.3.0 on Catalina. On Catalina, guest GCC produces assembly and host Apple
tools assemble and link it for macOS 10.15 using the MacOSX15.4 SDK. The
resulting binary runs in the guest. Both user compilations use
`-std=c11 -O2 -Wall -Wextra -Werror -fno-builtin`; GCC minor versions and the
linking arrangement remain qualifications on the comparison.

You can recompute the final user-space results from the preserved records in the Vinix
checkout without booting either VM:

```sh
python3 tests/alloc-bench/results/2026-10-03-userspace-v6/recompute.py
```

The [full final report](https://github.com/vlang/vinix/blob/c16eb3f85a303f2b690b6ca351d3d4fef80e6d36/tests/alloc-bench/results/2026-10-03-userspace-v6/README.md)
includes every sample, per-cohort tables, hashes, commands and the preceding
failed campaigns. Emulated execution and host-load variation limit precise
ratios; these samples do not establish native performance or exhaustive
multithreaded behavior.

## Keeping the allocator correct

The final allocator passed **262,816 checks** across eight configurations:
x86-64 and ARM64, retained and disabled policies, and dynamic and static linkage.
Tests cover class boundaries, live-object bursts, dirty reuse, trimming, thread
transitions and allocation inside all three public atfork callbacks. All 32
corruption children terminated by a signal, and all eight post-trim mapping
footprint deltas returned to zero. That measures mapped storage, not physical
RAM use.

Three core VM configurations passed, including four-level and five-level x86
page tables and ARM's second persistent boot. The desktop harness completed
all 43 required measurements across ops, churn, cache, idle, apps and drag.
Those logs also preserve existing kernel memory growth, and the allocation-site
allowlist audit still has baseline failures. We do not turn those into passing
leak checks. The performance result is narrower and concrete: less measured
kernel allocation work, and a complete user-space campaign that now meets its
Catalina comparison target with every sample retained.
