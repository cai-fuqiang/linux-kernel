# commit message
```diff
From d71be2b6c0e19180b5f80a6d42039cc074a693a2 Mon Sep 17 00:00:00 2001
From: Will Deacon <will.deacon@arm.com>
Date: Fri, 15 Jun 2018 11:37:34 +0100
Subject: [PATCH 2/7] arm64: cpufeature: Detect SSBS and advertise to userspace

Armv8.5 introduces a new PSTATE bit known as Speculative Store Bypass
Safe (SSBS) which can be used as a mitigation against Spectre variant 4.

Armv8.5 引入一个 新的 PSTATE bit -- ssbs, 用于 mitigation Spectre variant 4
(不一定8.5支持, 里面有两个 FEATURE : FEAT_SSBS, FEAT_SSBS2, 8.5肯定是 支持
他两个, 但是低版本也可能支持)

Additionally, a CPU may provide instructions to manipulate PSTATE.SSBS
directly, so that userspace can toggle the SSBS control without trapping
to the kernel.

manipulate : 操作，操控

toggle /ˈtɒɡl/ : 切换

另外, CPU 可能提供了一个指令，用于直接操作 PSTATE.SSBS, 这样 userspace 可以直接
切换 SSBS 的控制权, 在不trap到kernel的情况下.

This patch probes for the existence of SSBS and advertise the new instructions
to userspace if they exist.

Reviewed-by: Suzuki K Poulose <suzuki.poulose@arm.com>
Signed-off-by: Will Deacon <will.deacon@arm.com>
Signed-off-by: Catalin Marinas <catalin.marinas@arm.com>
---
 arch/arm64/include/asm/cpucaps.h    |  3 ++-
 arch/arm64/include/asm/sysreg.h     | 16 ++++++++++++----
 arch/arm64/include/uapi/asm/hwcap.h |  1 +
 arch/arm64/kernel/cpufeature.c      | 19 +++++++++++++++++--
 arch/arm64/kernel/cpuinfo.c         |  1 +
 5 files changed, 33 insertions(+), 7 deletions(-)
```
# diff
```diff
diff --git a/arch/arm64/include/asm/cpucaps.h b/arch/arm64/include/asm/cpucaps.h
index 9932aca9704b..38eec9cf30f2 100644
--- a/arch/arm64/include/asm/cpucaps.h
+++ b/arch/arm64/include/asm/cpucaps.h
@@ -52,7 +52,8 @@
 #define ARM64_MISMATCHED_CACHE_TYPE		31
 #define ARM64_HAS_STAGE2_FWB			32
 #define ARM64_HAS_CRC32				33
+#define ARM64_SSBS				34
 
-#define ARM64_NCAPS				34
+#define ARM64_NCAPS				35
 
 #endif /* __ASM_CPUCAPS_H */
diff --git a/arch/arm64/include/asm/sysreg.h b/arch/arm64/include/asm/sysreg.h
index c1470931b897..2fc6242baf11 100644
--- a/arch/arm64/include/asm/sysreg.h
+++ b/arch/arm64/include/asm/sysreg.h
@@ -419,6 +419,7 @@
 #define SYS_ICH_LR15_EL2		__SYS__LR8_EL2(7)
 
 /* Common SCTLR_ELx flags. */
+#define SCTLR_ELx_DSSBS	(1UL << 44)
 #define SCTLR_ELx_EE    (1 << 25)
 #define SCTLR_ELx_IESB	(1 << 21)
 #define SCTLR_ELx_WXN	(1 << 19)
@@ -439,7 +440,7 @@
 			 (1 << 10) | (1 << 13) | (1 << 14) | (1 << 15) | \
 			 (1 << 17) | (1 << 20) | (1 << 24) | (1 << 26) | \ 
 			 (1 << 27) | (1 << 30) | (1 << 31) | \
-			 (0xffffffffUL << 32))
+			 (0xffffefffUL << 32))
 
 #ifdef CONFIG_CPU_BIG_ENDIAN
 #define ENDIAN_SET_EL2		SCTLR_ELx_EE
@@ -453,7 +454,7 @@
 #define SCTLR_EL2_SET	(SCTLR_ELx_IESB   | ENDIAN_SET_EL2   | SCTLR_EL2_RES1)
 #define SCTLR_EL2_CLEAR	(SCTLR_ELx_M      | SCTLR_ELx_A    | SCTLR_ELx_C   | \
 			 SCTLR_ELx_SA     | SCTLR_ELx_I    | SCTLR_ELx_WXN | \
-			 ENDIAN_CLEAR_EL2 | SCTLR_EL2_RES0)
+			 SCTLR_ELx_DSSBS | ENDIAN_CLEAR_EL2 | SCTLR_EL2_RES0)
 
 #if (SCTLR_EL2_SET ^ SCTLR_EL2_CLEAR) != 0xffffffffffffffff
 #error "Inconsistent SCTLR_EL2 set/clear bits"
@@ -477,7 +478,7 @@
 			 (1 << 29))
 #define SCTLR_EL1_RES0  ((1 << 6)  | (1 << 10) | (1 << 13) | (1 << 17) | \
 			 (1 << 27) | (1 << 30) | (1 << 31) | \
-			 (0xffffffffUL << 32))
+			 (0xffffefffUL << 32))
 
 #ifdef CONFIG_CPU_BIG_ENDIAN
 #define ENDIAN_SET_EL1		(SCTLR_EL1_E0E | SCTLR_ELx_EE)
@@ -494,7 +495,7 @@
 			 ENDIAN_SET_EL1 | SCTLR_EL1_UCI  | SCTLR_EL1_RES1)
 #define SCTLR_EL1_CLEAR	(SCTLR_ELx_A   | SCTLR_EL1_CP15BEN | SCTLR_EL1_ITD    |\
 			 SCTLR_EL1_UMA | SCTLR_ELx_WXN     | ENDIAN_CLEAR_EL1 |\
-			 SCTLR_EL1_RES0)
+			 SCTLR_ELx_DSSBS | SCTLR_EL1_RES0)
 
 #if (SCTLR_EL1_SET ^ SCTLR_EL1_CLEAR) != 0xffffffffffffffff
 #error "Inconsistent SCTLR_EL1 set/clear bits"
@@ -544,6 +545,13 @@
 #define ID_AA64PFR0_EL0_64BIT_ONLY	0x1
 #define ID_AA64PFR0_EL0_32BIT_64BIT	0x2
 
+/* id_aa64pfr1 */
+#define ID_AA64PFR1_SSBS_SHIFT		4
+
+#define ID_AA64PFR1_SSBS_PSTATE_NI	0
+#define ID_AA64PFR1_SSBS_PSTATE_ONLY	1
+#define ID_AA64PFR1_SSBS_PSTATE_INSNS	2
+
 /* id_aa64mmfr0 */
 #define ID_AA64MMFR0_TGRAN4_SHIFT	28
 #define ID_AA64MMFR0_TGRAN64_SHIFT	24
diff --git a/arch/arm64/include/uapi/asm/hwcap.h b/arch/arm64/include/uapi/asm/hwcap.h
index 17c65c8f33cb..2bcd6e4f3474 100644
--- a/arch/arm64/include/uapi/asm/hwcap.h
+++ b/arch/arm64/include/uapi/asm/hwcap.h
@@ -48,5 +48,6 @@
 #define HWCAP_USCAT		(1 << 25)
 #define HWCAP_ILRCPC		(1 << 26)
 #define HWCAP_FLAGM		(1 << 27)
+#define HWCAP_SSBS		(1 << 28)
 
 #endif /* _UAPI__ASM_HWCAP_H */
diff --git a/arch/arm64/kernel/cpufeature.c b/arch/arm64/kernel/cpufeature.c
index 7626b80128f5..5794959d8beb 100644
--- a/arch/arm64/kernel/cpufeature.c
+++ b/arch/arm64/kernel/cpufeature.c
@@ -164,6 +164,11 @@ static const struct arm64_ftr_bits ftr_id_aa64pfr0[] = {
 	ARM64_FTR_END,
 };
+static const struct arm64_ftr_bits ftr_id_aa64pfr1[] = {
+	ARM64_FTR_BITS(FTR_VISIBLE, FTR_STRICT, FTR_LOWER_SAFE, ID_AA64PFR1_SSBS_SHIFT, 4, ID_AA64PFR1_SSBS_PSTATE_NI),
+	ARM64_FTR_END,
+};
+
 static const struct arm64_ftr_bits ftr_id_aa64mmfr0[] = {
 	S_ARM64_FTR_BITS(FTR_HIDDEN, FTR_STRICT, FTR_LOWER_SAFE, ID_AA64MMFR0_TGRAN4_SHIFT, 4, ID_AA64MMFR0_TGRAN4_NI),
 	S_ARM64_FTR_BITS(FTR_HIDDEN, FTR_STRICT, FTR_LOWER_SAFE, ID_AA64MMFR0_TGRAN64_SHIFT, 4, ID_AA64MMFR0_TGRAN64_NI),
@@ -371,7 +376,7 @@ static const struct __ftr_reg_entry {
 
 	/* Op1 = 0, CRn = 0, CRm = 4 */
 	ARM64_FTR_REG(SYS_ID_AA64PFR0_EL1, ftr_id_aa64pfr0),
-	ARM64_FTR_REG(SYS_ID_AA64PFR1_EL1, ftr_raz),
+	ARM64_FTR_REG(SYS_ID_AA64PFR1_EL1, ftr_id_aa64pfr1),
 	ARM64_FTR_REG(SYS_ID_AA64ZFR0_EL1, ftr_raz),
 
 	/* Op1 = 0, CRn = 0, CRm = 5 */
@@ -657,7 +662,6 @@ void update_cpu_features(int cpu,
 
 	/*
 	 * EL3 is not our concern.
-	 * ID_AA64PFR1 is currently RES0.
 	 */
 	taint |= check_update_ftr_reg(SYS_ID_AA64PFR0_EL1, cpu,
 				      info->reg_id_aa64pfr0, boot->reg_id_aa64pfr0);
@@ -1231,6 +1235,16 @@ static const struct arm64_cpu_capabilities arm64_features[] = {
 		.field_pos = ID_AA64ISAR0_CRC32_SHIFT,
 		.min_field_value = 1,
 	},
    //============(1)===============
+	{
+		.desc = "Speculative Store Bypassing Safe (SSBS)",
+		.capability = ARM64_SSBS,
+		.type = ARM64_CPUCAP_WEAK_LOCAL_CPU_FEATURE,
+		.matches = has_cpuid_feature,
+		.sys_reg = SYS_ID_AA64PFR1_EL1,
+		.field_pos = ID_AA64PFR1_SSBS_SHIFT,
+		.sign = FTR_UNSIGNED,
+		.min_field_value = ID_AA64PFR1_SSBS_PSTATE_ONLY,
+	},
 	{},
 };
 
@@ -1276,6 +1290,7 @@ static const struct arm64_cpu_capabilities arm64_elf_hwcaps[] = {
 #ifdef CONFIG_ARM64_SVE
 	HWCAP_CAP(SYS_ID_AA64PFR0_EL1, ID_AA64PFR0_SVE_SHIFT, FTR_UNSIGNED, ID_AA64PFR0_SVE, CAP_HWCAP, HWCAP_SVE),
 #endif
    //============(2)===============
+	HWCAP_CAP(SYS_ID_AA64PFR1_EL1, ID_AA64PFR1_SSBS_SHIFT, FTR_UNSIGNED, ID_AA64PFR1_SSBS_PSTATE_INSNS, CAP_HWCAP, HWCAP_SSBS),
 	{},
 };
 
diff --git a/arch/arm64/kernel/cpuinfo.c b/arch/arm64/kernel/cpuinfo.c
index e9ab7b3ed317..dce971f2c167 100644
--- a/arch/arm64/kernel/cpuinfo.c
+++ b/arch/arm64/kernel/cpuinfo.c
@@ -81,6 +81,7 @@ static const char *const hwcap_str[] = {
 	"uscat",
 	"ilrcpc",
 	"flagm",
+	"ssbs",
 	NULL
 };
 
-- 
2.39.0
```
1. 该ARM64_SSBS cap 需要满足 aa64pfr1_el1.ssbs = 0x1, 也就是 FEAT_SSBS2
2. 该feature 需要满足 aa64pfr1_el1.ssbs = 0x2 , 也就是 FEAT_SSBS
