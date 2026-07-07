# Transparent Huge Pages (THP) Implementation in .NET CoreCLR GC

## Overview

This document describes the implementation of Transparent Huge Pages (THP) support in the .NET CoreCLR Garbage Collector. THP allows the GC to use 2MB huge pages instead of standard 4KB pages for heap memory, potentially reducing TLB (Translation Lookaside Buffer) misses and improving memory access performance.


**Target Platform:** Linux x86_64 (also stubbed for Windows)  
**Runtime Version:** .NET 11.0.0

---

## System Requirements

### Linux Kernel Configuration

THP must be enabled in **madvise mode** on the system:

```bash
$ cat /sys/kernel/mm/transparent_hugepage/enabled
always [madvise] never
```
This shows the current THP policy (`always`, `madvise`, or `never`), with the active mode in brackets`[]`.  The `[madvise]` setting means applications must explicitly request THP via `madvise(MADV_HUGEPAGE)` system call.

THP can be enabled in the recommended `madvise` mode(This will not be part of the runtime. The GC merely checks if it is in this mode) using the following command:

`echo madvise | sudo tee /sys/kernel/mm/transparent_hugepage/enabled`


**Why madvise mode?**
- Applications opt-in to THP (prevents system-wide overhead)
- Kernel allocates 2MB huge pages only when requested

### Hardware Requirements

- **x86_64 architecture**: Native 2MB huge page support
- **CPU**: Modern processor with TLB support (all contemporary CPUs)

---

## GC Configuration

### Environment Variable

THP is controlled via the `DOTNET_GCTHP` environment variable:

```bash
# Enable THP
export DOTNET_GCTHP=1

# Disable THP (default)
export DOTNET_GCTHP=0
```

### Configuration Reading

**File:** [src/coreclr/gc/gcconfig.h](src/coreclr/gc/gcconfig.h)

```cpp
BOOL_CONFIG(DOTNET_GCTHP, ReadTHPEnabled, true, "Enable Transparent Huge Pages")
```

**Implementation:**
- Configuration key: `DOTNET_GCTHP`
- Default value: `false` (disabled by default)
- Read at GC initialization
- Stored in heap instance: `gc_heap::use_thp_p` member variable

**File:** [src/coreclr/gc/gcpriv.h](src/coreclr/gc/gcpriv.h)  
**Line:** 5370-5372

```cpp
class gc_heap
{
    BOOL use_thp_p;  // Whether THP is enabled for this heap
    // ...
};
```

---

## Implementation Architecture

### Memory Flow

The .NET GC uses a **two-phase memory model** on Unix systems: reserve address space first, then commit it on demand. This allows the GC to pre-allocate large virtual address ranges without consuming physical memory upfront. THP integrates into the commit phase by hinting to the kernel to use 2MB pages instead of standard 4KB pages when backing the committed memory with physical RAM.

**Phase 1 (Reserve):** The GC reserves a large contiguous virtual address range (e.g., 1GB for a heap segment) using `mmap(PROT_NONE)`. This creates entries in the process's page tables but does not allocate any physical memory. The reserved region is inaccessible—any attempted access triggers a segmentation fault.

**Phase 2 (Commit):** When the GC actually needs memory, it commits portions of the reserved range by calling `mprotect(PROT_READ|PROT_WRITE)` to make the region accessible. If THP is enabled (`DOTNET_GCTHP=1`) and the commit size is ≥2MB, the GC immediately follows with `madvise(MADV_HUGEPAGE)` to request that subsequent page faults use huge pages. The `madvise()` call is purely advisory—it tells the kernel "prefer 2MB pages for this range if possible," but the kernel makes the final allocation decision.

**Phase 3 (Page Fault / Physical Allocation):** Memory is not actually allocated until the application **writes** to a committed address. This triggers a page fault, and the kernel's page fault handler allocates the backing store:
- **Without THP:** Allocates one 4KB page per fault (512 faults needed for 2MB)
- **With THP (after `madvise()`):** Attempts to allocate a single 2MB huge page if the fault address is 2MB-aligned and contiguous memory is available. Falls back to 4KB pages if huge page allocation fails.

