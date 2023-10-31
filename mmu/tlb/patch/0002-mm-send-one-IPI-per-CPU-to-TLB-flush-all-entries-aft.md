```diff
From 72b252aed506b8f1a03f7abd29caef4cdf6a043b Mon Sep 17 00:00:00 2001
From: Mel Gorman <mgorman@suse.de>
Date: Fri, 4 Sep 2015 15:47:32 -0700
Subject: [PATCH] mm: send one IPI per CPU to TLB flush all entries after
 unmapping pages

An IPI is sent to flush remote TLBs when a page is unmapped that was
potentially accesssed by other CPUs.  There are many circumstances where
this happens but the obvious one is kswapd reclaiming pages belonging to a
running process as kswapd and the task are likely running on separate
CPUs.

> potentially : 可能的;潜在的
>
> 当页面取消映射时，需要发送ipi 来 flush remote TLBs, 这些TLBs 可能会被
> 其他的 CPUs 使用。有很多场景可以触发, 但是最明显的是 kswapd 回收属于
> 正在运行进程的页面, 因为 kswapd 和 该task 可能在单独的CPU 上运行。

On small machines, this is not a significant problem but as machine gets
larger with more cores and more memory, the cost of these IPIs can be
high.  This patch uses a simple structure that tracks CPUs that
potentially have TLB entries for pages being unmapped.  When the unmapping
is complete, the full TLB is flushed on the assumption that a refill cost
is lower than flushing individual entries.

> 在小型机器上，这不是一个大问题，但随着机器越来越大，拥有更多的内核和内存，
> 这些IPIs的成本可能会很高。此patch 使用一个简单的结构来跟踪可能具有未映射
> 页面的TLB条目的CPU。当 unmapping 完成时，假设重新填充成本低于刷新单个条目，
> 则会刷新 full TLB。

Architectures wishing to do this must give the following guarantee.

> 希望这样做的体系结构必须提供以下保证。

        If a clean page is unmapped and not immediately flushed, the
        architecture must guarantee that a write to that linear address
        from a CPU with a cached TLB entry will trap a page fault.

        > 如果一个干净的页面被 umapped 且未立即刷新，则体系结构必须保证
        > 带有该页面的cached TLB entry的 CPU 对这样的线性地址写入将会
        > trap一个page fault

This is essentially what the kernel already depends on but the window is
much larger with this patch applied and is worth highlighting.  The
architecture should consider whether the cost of the full TLB flush is
higher than sending an IPI to flush each individual entry.  An additional
architecture helper called flush_tlb_local is required.  It's a trivial
wrapper with some accounting in the x86 case.

trivial: /ˈtrɪviəl/ 琐碎的，不重要的
wrapper :封装

> 这基本上是内核已经依赖的，但应用了这个补丁后，窗口要大得多，值得强调。
> 体系结构应该考虑完整TLB刷新的成本是否高于发送IPI来刷新每个单独的条目。
> 需要一个名为flush_tlb_local的 额外的 architecture helper。在x86的情况
> 下，它是一个包含一些accounting 的 trivial wrapper

The impact of this patch depends on the workload as measuring any benefit
requires both mapped pages co-located on the LRU and memory pressure.  The
case with the biggest impact is multiple processes reading mapped pages
taken from the vm-scalability test suite.  The test case uses NR_CPU
readers of mapped files that consume 10*RAM.

> 此补丁的影响取决于工作负载，因为测量任何收益都需要将映射页面放在LRU上，
> 并需要内存压力。影响最大的情况是多个进程读取从 vm-scalability 测试套件
> 中获取的映射页面。测试用例使用NR_CPU 个 映射了 大概 10 * RAM file的 
> readers。

Linear mapped reader on a 4-node machine with 64G RAM and 48 CPUs

                                           4.2.0-rc1          4.2.0-rc1
                                             vanilla       flushfull-v7
Ops lru-file-mmap-read-elapsed      159.62 (  0.00%)   120.68 ( 24.40%)
Ops lru-file-mmap-read-time_range    30.59 (  0.00%)     2.80 ( 90.85%)
Ops lru-file-mmap-read-time_stddv     6.70 (  0.00%)     0.64 ( 90.38%)

           4.2.0-rc1    4.2.0-rc1
             vanilla flushfull-v7
User          581.00       611.43
System       5804.93      4111.76
Elapsed       161.03       122.12

This is showing that the readers completed 24.40% faster with 29% less
system CPU time.  From vmstats, it is known that the vanilla kernel was
interrupted roughly 900K times per second during the steady phase of the
test and the patched kernel was interrupts 180K times per second.

The impact is lower on a single socket machine.

                                           4.2.0-rc1          4.2.0-rc1
                                             vanilla       flushfull-v7
Ops lru-file-mmap-read-elapsed       25.33 (  0.00%)    20.38 ( 19.54%)
Ops lru-file-mmap-read-time_range     0.91 (  0.00%)     1.44 (-58.24%)
Ops lru-file-mmap-read-time_stddv     0.28 (  0.00%)     0.47 (-65.34%)

           4.2.0-rc1    4.2.0-rc1
             vanilla flushfull-v7
User           58.09        57.64
System        111.82        76.56
Elapsed        27.29        22.55

It's still a noticeable improvement with vmstat showing interrupts went
from roughly 500K per second to 45K per second.

The patch will have no impact on workloads with no memory pressure or have
relatively few mapped pages.  It will have an unpredictable impact on the
workload running on the CPU being flushed as it'll depend on how many TLB
entries need to be refilled and how long that takes.  Worst case, the TLB
will be completely cleared of active entries when the target PFNs were not
resident at all.

[sasha.levin@oracle.com: trace tlb flush after disabling preemption in try_to_unmap_flush]
Signed-off-by: Mel Gorman <mgorman@suse.de>
Reviewed-by: Rik van Riel <riel@redhat.com>
Cc: Dave Hansen <dave.hansen@intel.com>
Acked-by: Ingo Molnar <mingo@kernel.org>
Cc: Linus Torvalds <torvalds@linux-foundation.org>
Signed-off-by: Sasha Levin <sasha.levin@oracle.com>
Cc: Michal Hocko <mhocko@suse.com>
Signed-off-by: Andrew Morton <akpm@linux-foundation.org>
Signed-off-by: Linus Torvalds <torvalds@linux-foundation.org>
---
 arch/x86/Kconfig                |   1 +
 arch/x86/include/asm/tlbflush.h |   6 ++
 include/linux/rmap.h            |   3 +
 include/linux/sched.h           |  16 +++++
 init/Kconfig                    |  10 +++
 mm/internal.h                   |  11 ++++
 mm/rmap.c                       | 104 +++++++++++++++++++++++++++++++-
 mm/vmscan.c                     |  23 ++++++-
 8 files changed, 172 insertions(+), 2 deletions(-)

diff --git a/arch/x86/Kconfig b/arch/x86/Kconfig
index 48f7433dac6f..117e2f373e50 100644
--- a/arch/x86/Kconfig
+++ b/arch/x86/Kconfig
@@ -41,6 +41,7 @@ config X86
 	select ARCH_USE_CMPXCHG_LOCKREF		if X86_64
 	select ARCH_USE_QUEUED_RWLOCKS
 	select ARCH_USE_QUEUED_SPINLOCKS
+	select ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH if SMP
 	select ARCH_WANTS_DYNAMIC_TASK_STRUCT
 	select ARCH_WANT_FRAME_POINTERS
 	select ARCH_WANT_IPC_PARSE_VERSION	if X86_32
diff --git a/arch/x86/include/asm/tlbflush.h b/arch/x86/include/asm/tlbflush.h
index cd791948b286..6df2029405a3 100644
--- a/arch/x86/include/asm/tlbflush.h
+++ b/arch/x86/include/asm/tlbflush.h
@@ -261,6 +261,12 @@ static inline void reset_lazy_tlbstate(void)
 
 #endif	/* SMP */
 
+/* Not inlined due to inc_irq_stat not being defined yet */
+#define flush_tlb_local() {		\
+	inc_irq_stat(irq_tlb_count);	\
+	local_flush_tlb();		\
+}
+
 #ifndef CONFIG_PARAVIRT
 #define flush_tlb_others(mask, mm, start, end)	\
 	native_flush_tlb_others(mask, mm, start, end)
diff --git a/include/linux/rmap.h b/include/linux/rmap.h
index c89c53a113a8..29446aeef36e 100644
--- a/include/linux/rmap.h
+++ b/include/linux/rmap.h
@@ -89,6 +89,9 @@ enum ttu_flags {
 	TTU_IGNORE_MLOCK = (1 << 8),	/* ignore mlock */
 	TTU_IGNORE_ACCESS = (1 << 9),	/* don't age */
 	TTU_IGNORE_HWPOISON = (1 << 10),/* corrupted page is recoverable */
+	TTU_BATCH_FLUSH = (1 << 11),	/* Batch TLB flushes where possible
+					 * and caller guarantees they will
+					 * do a final flush if necessary */
 };
 
 #ifdef CONFIG_MMU
diff --git a/include/linux/sched.h b/include/linux/sched.h
index 119823decc46..3c602c20c717 100644
--- a/include/linux/sched.h
+++ b/include/linux/sched.h
@@ -1344,6 +1344,18 @@ enum perf_event_task_context {
 	perf_nr_task_contexts,
 };
 
+/* Track pages that require TLB flushes */
+struct tlbflush_unmap_batch {
+	/*
+	 * Each bit set is a CPU that potentially has a TLB entry for one of
+	 * the PFNs being flushed. See set_tlb_ubc_flush_pending().
+	 */
+	struct cpumask cpumask;
+
+	/* True if any bit in cpumask is set */
+	bool flush_required;
+};
+
 struct task_struct {
 	volatile long state;	/* -1 unrunnable, 0 runnable, >0 stopped */
 	void *stack;
@@ -1700,6 +1712,10 @@ struct task_struct {
 	unsigned long numa_pages_migrated;
 #endif /* CONFIG_NUMA_BALANCING */
 
+#ifdef CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH
+	struct tlbflush_unmap_batch tlb_ubc;
+#endif
+
 	struct rcu_head rcu;
 
 	/*
diff --git a/init/Kconfig b/init/Kconfig
index 161acd8bc56f..cf7e4824c8d0 100644
--- a/init/Kconfig
+++ b/init/Kconfig
@@ -882,6 +882,16 @@ config GENERIC_SCHED_CLOCK
 config ARCH_SUPPORTS_NUMA_BALANCING
 	bool
 
+#
+# For architectures that prefer to flush all TLBs after a number of pages
+# are unmapped instead of sending one IPI per page to flush. The architecture
+# must provide guarantees on what happens if a clean TLB cache entry is
+# written after the unmap. Details are in mm/rmap.c near the check for
+# should_defer_flush. The architecture should also consider if the full flush
+# and the refill costs are offset by the savings of sending fewer IPIs.
+config ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH
+	bool
+
 #
 # For architectures that know their GCC __int128 support is sound
 #
diff --git a/mm/internal.h b/mm/internal.h
index 36b23f1e2ca6..bd6372ac5f7f 100644
--- a/mm/internal.h
+++ b/mm/internal.h
@@ -426,4 +426,15 @@ unsigned long reclaim_clean_pages_from_list(struct zone *zone,
 #define ALLOC_CMA		0x80 /* allow allocations from CMA areas */
 #define ALLOC_FAIR		0x100 /* fair zone allocation */
 
+enum ttu_flags;
+struct tlbflush_unmap_batch;
+
+#ifdef CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH
+void try_to_unmap_flush(void);
+#else
+static inline void try_to_unmap_flush(void)
+{
+}
+
+#endif /* CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH */
 #endif	/* __MM_INTERNAL_H */
diff --git a/mm/rmap.c b/mm/rmap.c
index 171b68768df1..326d5d89e45c 100644
--- a/mm/rmap.c
+++ b/mm/rmap.c
@@ -62,6 +62,8 @@
 
 #include <asm/tlbflush.h>
 
+#include <trace/events/tlb.h>
+
 #include "internal.h"
 
 static struct kmem_cache *anon_vma_cachep;
@@ -583,6 +585,89 @@ vma_address(struct page *page, struct vm_area_struct *vma)
 	return address;
 }
 
+#ifdef CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH
+static void percpu_flush_tlb_batch_pages(void *data)
+{
+	/*
+	 * All TLB entries are flushed on the assumption that it is
+	 * cheaper to flush all TLBs and let them be refilled than
+	 * flushing individual PFNs. Note that we do not track mm's
+	 * to flush as that might simply be multiple full TLB flushes
+	 * for no gain.
+	 */
+	count_vm_tlb_event(NR_TLB_REMOTE_FLUSH_RECEIVED);
+	flush_tlb_local();
+}
+
+/*
+ * Flush TLB entries for recently unmapped pages from remote CPUs. It is
+ * important if a PTE was dirty when it was unmapped that it's flushed
+ * before any IO is initiated on the page to prevent lost writes. Similarly,
+ * it must be flushed before freeing to prevent data leakage.
+ */
+void try_to_unmap_flush(void)
+{
+	struct tlbflush_unmap_batch *tlb_ubc = &current->tlb_ubc;
+	int cpu;
+
+	if (!tlb_ubc->flush_required)
+		return;
+
+	cpu = get_cpu();
+
+	trace_tlb_flush(TLB_REMOTE_SHOOTDOWN, -1UL);
+
+	if (cpumask_test_cpu(cpu, &tlb_ubc->cpumask))
+		percpu_flush_tlb_batch_pages(&tlb_ubc->cpumask);
+
+	if (cpumask_any_but(&tlb_ubc->cpumask, cpu) < nr_cpu_ids) {
+		smp_call_function_many(&tlb_ubc->cpumask,
+			percpu_flush_tlb_batch_pages, (void *)tlb_ubc, true);
+	}
+	cpumask_clear(&tlb_ubc->cpumask);
+	tlb_ubc->flush_required = false;
+	put_cpu();
+}
+
+static void set_tlb_ubc_flush_pending(struct mm_struct *mm,
+		struct page *page)
+{
+	struct tlbflush_unmap_batch *tlb_ubc = &current->tlb_ubc;
+
+	cpumask_or(&tlb_ubc->cpumask, &tlb_ubc->cpumask, mm_cpumask(mm));
+	tlb_ubc->flush_required = true;
+}
+
+/*
+ * Returns true if the TLB flush should be deferred to the end of a batch of
+ * unmap operations to reduce IPIs.
+ */
+static bool should_defer_flush(struct mm_struct *mm, enum ttu_flags flags)
+{
+	bool should_defer = false;
+
+	if (!(flags & TTU_BATCH_FLUSH))
+		return false;
+
+	/* If remote CPUs need to be flushed then defer batch the flush */
+	if (cpumask_any_but(mm_cpumask(mm), get_cpu()) < nr_cpu_ids)
+		should_defer = true;
+	put_cpu();
+
+	return should_defer;
+}
+#else
+static void set_tlb_ubc_flush_pending(struct mm_struct *mm,
+		struct page *page)
+{
+}
+
+static bool should_defer_flush(struct mm_struct *mm, enum ttu_flags flags)
+{
+	return false;
+}
+#endif /* CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH */
+
 /*
  * At what user virtual address is page expected in vma?
  * Caller should check the page is actually part of the vma.
@@ -1220,7 +1305,24 @@ static int try_to_unmap_one(struct page *page, struct vm_area_struct *vma,
 
 	/* Nuke the page table entry. */
 	flush_cache_page(vma, address, page_to_pfn(page));
-	pteval = ptep_clear_flush(vma, address, pte);
+	if (should_defer_flush(mm, flags)) {
+		/*
+		 * We clear the PTE but do not flush so potentially a remote
+		 * CPU could still be writing to the page. If the entry was
+		 * previously clean then the architecture must guarantee that
+		 * a clear->dirty transition on a cached TLB entry is written
+		 * through and traps if the PTE is unmapped.
+		 */
+		pteval = ptep_get_and_clear(mm, address, pte);
+
+		/* Potentially writable TLBs must be flushed before IO */
+		if (pte_dirty(pteval))
+			flush_tlb_page(vma, address);
+		else
+			set_tlb_ubc_flush_pending(mm, page);
+	} else {
+		pteval = ptep_clear_flush(vma, address, pte);
+	}
 
 	/* Move the dirty bit to the physical page now the pte is gone. */
 	if (pte_dirty(pteval))
diff --git a/mm/vmscan.c b/mm/vmscan.c
index 8286938c70de..99ec00d6a5dd 100644
--- a/mm/vmscan.c
+++ b/mm/vmscan.c
@@ -1057,7 +1057,8 @@ static unsigned long shrink_page_list(struct list_head *page_list,
 		 * processes. Try to unmap it here.
 		 */
 		if (page_mapped(page) && mapping) {
-			switch (try_to_unmap(page, ttu_flags)) {
+			switch (try_to_unmap(page,
+					ttu_flags|TTU_BATCH_FLUSH)) {
 			case SWAP_FAIL:
 				goto activate_locked;
 			case SWAP_AGAIN:
@@ -1208,6 +1209,7 @@ static unsigned long shrink_page_list(struct list_head *page_list,
 	}
 
 	mem_cgroup_uncharge_list(&free_pages);
+	try_to_unmap_flush();
 	free_hot_cold_page_list(&free_pages, true);
 
 	list_splice(&ret_pages, page_list);
@@ -2151,6 +2153,23 @@ static void get_scan_count(struct lruvec *lruvec, int swappiness,
 	}
 }
 
+#ifdef CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH
+static void init_tlb_ubc(void)
+{
+	/*
+	 * This deliberately does not clear the cpumask as it's expensive
+	 * and unnecessary. If there happens to be data in there then the
+	 * first SWAP_CLUSTER_MAX pages will send an unnecessary IPI and
+	 * then will be cleared.
+	 */
+	current->tlb_ubc.flush_required = false;
+}
+#else
+static inline void init_tlb_ubc(void)
+{
+}
+#endif /* CONFIG_ARCH_WANT_BATCHED_UNMAP_TLB_FLUSH */
+
 /*
  * This is a basic per-zone page freer.  Used by both kswapd and direct reclaim.
  */
@@ -2185,6 +2204,8 @@ static void shrink_lruvec(struct lruvec *lruvec, int swappiness,
 	scan_adjusted = (global_reclaim(sc) && !current_is_kswapd() &&
 			 sc->priority == DEF_PRIORITY);
 
+	init_tlb_ubc();
+
 	blk_start_plug(&plug);
 	while (nr[LRU_INACTIVE_ANON] || nr[LRU_ACTIVE_FILE] ||
 					nr[LRU_INACTIVE_FILE]) {
-- 
2.41.0
```
