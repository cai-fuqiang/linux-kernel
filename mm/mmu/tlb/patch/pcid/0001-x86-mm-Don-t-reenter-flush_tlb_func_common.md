```diff
From bc0d5a89fbe3c83ac45438d7ba88309f4713615d Mon Sep 17 00:00:00 2001
From: Andy Lutomirski <luto@kernel.org>
Date: Thu, 29 Jun 2017 08:53:13 -0700
Subject: [PATCH 1/2] x86/mm: Don't reenter flush_tlb_func_common()

It was historically possible to have two concurrent TLB flushes
targetting the same CPU: one initiated locally and one initiated
remotely.  This can now cause an OOPS in leave_mm() at
arch/x86/mm/tlb.c:47:

        if (this_cpu_read(cpu_tlbstate.state) == TLBSTATE_OK)
                BUG();

with this call trace:
 flush_tlb_func_local arch/x86/mm/tlb.c:239 [inline]
 flush_tlb_mm_range+0x26d/0x370 arch/x86/mm/tlb.c:317

Without reentrancy, this OOPS is impossible: leave_mm() is only
called if we're not in TLBSTATE_OK, but then we're unexpectedly
in TLBSTATE_OK in leave_mm().

This can be caused by flush_tlb_func_remote() happening between
the two checks and calling leave_mm(), resulting in two consecutive
leave_mm() calls on the same CPU with no intervening switch_mm()
calls.

We never saw this OOPS before because the old leave_mm()
implementation didn't put us back in TLBSTATE_OK, so the assertion
didn't fire.

Nadav noticed the reentrancy issue in a different context, but
neither of us realized that it caused a problem yet.

Reported-by: Levin, Alexander (Sasha Levin) <alexander.levin@verizon.com>
Signed-off-by: Andy Lutomirski <luto@kernel.org>
Reviewed-by: Nadav Amit <nadav.amit@gmail.com>
Reviewed-by: Thomas Gleixner <tglx@linutronix.de>
Cc: Andrew Morton <akpm@linux-foundation.org>
Cc: Arjan van de Ven <arjan@linux.intel.com>
Cc: Borislav Petkov <bp@alien8.de>
Cc: Dave Hansen <dave.hansen@intel.com>
Cc: Linus Torvalds <torvalds@linux-foundation.org>
Cc: Mel Gorman <mgorman@suse.de>
Cc: Peter Zijlstra <peterz@infradead.org>
Cc: Rik van Riel <riel@redhat.com>
Cc: linux-mm@kvack.org
Fixes: 3d28ebceaffa ("x86/mm: Rework lazy TLB to track the actual loaded mm")
Link: http://lkml.kernel.org/r/855acf733268d521c9f2e191faee2dcc23a29729.1498751203.git.luto@kernel.org
Signed-off-by: Ingo Molnar <mingo@kernel.org>
---
 arch/x86/mm/tlb.c | 17 +++++++++++++++--
 1 file changed, 15 insertions(+), 2 deletions(-)

diff --git a/arch/x86/mm/tlb.c b/arch/x86/mm/tlb.c
index b2485d69f7c2..1cc47838d1e8 100644
--- a/arch/x86/mm/tlb.c
+++ b/arch/x86/mm/tlb.c
@@ -192,6 +192,9 @@ void switch_mm_irqs_off(struct mm_struct *prev, struct mm_struct *next,
 static void flush_tlb_func_common(const struct flush_tlb_info *f,
 				  bool local, enum tlb_flush_reason reason)
 {
+	/* This code cannot presently handle being reentered. */
+	VM_WARN_ON(!irqs_disabled());
+
 	if (this_cpu_read(cpu_tlbstate.state) != TLBSTATE_OK) {
 		leave_mm(smp_processor_id());
 		return;
@@ -297,8 +300,13 @@ void flush_tlb_mm_range(struct mm_struct *mm, unsigned long start,
 		info.end = TLB_FLUSH_ALL;
 	}
 
-	if (mm == this_cpu_read(cpu_tlbstate.loaded_mm))
+	if (mm == this_cpu_read(cpu_tlbstate.loaded_mm)) {
+		VM_WARN_ON(irqs_disabled());
+		local_irq_disable();
 		flush_tlb_func_local(&info, TLB_LOCAL_MM_SHOOTDOWN);
+		local_irq_enable();
+	}
+
 	if (cpumask_any_but(mm_cpumask(mm), cpu) < nr_cpu_ids)
 		flush_tlb_others(mm_cpumask(mm), &info);
 	put_cpu();
@@ -354,8 +362,13 @@ void arch_tlbbatch_flush(struct arch_tlbflush_unmap_batch *batch)
 
 	int cpu = get_cpu();
 
-	if (cpumask_test_cpu(cpu, &batch->cpumask))
+	if (cpumask_test_cpu(cpu, &batch->cpumask)) {
+		VM_WARN_ON(irqs_disabled());
+		local_irq_disable();
 		flush_tlb_func_local(&info, TLB_LOCAL_SHOOTDOWN);
+		local_irq_enable();
+	}
+
 	if (cpumask_any_but(&batch->cpumask, cpu) < nr_cpu_ids)
 		flush_tlb_others(&batch->cpumask, &info);
 	cpumask_clear(&batch->cpumask);
-- 
2.41.0
```
