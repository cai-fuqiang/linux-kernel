From 647d0519b53f440a55df163de21c52a8205431cc Mon Sep 17 00:00:00 2001
From: Marc Zyngier <marc.zyngier@arm.com>
Date: Tue, 29 May 2018 13:11:12 +0100
Subject: [PATCH 08/14] arm64: ssbd: Restore mitigation status on CPU resume

On a system where firmware can dynamically change the state of the
mitigation, the CPU will always come up with the mitigation enabled,
including when coming back from suspend.

If the user has requested "no mitigation" via a command line option,
let's enforce it by calling into the firmware again to disable it.

Similarily, for a resume from hibernate, the mitigation could have
been disabled by the boot kernel. Let's ensure that it is set
back on in that case.

Acked-by: Will Deacon <will.deacon@arm.com>
Reviewed-by: Mark Rutland <mark.rutland@arm.com>
Signed-off-by: Marc Zyngier <marc.zyngier@arm.com>
Signed-off-by: Catalin Marinas <catalin.marinas@arm.com>
---
 arch/arm64/include/asm/cpufeature.h |  6 ++++++
 arch/arm64/kernel/cpu_errata.c      |  2 +-
 arch/arm64/kernel/hibernate.c       | 11 +++++++++++
 arch/arm64/kernel/suspend.c         |  8 ++++++++
 4 files changed, 26 insertions(+), 1 deletion(-)

diff --git a/arch/arm64/include/asm/cpufeature.h b/arch/arm64/include/asm/cpufeature.h
index b0fc3224ce8a..55bc1f073bfb 100644
--- a/arch/arm64/include/asm/cpufeature.h
+++ b/arch/arm64/include/asm/cpufeature.h
@@ -553,6 +553,12 @@ static inline int arm64_get_ssbd_state(void)
 #endif
 }
 
+#ifdef CONFIG_ARM64_SSBD
+void arm64_set_ssbd_mitigation(bool state);
+#else
+static inline void arm64_set_ssbd_mitigation(bool state) {}
+#endif
+
 #endif /* __ASSEMBLY__ */
 
 #endif
diff --git a/arch/arm64/kernel/cpu_errata.c b/arch/arm64/kernel/cpu_errata.c
index 2797bc2c8c6a..cf37ca6fa5f2 100644
--- a/arch/arm64/kernel/cpu_errata.c
+++ b/arch/arm64/kernel/cpu_errata.c
@@ -303,7 +303,7 @@ void __init arm64_enable_wa2_handling(struct alt_instr *alt,
 		*updptr = cpu_to_le32(aarch64_insn_gen_nop());
 }
 
-static void arm64_set_ssbd_mitigation(bool state)
+void arm64_set_ssbd_mitigation(bool state)
 {
 	switch (psci_ops.conduit) {
 	case PSCI_CONDUIT_HVC:
diff --git a/arch/arm64/kernel/hibernate.c b/arch/arm64/kernel/hibernate.c
index 1ec5f28c39fc..6b2686d54411 100644
--- a/arch/arm64/kernel/hibernate.c
+++ b/arch/arm64/kernel/hibernate.c
@@ -313,6 +313,17 @@ int swsusp_arch_suspend(void)
 
 		sleep_cpu = -EINVAL;
 		__cpu_suspend_exit();
+
+		/*
+		 * Just in case the boot kernel did turn the SSBD
+		 * mitigation off behind our back, let's set the state
+		 * to what we expect it to be.
+		 */
+		switch (arm64_get_ssbd_state()) {
+		case ARM64_SSBD_FORCE_ENABLE:
+		case ARM64_SSBD_KERNEL:
+			arm64_set_ssbd_mitigation(true);
+		}
 	}
 
 	local_daif_restore(flags);
diff --git a/arch/arm64/kernel/suspend.c b/arch/arm64/kernel/suspend.c
index a307b9e13392..70c283368b64 100644
--- a/arch/arm64/kernel/suspend.c
+++ b/arch/arm64/kernel/suspend.c
@@ -62,6 +62,14 @@ void notrace __cpu_suspend_exit(void)
 	 */
 	if (hw_breakpoint_restore)
 		hw_breakpoint_restore(cpu);
+
+	/*
+	 * On resume, firmware implementing dynamic mitigation will
+	 * have turned the mitigation on. If the user has forcefully
+	 * disabled it, make sure their wishes are obeyed.
+	 */
+	if (arm64_get_ssbd_state() == ARM64_SSBD_FORCE_DISABLE)
+		arm64_set_ssbd_mitigation(false);
 }
 
 /*
-- 
2.39.0

