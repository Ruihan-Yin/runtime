# Improve THP Utilization in CoreCLR GC (DOTNET_GCTHP)

## Context

`DOTNET_GCTHP` lets the GC hint the Linux kernel to back heap memory with 2MB
Transparent Huge Pages via `madvise(MADV_HUGEPAGE)`. In designed workloads THP
utilization is `<1%` of allocated memory.

**Root cause (from investigation — corrects the original premise):**
The THP hint is *already* wired into the whole commit path — `virtual_commit`
(gc.cpp:7447) routes GC-heap commits through `VirtualCommitThp`
(via `virtual_alloc_commit_for_heap`, gc.cpp:7438) and bookkeeping commits
through it directly (gc.cpp:7541). Low utilization is **not** a missing-call-site
problem. It comes from:

1. **Commit granularity is far below 2MB.** `VirtualCommitThp`
   (gcenv.unix.cpp:694) skips `madvise` when the commit is `< 2MB`. Real commits
   are 8KB (`SEGMENT_INITIAL_COMMIT`, gc.cpp:12444) or `max(demand, 64KB)`
   (`grow_heap_segment`, gc.cpp:15819-15821) — almost never ≥ 2MB. So the hint is
   virtually never issued, and the 2MB window is never fully populated, so the
   kernel can't form a huge page.
2. **Commit ranges are not 2MB-aligned.** Region *bases* are already 2MB-aligned
   for the default 2MB/4MB region sizes (`region_allocator::init`, gc.cpp:3897),
   but the 8KB initial commit offsets `heap_segment_committed` to `base+8KB`, so
   every subsequent commit starts unaligned. (1MB regions on small heaps can't
   hold a huge page at all and are out of scope.)

**Intended outcome:** raise THP-backed heap bytes well above 1% with bounded RSS
overshoot, gated entirely behind `use_thp_p`.

## Chosen approach (confirmed with user)

- **Strategy: Hybrid** — madvise the whole 2MB-aligned region VMA once at
  creation, *plus* eager 2MB-aligned commit for high-churn ephemeral/SOH regions
  that reliably fill.
- **RSS: Balanced** — eager-commit only where regions fill; keep page-granular
  decommit but snap the *retained* boundary up to 2MB (avoid splitting huge
  pages) under memory pressure.
- **Scope: Regions + bookkeeping** (card / brick / mark-array / seg-map).

## Implementation

### 1. OS helper: madvise-only hint (no commit)
`VirtualCommitThp` couples `mprotect` + `madvise`; for whole-region hinting we
need `madvise` alone (works on the reserved `PROT_NONE` VMA — sets `VM_HUGEPAGE`).
- Add `static bool GCToOSInterface::VirtualHugePageHint(void* address, size_t size)`
  - decl: [gcenv.os.h](../../../src/coreclr/gc/env/gcenv.os.h) near line 291 (next to `VirtualCommitThp`)
  - unix: [gcenv.unix.cpp](../../../src/coreclr/gc/unix/gcenv.unix.cpp) — `#ifdef MADV_HUGEPAGE` → `madvise(address, size, MADV_HUGEPAGE)`; ignore/log errno; no `mprotect`.
  - windows/other: [gcenv.windows.cpp](../../../src/coreclr/gc/windows/gcenv.windows.cpp) — no-op returning true.

### 2. Constants + helpers (gc.cpp / gcpriv.h)
- `GC_HUGE_PAGE_SIZE` = `2 * 1024 * 1024`.
- `align_on_huge_page` / `align_lower_huge_page` (mirror `align_on_page` at gc.cpp:2050).
- Eligibility predicate: `use_thp_p && gc_region_size >= GC_HUGE_PAGE_SIZE`
  (excludes 1MB regions, which guarantees 2MB-aligned bases).

### 3. Whole-region madvise at creation
- [gc.cpp](../../../src/coreclr/gc/gc.cpp) `make_heap_segment` (12441): after the initial
  commit, if eligible and `size >= GC_HUGE_PAGE_SIZE`, call
  `VirtualHugePageHint(new_pages, align_lower_huge_page(size))`. This sets the VMA
  flag so any fully-committed 2MB window can be collapsed/allocated as a huge page.

### 4. Eager 2MB-aligned commit for high-churn regions
- [gc.cpp](../../../src/coreclr/gc/gc.cpp) `grow_heap_segment` (15819): when eligible **and**
  the region is a high-churn OH (SOH / ephemeral — not UOH), round the commit
  *end* up to a huge-page boundary before the existing floor/cap:
  ```
  uint8_t* aligned_end = align_on_huge_page(high_address);
  c_size = aligned_end - heap_segment_committed(seg);   // then existing max(commit_min_th)/min(reserve)
  ```
  Because commits are contiguous from the 2MB-aligned base, `heap_segment_committed`
  now advances in 2MB steps → each 2MB window is fully accessible → fault-time
  huge page. **Accounting stays consistent**: the same `c_size` flows into
  `virtual_commit` (hard-limit accounting at gc.cpp:7480-7527) and
  `heap_segment_committed += c_size`, and decommit derives from
  `heap_segment_committed`, so commit/decommit remain symmetric.

### 5. Balanced decommit (avoid splitting huge pages)
- [gc.cpp](../../../src/coreclr/gc/gc.cpp) `decommit_heap_segment_pages_worker` (12633) and
  `decommit_heap_segment` (12663): when `use_thp_p` and eligible, snap the retained
  boundary *up* to a huge-page multiple — i.e. decommit only down to
  `align_on_huge_page(new_committed)` — so a partially-used huge page isn't split.
