```diff
From 0bf0f444b2c49241b2b39aa3cf210d7c95ef6c34 Mon Sep 17 00:00:00 2001
From: Will Deacon <will.deacon@arm.com>
Date: Tue, 7 Aug 2018 13:43:06 +0100
Subject: [PATCH 4/7] arm64: entry: Allow handling of undefined instructions
 from EL1

Rather than panic() when taking an undefined instruction exception from
EL1, allow a hook to be registered in case we want to emulate the
instruction, like we will for the SSBS PSTATE manipulation instructions.

Signed-off-by: Will Deacon <will.deacon@arm.com>
Signed-off-by: Catalin Marinas <catalin.marinas@arm.com>
---
 arch/arm64/kernel/entry.S |  2 +-
 arch/arm64/kernel/traps.c | 11 +++++++----
 2 files changed, 8 insertions(+), 5 deletions(-)

diff --git a/arch/arm64/kernel/entry.S b/arch/arm64/kernel/entry.S
index 09dbea221a27..8556876c9109 100644
--- a/arch/arm64/kernel/entry.S
+++ b/arch/arm64/kernel/entry.S
@@ -589,7 +589,7 @@ el1_undef:
 	inherit_daif	pstate=x23, tmp=x2
 	mov	x0, sp
 	bl	do_undefinstr
-	ASM_BUG()
+	kernel_exit 1
 el1_dbg:
 	/*
 	 * Debug exception handling
diff --git a/arch/arm64/kernel/traps.c b/arch/arm64/kernel/traps.c
index 039e9ff379cc..b9da093e0341 100644
--- a/arch/arm64/kernel/traps.c
+++ b/arch/arm64/kernel/traps.c
@@ -310,10 +310,12 @@ static int call_undef_hook(struct pt_regs *regs)
 	int (*fn)(struct pt_regs *regs, u32 instr) = NULL;
 	void __user *pc = (void __user *)instruction_pointer(regs);
 
-	if (!user_mode(regs))
-		return 1;
-
-	if (compat_thumb_mode(regs)) {
+	if (!user_mode(regs)) {
+		__le32 instr_le;
+		if (probe_kernel_address((__force __le32 *)pc, instr_le))
+			goto exit;
+		instr = le32_to_cpu(instr_le);
+	} else if (compat_thumb_mode(regs)) {
 		/* 16-bit Thumb instruction */
 		__le16 instr_le;
 		if (get_user(instr_le, (__le16 __user *)pc))
@@ -407,6 +409,7 @@ asmlinkage void __exception do_undefinstr(struct pt_regs *regs)
 		return;
 
 	force_signal_inject(SIGILL, ILL_ILLOPC, regs->pc);
+	BUG_ON(!user_mode(regs));
 }
 
 void cpu_enable_cache_maint_trap(const struct arm64_cpu_capabilities *__unused)
-- 
2.39.0
```
