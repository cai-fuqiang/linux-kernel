From a43ae4dfe56a01f5b98ba0cb2f784b6a43bafcc6 Mon Sep 17 00:00:00 2001
From: Marc Zyngier <marc.zyngier@arm.com>
Date: Tue, 29 May 2018 13:11:09 +0100
Subject: [PATCH 05/14] arm64: Add 'ssbd' command-line option

On a system where the firmware implements ARCH_WORKAROUND_2,
it may be useful to either permanently enable or disable the
workaround for cases where the user decides that they'd rather
not get a trap overhead, and keep the mitigation permanently
on or off instead of switching it on exception entry/exit.

In any case, default to the mitigation being enabled.

Reviewed-by: Julien Grall <julien.grall@arm.com>
Reviewed-by: Mark Rutland <mark.rutland@arm.com>
Acked-by: Will Deacon <will.deacon@arm.com>
Signed-off-by: Marc Zyngier <marc.zyngier@arm.com>
Signed-off-by: Catalin Marinas <catalin.marinas@arm.com>
---
 .../admin-guide/kernel-parameters.txt         |  17 +++
 arch/arm64/include/asm/cpufeature.h           |   6 +
 arch/arm64/kernel/cpu_errata.c                | 103 +++++++++++++++---
 3 files changed, 110 insertions(+), 16 deletions(-)

diff --git a/Documentation/admin-guide/kernel-parameters.txt b/Documentation/admin-guide/kernel-parameters.txt
index 11fc28ecdb6d..7db8868fabab 100644
--- a/Documentation/admin-guide/kernel-parameters.txt
+++ b/Documentation/admin-guide/kernel-parameters.txt
@@ -4047,6 +4047,23 @@
 			expediting.  Set to zero to disable automatic
 			expediting.
 
+	ssbd=		[ARM64,HW]
+			Speculative Store Bypass Disable control
+
+			On CPUs that are vulnerable to the Speculative
+			Store Bypass vulnerability and offer a
+			firmware based mitigation, this parameter
+			indicates how the mitigation should be used:
+
+			force-on:  Unconditionally enable mitigation for
+				   for both kernel and userspace
+			force-off: Unconditionally disable mitigation for
+				   for both kernel and userspace
+			kernel:    Always enable mitigation in the
+				   kernel, and offer a prctl interface
+				   to allow userspace to register its
+				   interest in being mitigated too.
+
 	stack_guard_gap=	[MM]
 			override the default stack gap protection. The value
 			is in page units and it defines how many pages prior
diff --git a/arch/arm64/include/asm/cpufeature.h b/arch/arm64/include/asm/cpufeature.h
index 09b0f2a80c8f..b50650f3e496 100644
--- a/arch/arm64/include/asm/cpufeature.h
+++ b/arch/arm64/include/asm/cpufeature.h
@@ -537,6 +537,12 @@ static inline u64 read_zcr_features(void)
 	return zcr;
 }
 
+#define ARM64_SSBD_UNKNOWN		-1
+#define ARM64_SSBD_FORCE_DISABLE	0
+#define ARM64_SSBD_KERNEL		1
+#define ARM64_SSBD_FORCE_ENABLE		2
+#define ARM64_SSBD_MITIGATED		3
+
 #endif /* __ASSEMBLY__ */
 
 #endif
diff --git a/arch/arm64/kernel/cpu_errata.c b/arch/arm64/kernel/cpu_errata.c
index 7e8f12d85d99..1075f90fdd8c 100644
--- a/arch/arm64/kernel/cpu_errata.c
+++ b/arch/arm64/kernel/cpu_errata.c
@@ -235,6 +235,38 @@ enable_smccc_arch_workaround_1(const struct arm64_cpu_capabilities *entry)
 #ifdef CONFIG_ARM64_SSBD
 DEFINE_PER_CPU_READ_MOSTLY(u64, arm64_ssbd_callback_required);
 