- **Correctness note:** `VirtualDecommit` re-maps with `MAP_FIXED` mmap
  (gcenv.unix.cpp:759), which **wipes `VM_HUGEPAGE`** on the decommitted range. On
  the next commit of that range, the existing per-commit `VirtualCommitThp` madvise
  re-establishes the hint — keep that path (its 2MB gate now passes for eager
  commits). No extra work needed for eager regions; lazy tails rely on
  re-madvise at recommit.

### 6. Bookkeeping tables (regions + bookkeeping scope)
- Card / brick / seg-map commit computation ([gc.cpp](../../../src/coreclr/gc/gc.cpp)
  9451-9585) and mark-array commit (`commit_mark_array_*`, 38560-38579): when
  `use_thp_p`, widen the computed `[commit_begin, commit_end)` to huge-page
  boundaries (`align_lower_huge_page` / `align_on_huge_page`) within the reserved
  bookkeeping bounds. These flow through the existing `h_number < 0`
  `VirtualCommitThp` path (gc.cpp:7541), which now issues `madvise` (≥2MB).
  Bookkeeping is contiguous, long-lived, low-churn → low overshoot risk.
- Verify the bookkeeping reservation base is 2MB-aligned; if not, request 2MB
  alignment where it is reserved (the `VirtualReserve` alignment arg already
  supports this — over-reserve + trim, gcenv.unix.cpp:566-579).

### 7. Docs
- Update [docs/design/features/gc-thp.md](gc-thp.md): the
  commit-path description (THP already covers all commits), the alignment/granularity
  root cause, and the new hybrid design. (Note: current doc shows `printf`; code
  uses `dprintf` under `_DEBUG` — reconcile.)

## Files to modify
- [src/coreclr/gc/gc.cpp](../../../src/coreclr/gc/gc.cpp) — helpers, `make_heap_segment`, `grow_heap_segment`, decommit workers, bookkeeping commit paths.
- [src/coreclr/gc/gcpriv.h](../../../src/coreclr/gc/gcpriv.h) — constants / helper / predicate decls.
- [src/coreclr/gc/env/gcenv.os.h](../../../src/coreclr/gc/env/gcenv.os.h), [unix/gcenv.unix.cpp](../../../src/coreclr/gc/unix/gcenv.unix.cpp), [windows/gcenv.windows.cpp](../../../src/coreclr/gc/windows/gcenv.windows.cpp) — `VirtualHugePageHint`.
- [docs/design/features/gc-thp.md](gc-thp.md).

## Verification
1. **Build:** build the coreclr GC (`./build.sh clr.gc` or subset appropriate to the tree).
2. **Precondition:** host in madvise mode
   (`cat /sys/kernel/mm/transparent_hugepage/enabled` shows `[madvise]`).
3. **Measure utilization** on an allocation-heavy workload (e.g. GCPerfSim or a
   simple gen0/LOH churn app) with `DOTNET_GCTHP=1`:
   - THP-backed bytes: `awk '/AnonHugePages/{s+=$2} END{print s}' /proc/<pid>/smaps`
     (or `/proc/<pid>/smaps_rollup`).
   - Heap committed / RSS: `VmRSS` in `/proc/<pid>/status`.
   - Ratio `AnonHugePages / committed heap` should be well above the current <1%.
4. **A/B:** same workload with `DOTNET_GCTHP=0` — confirm utilization returns to
   baseline and RSS overshoot with THP on is bounded (Balanced target).
5. **Correctness / no-regression:**
   - Run GC functional tests; run under `DOTNET_GCHeapHardLimit` to confirm the
     rounded commit sizes don't trigger spurious hard-limit OOM.
   - Server GC (`DOTNET_gcServer=1`, multiple heaps) — confirm no crashes and RSS
     overshoot stays bounded (eager commit is SOH/ephemeral-only).
   - Confirm memory is still returned after decommit (drive high memory load).

## Second-stage TODOs (deferred refinements)

- **Clamp eager commit to the gen0 allocation budget (SOH cooperation).**
  In `grow_heap_segment` for the ephemeral segment, cap the rounded-up
  `aligned_end` at the budget-derived high-water (e.g. `end_gen0_region_space`
  / `alloc_allocated + gen0 budget`) *before* rounding to a 2MB boundary. This
  prevents eager-committing a full huge page into space the GC will abandon
  before the next gen0 GC — the residual over-commit case when the per-cycle
  gen0 allocation into a region is `< 2MB` (small/constrained heaps, and
  multiplied across server-GC heaps). First stage relies on `region_size >= 2MB`
  gating + demand-driven growth (overshoot bounded to `<2MB` per *active*
  region) + Balanced decommit; this refinement tightens it further.

## Risks / mitigations
- **RSS overshoot** → eager commit limited to SOH/ephemeral + region_size≥2MB;
  balanced decommit.
- **`VM_HUGEPAGE` wiped by MAP_FIXED decommit** → per-commit `VirtualCommitThp`
  re-madvises on recommit.
- **Hard-limit accounting drift** → all 2MB rounding done in gc.cpp so
  commit/decommit stay symmetric; nothing rounded inside gcenv.
- **1MB-region / small heaps** → explicitly excluded (can't hold a huge page).