**Phase 4 (Decommit):** When the GC shrinks the heap or releases memory, it decommits regions by calling `mmap(PROT_NONE)` over the address range. This unmaps the physical memory and returns it to the OS, but keeps the virtual address reservation intact for future reuse. There is **no THP-specific cleanup**—the kernel automatically handles freeing huge pages the same way it handles normal pages.

**Key Insight:** THP operates **lazily**. The `madvise()` hint is given at commit time, but the actual huge page allocation happens later during page faults. This means you won't see `AnonHugePages` increase immediately after `VirtualCommitThp()` returns—only after the application has written to the committed memory.

**Memory Lifecycle Diagram:**

```
1. VirtualReserve()
   └─> mmap(PROT_NONE) - Reserve address space

2. VirtualCommit() / VirtualCommitThp()
   └─> mprotect(PROT_READ|PROT_WRITE) - Make memory accessible
   └─> [THP] madvise(MADV_HUGEPAGE) - Request huge pages

3. Page Fault
   └─> Kernel allocates 2MB huge page (if THP requested and feasible)

4. VirtualDecommit()
   └─> mmap(PROT_NONE) - Decommit memory (standard, no THP-specific cleanup)
```

### Core Implementation: VirtualCommitThp()

**File:** [src/coreclr/gc/unix/gcenv.unix.cpp](src/coreclr/gc/unix/gcenv.unix.cpp)  
**Lines:** 485-518

```cpp
bool GCToOSInterface::VirtualCommitThp(void* address, size_t size, uint16_t node)
{
    // First, perform standard commit (mprotect + NUMA binding)
    bool result = VirtualCommitInner(address, size, node, /* newMemory */ false);
    
    if (result)
    {
#ifdef MADV_HUGEPAGE
        // Only apply THP for allocations >= 2MB (one huge page)
        // Smaller allocations won't benefit and just add syscall overhead
        const size_t MIN_THP_SIZE = 2 * 1024 * 1024; // 2MB
        
        if (size >= MIN_THP_SIZE)
        {
            int rc = madvise(address, size, MADV_HUGEPAGE);
            if (rc == 0)
            {
                printf("THP: madvise(MADV_HUGEPAGE) succeeded for %p, size=%zu MB\n", 
                       address, size / (1024 * 1024));
            }
            else
            {
                printf("THP: madvise(MADV_HUGEPAGE) failed for %p, errno=%d\n", 
                       address, errno);
            }
        }
        else
        {
            printf("THP: Skipping madvise for small allocation %p, size=%zu KB (< 2MB threshold)\n", 
                   address, size / 1024);
        }
#endif
    }
    return result;
}
```

**Key Design Decisions:**

1. **2MB Minimum Threshold:**
   - Only allocations >= 2MB trigger `madvise(MADV_HUGEPAGE)`
   - Rationale: Huge pages are 2MB; smaller allocations cannot benefit
   - Avoids syscall overhead for small GC bookkeeping structures

2. **Syscall Order:**
   ```
   mprotect(PROT_READ|PROT_WRITE)  // Make memory accessible
   ↓
   mbind()                          // NUMA binding (optional)
   ↓
   madvise(MADV_HUGEPAGE)          // Request THP
   ```
   - `madvise()` must happen AFTER `mprotect()` because THP operates on accessible memory
   - Kernel allocates huge pages on subsequent page faults

3. **Windows Stub:**  
   **File:** [src/coreclr/gc/windows/gcenv.windows.cpp](src/coreclr/gc/windows/gcenv.windows.cpp)  
   **Lines:** 759-767
   ```cpp
   bool GCToOSInterface::VirtualCommitThp(void* address, size_t size, uint16_t node)
   {
       // Windows Large Pages require different mechanism (not implemented)
       return VirtualCommit(address, size, node);
   }
   ```

---

## Improving THP Utilization (2MB alignment & commit)

