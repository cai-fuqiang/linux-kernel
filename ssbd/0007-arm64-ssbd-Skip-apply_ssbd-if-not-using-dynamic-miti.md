From 986372c4367f46b34a3c0f6918d7fb95cbdf39d6 Mon Sep 17 00:00:00 2001
From: Marc Zyngier <marc.zyngier@arm.com>
Date: Tue, 29 May 2018 13:11:11 +0100
Subject: [PATCH 07/14] arm64: ssbd: Skip apply_ssbd if not using dynamic
 mitigation

In order to avoid checking arm64_ssbd_callback_required on each
kernel entry/exit even if no mitigation is required, let's
add yet another alternative that by default jumps over the mitigation,
and that gets nop'ed out if we're doing dynamic mitigation.

Think of it as a poor man's static key...

Reviewed-by: Julien Grall <julien.grall@arm.com>
Reviewed-by: Mark Rutland <mark.rutland@arm.com>
Acked-by: Will Deacon <will.deacon@arm.com>
Signed-off-by: Marc Zyngier <marc.zyngier@arm.com>
Signed-off-by: Catalin Marinas <catalin.marinas@arm.com>
---
 arch/arm64/kernel/cpu_errata.c | 14 ++++++++++++++
 arch/arm64/kernel/entry.S      |  3 +++
 2 files changed, 17 insertions(+)

diff --git a/arch/arm64/kernel/cpu_errata.c b/arch/arm64/kernel/cpu_errata.c
index 1075f90fdd8c..2797bc2c8c6a 100644
--- a/arch/arm64/kernel/cpu_errata.c
+++ b/arch/arm64/kernel/cpu_errata.c
@@ -289,6 +289,20 @@ void __init arm64_update_smccc_conduit(struct alt_instr *alt,
 	*updptr = cpu_to_le32(insn);
 }
 
+void __init arm64_enable_wa2_handling(struct alt_instr *alt,
+				      __le32 *origptr, __le32 *updptr,
+				      int nr_inst)
+{
+	BUG_ON(nr_inst != 1);
+	/*
+	 * Only allow mitigation on EL1 entry/exit and guest
+	 * ARCH_WORKAROUND_2 handling if the SSBD state allows it to
+	 * be flipped.
+	 */
+	if (arm64_get_ssbd_state() == ARM64_SSBD_KERNEL)
+		*updptr = cpu_to_le32(aarch64_insn_gen_nop());
+}
+
 static void arm64_set_ssbd_mitigation(bool state)
 {
 	switch (psci_ops.conduit) {
diff --git a/arch/arm64/kernel/entry.S b/arch/arm64/kernel/entry.S
index 29ad672a6abd..e6f6e2339b22 100644
--- a/arch/arm64/kernel/entry.S
+++ b/arch/arm64/kernel/entry.S
@@ -142,6 +142,9 @@ alternative_else_nop_endif
 	// to save/restore them if required.
 	.macro	apply_ssbd, state, targ, tmp1, tmp2
 #ifdef CONFIG_ARM64_SSBD
+alternative_cb	arm64_enable_wa2_handling
+	b	\targ
+alternative_cb_end
 	ldr_this_cpu	\tmp2, arm64_ssbd_callback_required, \tmp1
 	cbz	\tmp2, \targ
 	mov	w0, #ARM_SMCCC_ARCH_WORKAROUND_2
-- 
2.39.0