+int ssbd_state __read_mostly = ARM64_SSBD_KERNEL;
+
+static const struct ssbd_options {
+	const char	*str;
+	int		state;
+} ssbd_options[] = {
+	{ "force-on",	ARM64_SSBD_FORCE_ENABLE, },
+	{ "force-off",	ARM64_SSBD_FORCE_DISABLE, },
+	{ "kernel",	ARM64_SSBD_KERNEL, },
+};
+
+static int __init ssbd_cfg(char *buf)
+{
+	int i;
+
+	if (!buf || !buf[0])
+		return -EINVAL;
+
+	for (i = 0; i < ARRAY_SIZE(ssbd_options); i++) {
+		int len = strlen(ssbd_options[i].str);
+
+		if (strncmp(buf, ssbd_options[i].str, len))
+			continue;
+
+		ssbd_state = ssbd_options[i].state;
+		return 0;
+	}
+
+	return -EINVAL;
+}
+early_param("ssbd", ssbd_cfg);
+
 void __init arm64_update_smccc_conduit(struct alt_instr *alt,
 				       __le32 *origptr, __le32 *updptr,
 				       int nr_inst)
@@ -278,44 +310,83 @@ static bool has_ssbd_mitigation(const struct arm64_cpu_capabilities *entry,
 				    int scope)
 {
 	struct arm_smccc_res res;
-	bool supported = true;
+	bool required = true;
+	s32 val;
 
 	WARN_ON(scope != SCOPE_LOCAL_CPU || preemptible());
 
-	if (psci_ops.smccc_version == SMCCC_VERSION_1_0)
+	if (psci_ops.smccc_version == SMCCC_VERSION_1_0) {
+		ssbd_state = ARM64_SSBD_UNKNOWN;
 		return false;
+	}
 
-	/*
-	 * The probe function return value is either negative
-	 * (unsupported or mitigated), positive (unaffected), or zero
-	 * (requires mitigation). We only need to do anything in the
-	 * last case.
-	 */
 	switch (psci_ops.conduit) {
 	case PSCI_CONDUIT_HVC:
 		arm_smccc_1_1_hvc(ARM_SMCCC_ARCH_FEATURES_FUNC_ID,
 				  ARM_SMCCC_ARCH_WORKAROUND_2, &res);
-		if ((int)res.a0 != 0)
-			supported = false;
 		break;
 
 	case PSCI_CONDUIT_SMC:
 		arm_smccc_1_1_smc(ARM_SMCCC_ARCH_FEATURES_FUNC_ID,
 				  ARM_SMCCC_ARCH_WORKAROUND_2, &res);
-		if ((int)res.a0 != 0)
-			supported = false;
 		break;
 
 	default:
-		supported = false;
+		ssbd_state = ARM64_SSBD_UNKNOWN;
+		return false;
 	}
 
-	if (supported) {
-		__this_cpu_write(arm64_ssbd_callback_required, 1);
+	val = (s32)res.a0;
+
+	switch (val) {
+	case SMCCC_RET_NOT_SUPPORTED:
+		ssbd_state = ARM64_SSBD_UNKNOWN;
+		return false;
+
+	case SMCCC_RET_NOT_REQUIRED:
+		pr_info_once("%s mitigation not required\n", entry->desc);
+		ssbd_state = ARM64_SSBD_MITIGATED;
+		return false;
+
+	case SMCCC_RET_SUCCESS:
+		required = true;
+		break;
+
+	case 1:	/* Mitigation not required on this CPU */
+		required = false;
+		break;
+
+	default:
+		WARN_ON(1);
+		return false;
+	}
+
+	switch (ssbd_state) {
+	case ARM64_SSBD_FORCE_DISABLE:
+		pr_info_once("%s disabled from command-line\n", entry->desc);
+		arm64_set_ssbd_mitigation(false);
+		required = false;
+		break;
+
+	case ARM64_SSBD_KERNEL:
+		if (required) {
+			__this_cpu_write(arm64_ssbd_callback_required, 1);
+			arm64_set_ssbd_mitigation(true);
+		}
+		break;
+
+	case ARM64_SSBD_FORCE_ENABLE:
+		pr_info_once("%s forced from command-line\n", entry->desc);
 		arm64_set_ssbd_mitigation(true);
+		required = true;
+		break;
+
+	default:
+		WARN_ON(1);
+		break;
 	}
 
-	return supported;
+	return required;
 }
 #endif	/* CONFIG_ARM64_SSBD */
 
-- 
2.39.0