> **STATUS (post-evaluation).** The **coverage fix now lives in the OS abstraction
> layer**, not in `gc.cpp` — see [OS-layer THP region interposer](#os-layer-thp-region-interposer-shipped)
> below for the shipped design. The in-GC commit-side mechanisms described in this
> section (eager 2MB commit in `grow_heap_segment` §2, balanced decommit-snap §3,
> huge-backed free-region retention §5, and the gen0-budget clamp) were **REVERTED**
> and are kept here only for the design record. Reason: the **eager 2MB commit
> intermittently corrupts the heap under heavy compaction** — it advances
> `heap_segment_committed` in 2MB steps ahead of the allocation/plan frontier,
> violating a compaction invariant, which shows up as
> an intermittent infinite loop in `plan_phase`'s plug scan (bisected: eager-commit ON
> ⇒ ~40–50% hang on a high-survival workload; OFF ⇒ 0/18 runs hung). It also gave only
> a within-noise throughput effect. The sections below are kept for the design record;
> §2/§3/§5 and the clamp are **not in the tree**. If revisited, the supported path for
> *pinned* huge pages with correct accounting is `GCLargePages` (hugetlb), not
> extending the GC's own commits.

The per-commit `madvise` above is necessary but not sufficient: it is skipped for
commits `< 2MB`, and the kernel can only form a huge page when a full, 2MB-aligned
window is committed and populated. In practice GC commits are small (8KB initial
region commit, then `max(demand, 64KB)` steps), so the hint rarely fired and the
2MB windows were rarely fully populated — measured THP utilization was `<1%`.

To fix this, four cooperating mechanisms were added (all Unix + regions only, and
all gated on `use_thp_p`). Note THP is mutually exclusive with `GCLargePages`
(`use_thp_p = GetGCTHP() && ReadTHPEnabled() && !GetGCLargePages()`); large pages
pre-allocate/pin memory up front, whereas THP stays fully on-demand.

### Eligibility

`gc_heap::thp_region_eligible_p()` — true when `use_thp_p` and the region size is
`>= 2MB`. Region bases are aligned to the region size by `region_allocator::init`,
so a `>= 2MB` region size also guarantees **2MB-aligned region bases**. The 1MB
region size (small heaps) is excluded — it cannot hold a huge page.

### 1. Whole-region huge-page hint (`make_heap_segment`)

At region creation, after the initial commit, the GC issues a new advisory
**`GCToOSInterface::VirtualHugePageHint()`** over the entire region VMA
(`madvise(MADV_HUGEPAGE)` with no commit). This sets `VM_HUGEPAGE` on the whole
region so any 2MB window that later becomes fully committed can be backed by a
huge page — decoupling the hint from per-commit size.

### 2. Eager 2MB-aligned commit for SOH regions (`grow_heap_segment`)

For high-churn **SOH** regions (which fill linearly via a bump pointer), the
commit end is rounded up to a 2MB boundary (`align_on_huge_page`) before the
existing floor/cap. Because commits are contiguous from the 2MB-aligned region
base, `heap_segment_committed` advances in **2MB steps**, so each huge-page window
is fully accessible and the kernel can allocate a THP at fault time. UOH regions
keep demand-driven commit to avoid over-committing large, sparse regions.
Accounting stays consistent because the rounded `c_size` flows through
`virtual_commit` (hard-limit accounting) and `heap_segment_committed += c_size`,
and decommit derives from `heap_segment_committed`.

### 3. Balanced decommit (`decommit_heap_segment_pages_worker`)

When trimming a region's committed tail, the retained boundary is snapped **up**
to a 2MB boundary (`align_on_huge_page`). `VirtualDecommit` re-maps the
decommitted tail with `MAP_FIXED`, which would otherwise split the huge page whose
lower half is still in use. This retains at most one huge page beyond the trim
point. Note the `MAP_FIXED` remap also clears `VM_HUGEPAGE` on the decommitted
range; a later re-commit of that range re-establishes the hint via the per-commit
`VirtualCommitThp` path.

### 4. Bookkeeping tables (`make_card_table`)

The card table, brick table, mark array and seg-mapping table live in a single
contiguous reservation. That reservation is now **2MB-aligned** (via the
`VirtualReserve` alignment argument) and receives a one-shot
`VirtualHugePageHint()` over its whole range. These tables are large, contiguous
and long-lived, so their committed interior 2MB windows are good huge-page
candidates with low over-commit risk. (Per-element commit sizes are left
unchanged — the elements are packed with page-aligned boundaries, so widening
individual commits to 2MB would make adjacent elements contend for boundary
pages.)

### 5. Retain huge-backed free regions for reuse (`aged_region_p`)

Under a churny server workload, THP coverage plateaus not because the hint isn't
issued, but because huge pages are repeatedly **destroyed and re-formed**: freed
regions are decommitted (which clears `VM_HUGEPAGE` via the `MAP_FIXED` re-map),
and re-forming a huge page later has to compete for increasingly fragmented
physical memory, so it falls back to 4KB.

`get_free_region` already hands basic regions back **committed-first** (the free
list is kept committed-descending by `return_free_region`'s
`add_region_descending` and `sort_by_committed_and_age`), so reuse preserves huge
pages when a committed region is available. The gap was the **time-based decommit**
(`aged_region_p`): a fully-committed basic region idle for `AGE_IN_FREE_TO_DECOMMIT_BASIC`
GCs was decommitted even though keeping it would let a later allocation reuse it
with its huge pages intact.

The fix: when THP is eligible and we are **not** under memory pressure
(`!dt_high_memory_load_p() && !near_heap_hard_limit_p()`), a **fully-committed**
basic region is not aged out for time-based decommit. This is RSS-safe because
free basic regions in excess of the per-heap budget are still decommitted by the
budget path in `distribute_free_regions`, and under memory pressure normal aging
resumes so the memory is returned.

### New OS abstraction

**`GCToOSInterface::VirtualHugePageHint(void* address, size_t size)`** — issues
`madvise(MADV_HUGEPAGE)` only (no `mprotect`/commit), so it can be applied over a
whole reserved range. Advisory and best-effort: failure is ignored, and it is a
no-op on Windows and platforms without `MADV_HUGEPAGE`.

**Files:** [src/coreclr/gc/env/gcenv.os.h](src/coreclr/gc/env/gcenv.os.h),
[src/coreclr/gc/unix/gcenv.unix.cpp](src/coreclr/gc/unix/gcenv.unix.cpp),
[src/coreclr/gc/windows/gcenv.windows.cpp](src/coreclr/gc/windows/gcenv.windows.cpp).

> Note: the debug logging in `VirtualCommitThp` uses `dprintf(1, ...)` under
> `_DEBUG` (not `printf`) in the current implementation.

### Eager-commit clamp to the gen0 budget (second stage)

`grow_heap_segment` clamps the SOH eager commit to `heap_segment_decommit_target`,
the budget-derived committed high-water the GC expects for the tail ephemeral (gen0)
region. This prevents committing a full 2MB huge page into gen0 space that a GC will
abandon before it fills — over-commit that, under a hard limit, inflates committed
memory and triggers more frequent ephemeral GCs. The clamp never reduces the commit
below the actual allocation need, and non-tail / gen2 compaction regions have
`decommit_target == reserved`, so they keep the full eager commit (and thus the
huge-page benefit on the big compacting pauses, which is where the pause tail improves).

---

## OS-layer THP region interposer (shipped)

The in-GC commit-side approach above was reverted because widening the GC's own
commits corrupts the heap (it moves `heap_segment_committed` ahead of the plan
frontier). The shipped fix does the same 2MB widening **in the OS abstraction layer**
(`gcenv.unix.cpp`), *transparently to `gc.cpp`*. The GC keeps tracking
`heap_segment_committed` / commit accounting from its own **requested** sizes and
never reads back the actual OS mapping, so widening the underlying `mprotect` /
`mmap` is invisible to every invariant the in-GC version broke.

All behavior is gated on `use_thp_p` (Unix + regions, region size `>= 2MB`,
`!GCLargePages`) and applies only inside **registered** THP ranges.

### Why the coverage was ~0 before

A 1.5 GB managed array under `DOTNET_GCTHP=1` showed `AnonHugePages=0`. Two causes:
(a) the GC commits **sub-2MB `mprotect` ranges**, so no full 2MB window is ever
accessible at fault time; (b) the per-range `MADV_DONTDUMP`/`MADV_DODUMP` **split the
VMA mid-2MB**, and the kernel will not promote a THP unless a single VMA covers the
whole 2MB. khugepaged is far too slow (~16 MB / 10 s) to fix this in practice.

### 1. THP range registry

A small fixed set of `[start, end)` ranges plus a lock-free `thp_range_of(addr, size)`
lookup, populated once at GC init before concurrent activity via
**`GCToOSInterface::RegisterThpRange(address, size, widen_decommit)`**. Two coarse
reservations are registered:

- the **region range** `[g_gc_lowest_address, g_gc_highest_address)` in
  `initialize_gc`, with `widen_decommit = true` (region bases and ends are
  2MB-aligned because the region size is `>= 2MB`, so a widened decommit never crosses
  a region boundary into a neighbour's committed data);
- the **bookkeeping reservation** in `make_card_table`, with `widen_decommit = false`
  (the tables are packed with page-aligned element boundaries, so a widened decommit
  could clobber a neighbouring element still in use).

`RegisterThpRange` also issues the one-shot whole-range `madvise(MADV_HUGEPAGE)` so
any fully-committed 2MB window can be promoted.

### 2. Widen commit to 2MB — `VirtualCommitInner`

When `[addr, addr+size)` falls inside a registered range `R`, the `mprotect` is
widened up to the next 2MB boundary (clamped to `R.end`). Now every 2MB window the GC
touches is fully accessible, so the kernel forms a THP at fault time. `MADV_DODUMP` is
applied over the same widened range so its VMA boundary stays 2MB-aligned (no mid-2MB
split). The GC still tracks only the requested `[addr, size)` → no invariant broken.

### 3. Widen decommit END to 2MB — `VirtualDecommit`

For a range with `widen_decommit`, the decommit end is snapped **up** to the next 2MB
boundary (clamped to `R.end`). This releases the tail that step 2 widened (bounds RSS
overshoot to `< 2MB` per region) and only ever extends **above** the GC's new
committed point, so the `[used, committed)`-is-zero contract holds (a re-commit
re-zeroes via a fresh `MAP_FIXED` mmap). `MADV_DONTDUMP` is applied over the widened
range.

### 4. Avoid VMA splits / suppress `MADV_FREE`

`MADV_DODUMP` / `MADV_DONTDUMP` and NUMA `mbind` are all applied over the 2MB-widened
range so their VMA boundaries land on huge-page lines. `VirtualReset` (the `MADV_FREE`
path) is **skipped entirely** on registered ranges — `MADV_FREE` would let the kernel
reclaim individual base pages out of a huge page, fragmenting it; the reset is only an
optimization, not required for correctness.

### 5. Decommit is NOT suppressed

`VirtualDecommit` still returns memory to the OS (step 3 only widens the tail it
releases). RSS ≈ GC-committed rounded up to 2MB per active region and shrinks on GC
decommit, so it stays safe under a `DOTNET_GCHeapHardLimit` / cgroup. A future
opt-in "zero-in-place, keep THP across shrinks" mode would raise retention but pin RSS
at the high-water mark, so it is intentionally **not** in this design.

### Why this is safe (vs the reverted in-GC version)

- `heap_segment_committed` and the commit accounting are computed from `gc.cpp`'s
  requested sizes and never read the OS mapping → widening the `mprotect` underneath
  is invisible → the plan-phase clamp-to-committed and the `[used, committed)`-zero
  skip both see the *unchanged* committed → no corruption.
- Decommit only ever widens the **end** upward (never retains stale pages below the
  GC's committed) → the zero-on-recommit contract is preserved.
- Hard-limit enforcement uses only `gc.cpp` counters → unaffected.

### New OS abstraction

**`GCToOSInterface::RegisterThpRange(void* address, size_t size, bool widen_decommit)`**
— registers a coarse GC reservation as THP-eligible and issues the one-shot whole-range
huge-page hint. No-op on Windows and on platforms without `MADV_HUGEPAGE`.

**Files:** [src/coreclr/gc/env/gcenv.os.h](src/coreclr/gc/env/gcenv.os.h),
[src/coreclr/gc/unix/gcenv.unix.cpp](src/coreclr/gc/unix/gcenv.unix.cpp),
[src/coreclr/gc/windows/gcenv.windows.cpp](src/coreclr/gc/windows/gcenv.windows.cpp),
[src/coreclr/gc/gc.cpp](src/coreclr/gc/gc.cpp) (two registration calls).

### Verification findings (first activation)

Before the registration wiring above, `RegisterThpRange` was defined but never
declared/called, so the interposer had **never actually run**. First measurements on
a 30 GB box (server GC, Linux `enabled=madvise`, `defrag=madvise`):

- **Coverage — PASS.** TLB pointer-chase over a 1.5 GB gen2 array:
  `GCTHP=0 → AnonHugePages 0 MB, 265 ns/access`; `GCTHP=1 → 1550 MB, 243 ns/access`
  (full coverage, ~8% faster).
- **Correctness — no corruption.** The high-survival GCPerfSim config that hung the
  reverted in-GC version (`-sohsi 2 -lohar 0`) ran to completion under `GCTHP=1
  GCHeapVerify=1` with **0 hangs / 0 verify failures** and gen2 counts matching
  `GCTHP=0`.
- **Low footprint — PASS.** With no memory pressure (~4 GB RSS on the 30 GB box),
  `GCTHP=1` was *faster* (14.2 s vs 16.2 s) with *lower* RSS and comparable GC counts.
- **High footprint / high survival — REGRESSION.** When RSS approaches the box's
  available memory, `GCTHP=1` regressed **2.7–6.7×** (e.g. 367 s vs 55 s, no hard
  limit) with a **gen0-GC storm** (13 543 vs 153 gen0 GCs — gen0 budget collapses to
  ~2 MB) and **higher RSS** (17.8 GB vs 12.5 GB). Cause: the eager 2 MB commit faults
  whole huge-page windows in early, and `MADV_FREE` suppression keeps freed pages
  resident, so real memory load rises; the GC's high-memory-load heuristic
  (`dt_high_memory_load_p`, which reads system `MemAvailable`) then shrinks the gen0
  budget aggressively. This contradicts the "RSS-safe / self-limiting / no regression"
  assumption and is an **open design issue** — the widening (and/or `MADV_FREE`
  suppression) likely needs to back off under memory pressure, or be scoped away from
  short-lived ephemeral regions, before this is safe to enable by default.

### Options considered for the high-footprint regression

1. **Back off under memory pressure (IMPLEMENTED).** Gate the two RSS-inflating
   behaviours (commit widening and `MADV_FREE` suppression) on the GC's own pressure
   signal: skip them when `dt_high_memory_load_p() || near_heap_hard_limit_p()`. Keeps
   the coverage win when there is headroom, and reverts to plain page-granular
   behaviour when the system is tight. Decommit-widening stays on (it only releases
   memory, always RSS-safe).
2. **Scope widening away from ephemeral regions.** Only widen long-lived (gen2 / UOH)
   commits, so short-lived gen0 churn does not fault/retain 2MB windows. Needs the OS
   layer to learn each region's kind (extra registration state).
3. **Stop suppressing `MADV_FREE`** (or make it pressure-gated) to address only the
   RSS-retention half, leaving commit widening always on.
4. **Keep it strictly opt-in for dedicated, unpressured hosts** and document the
   constraint; it already helps there without further changes.

**Chosen: option 1.** Implemented via
**`GCToOSInterface::SetThpMemoryPressure(bool)`** — the GC calls it once per GC from
`distribute_free_regions` (a joined, single-threaded phase), and once at init to seed
the startup ramp. The backoff is asserted when
`joined_last_gc_before_oom || dt_high_memory_load_p() || near_heap_hard_limit_p()`
**or** any memory restriction is configured (`heap_hard_limit != 0 ||
is_restricted_physical_mem`). The interposer reads the flag on its hot paths and, when
set, stops widening commits and re-enables `MADV_FREE` in `VirtualReset`;
decommit-widening stays on (it only releases memory).

The memory-restriction term is essential: under a hard limit or container the GC sizes
the gen0 budget from a memory-load figure derived from **actual process RSS**
(`GetMemoryStatus`), which the eager 2 MB widening inflates. That feeds
`trim_youngest_desired` and collapses the gen0 budget into a GC storm *below* the 90%
high-memory-load threshold, so gating on `dt_high_memory_load_p()` alone is not enough
— the restriction itself must disable widening. Under a restriction THP therefore falls
back to the one-shot `MADV_HUGEPAGE` hint only (opportunistic promotion, no eager
widening), which keeps RSS bounded.

**Measured after the fix** (same box):
- Unrestricted TLB benchmark: full coverage retained (`AnonHugePages` 0 → 1550 MB),
  ~10% faster.
- 8 GB hard limit, high-survival GCPerfSim: `GCTHP=1` now matches `GCTHP=0`
  (≈62 s vs ≈61 s, RSS 8.52 GB both) — the storm is gone.
- Correctness: heap-verify + `GCTHP=1` under the hard limit ran clean (0 hangs,
  0 verify failures).

---
