# Transparent Hugepage Support
[forum-thp.pdf](http://www.linux-kvm.org/images/9/9e/2010-forum-thp.pdf)

<!--
另外一个链接:

https://www.linux-mm.org/TransparentHugepage?action=AttachFile&do=get&target=transparent-hugepage.pdf
-->

# MAIL LIST v7

https://lore.kernel.org/all/patchbomb.1264513915@v2.random/

# COMMIT
> NOTE
>
> FROM:
>
> https://git.kernel.org/pub/scm/linux/kernel/git/andrea/aa.git

```
commit ae52a2adb5afa5ac5ec5fb5c7b24777f84b6c926
Author: Hugh Dickins <hughd@google.com>
Date:   Thu Jan 13 15:46:28 2011 -0800

    thp: ksm: free swap when swapcache page is replaced


...

commit 22e5c47ee238abe636655c3862ed28d6eb084ad4
Author: Andrea Arcangeli <aarcange@redhat.com>
Date:   Thu Jan 13 15:47:20 2011 -0800

    thp: add compound_trans_head() helper
```


# COMMIT MESSAGE
```

commit 71e3aac0724ffe8918992d76acfe3aad7d8724a5
Author: Andrea Arcangeli <aarcange@redhat.com>
Date:   Thu Jan 13 15:46:52 2011 -0800

    thp: transparent hugepage core

    Lately I've been working to make KVM use hugepages transparently without
    the usual restrictions of hugetlbfs.  Some of the restrictions I'd like to
    see removed:

    > 最近, 最近，我一直致力于让 KVM 透明地使用hugepages，而不受通常的hugetlbfs 
    > 限制. 我希望移除如下的限制:

    1) hugepages have to be swappable or the guest physical memory remains
       locked in RAM and can't be paged out to swap

       > hugepages 必须是 swappable, 否则 guest physical memory 仍然 锁定
       > 在 内存中, 并且无法 paged out to swap.

    2) if a hugepage allocation fails, regular pages should be allocated
       instead and mixed in the same vma without any failure and without
       userland noticing

       > 如果 hugepage分配失败了, 则应分配常规的 pages(normal size page) 
       > 并将其混合到同一个vma中, 不会出现任何失败并且 userland将不会注意到.

    3) if some task quits and more hugepages become available in the
       buddy, guest physical memory backed by regular pages should be
       relocated on hugepages automatically in regions under
       madvise(MADV_HUGEPAGE) (ideally event driven by waking up the
       kernel deamon if the order=HPAGE_PMD_SHIFT-PAGE_SHIFT list becomes
       not null)

       > backed by: 支持,依靠 
       >
       > 如果某些task 退了, 然后更多的hugepages 在 buddy 中变为 available,
       > 由regular pages 构成的 guest physical memory应该被自动的 relocated
       > 到 madvise(MADV_HUGEPAGE)区域下的hugepages上. (理想情况下, 如果order=
       > HPAGE_PMD_SHIFT - PAGE_SHIFT list不为空, 则通过唤醒内核守护进程来驱动
       > 事件)

    4) avoidance of reservation and maximization of use of hugepages whenever
       possible. Reservation (needed to avoid runtime fatal faliures) may be ok for
       1 machine with 1 database with 1 database cache with 1 database cache size
       known at boot time. It's definitely not feasible with a virtualization
       hypervisor usage like RHEV-H that runs an unknown number of virtual machines
       with an unknown size of each virtual machine with an unknown amount of
       pagecache that could be potentially useful in the host for guest not using
       O_DIRECT (aka cache=off).

       > avoidance : 避开,躲避
       > whenever possible: 尽可能
       > fatal /ˈfeɪtl/: 致命的的
       > definitely /ˈdefɪnətli/:  肯定;当然;确实;明确地;清楚地;确切地;没问题
       > feasible [ˈfiːzəbl] : 可行的,行的通的
       > 
       > 尽可能的避免保留和最大限度的使用 hugepage. 对于具有 1 个database和 
       > 1 个database cache 且 1 个database cache 大小在启动时已知的 1 台machine来说，
       > 预留（需要避免运行时致命故障）可能也是可以的。但是对于下面的情况显然是不可行的:
       >
       >  + 带有使用 像 RHEV-H virtualization hypervisor
       >  + 运行了未知数量的 virtual machine
       >  + 每个virtual machine 的内存大小未知
       >  + 每个virtual machine 在 host 中潜在使用的 page cache( guest 没有使用O_DIRECT
       >    也就是说 cache=off)

    hugepages in the virtualization hypervisor (and also in the guest!) are
    much more important than in a regular host not using virtualization,
    becasue with NPT/EPT they decrease the tlb-miss cacheline accesses from 24
    to 19 in case only the hypervisor uses transparent hugepages, and they
    decrease the tlb-miss cacheline accesses from 19 to 15 in case both the
    linux hypervisor and the linux guest both uses this patch (though the
    guest will limit the addition speedup to anonymous regions only for
    now...).  Even more important is that the tlb miss handler is much slower
    on a NPT/EPT guest than for a regular shadow paging or no-virtualization
    scenario.  So maximizing the amount of virtual memory cached by the TLB
    pays off significantly more with NPT/EPT than without (even if there would
    be no significant speedup in the tlb-miss runtime).

    > 在 virtualization hypervisor 中 hugepages (在guest中也是如此) 比起
    > 没有使用 virtualization 的 regular host来说 更加重要, 因为 NPT/EPT 他们
    > 在仅 hypervisor 使用 transparent hugepages的情况下, ltb-miss cacheline access
    > 从24 减少到 19. 在linux hypervisor 和linux guest 都使用该patch的情况下
    > (尽管guest), tlb-miss cacheline accesses 将从 19 减少到 15.

    The first (and more tedious) part of this work requires allowing the VM to
    handle anonymous hugepages mixed with regular pages transparently on
    regular anonymous vmas.  This is what this patch tries to achieve in the
    least intrusive possible way.  We want hugepages and hugetlb to be used in
    a way so that all applications can benefit without changes (as usual we
    leverage the KVM virtualization design: by improving the Linux VM at
    large, KVM gets the performance boost too).

    The most important design choice is: always fallback to 4k allocation if
    the hugepage allocation fails!  This is the _very_ opposite of some large
    pagecache patches that failed with -EIO back then if a 64k (or similar)
    allocation failed...

    Second important decision (to reduce the impact of the feature on the
    existing pagetable handling code) is that at any time we can split an
    hugepage into 512 regular pages and it has to be done with an operation
    that can't fail.  This way the reliability of the swapping isn't decreased
    (no need to allocate memory when we are short on memory to swap) and it's
    trivial to plug a split_huge_page* one-liner where needed without
    polluting the VM.  Over time we can teach mprotect, mremap and friends to
    handle pmd_trans_huge natively without calling split_huge_page*.  The fact
    it can't fail isn't just for swap: if split_huge_page would return -ENOMEM
    (instead of the current void) we'd need to rollback the mprotect from the
    middle of it (ideally including undoing the split_vma) which would be a
    big change and in the very wrong direction (it'd likely be simpler not to
    call split_huge_page at all and to teach mprotect and friends to handle
    hugepages instead of rolling them back from the middle).  In short the
    very value of split_huge_page is that it can't fail.

    The collapsing and madvise(MADV_HUGEPAGE) part will remain separated and
    incremental and it'll just be an "harmless" addition later if this initial
    part is agreed upon.  It also should be noted that locking-wise replacing
    regular pages with hugepages is going to be very easy if compared to what
    I'm doing below in split_huge_page, as it will only happen when
    page_count(page) matches page_mapcount(page) if we can take the PG_lock
    and mmap_sem in write mode.  collapse_huge_page will be a "best effort"
    that (unlike split_huge_page) can fail at the minimal sign of trouble and
    we can try again later.  collapse_huge_page will be similar to how KSM
    works and the madvise(MADV_HUGEPAGE) will work similar to
    madvise(MADV_MERGEABLE).

    The default I like is that transparent hugepages are used at page fault
    time.  This can be changed with
    /sys/kernel/mm/transparent_hugepage/enabled.  The control knob can be set
    to three values "always", "madvise", "never" which mean respectively that
    hugepages are always used, or only inside madvise(MADV_HUGEPAGE) regions,
    or never used.  /sys/kernel/mm/transparent_hugepage/defrag instead
    controls if the hugepage allocation should defrag memory aggressively
    "always", only inside "madvise" regions, or "never".

    The pmd_trans_splitting/pmd_trans_huge locking is very solid.  The
    put_page (from get_user_page users that can't use mmu notifier like
    O_DIRECT) that runs against a __split_huge_page_refcount instead was a
    pain to serialize in a way that would result always in a coherent page
    count for both tail and head.  I think my locking solution with a
    compound_lock taken only after the page_first is valid and is still a
    PageHead should be safe but it surely needs review from SMP race point of
    view.  In short there is no current existing way to serialize the O_DIRECT
    final put_page against split_huge_page_refcount so I had to invent a new
    one (O_DIRECT loses knowledge on the mapping status by the time gup_fast
    returns so...).  And I didn't want to impact all gup/gup_fast users for
    now, maybe if we change the gup interface substantially we can avoid this
    locking, I admit I didn't think too much about it because changing the gup
    unpinning interface would be invasive.

    If we ignored O_DIRECT we could stick to the existing compound refcounting
    code, by simply adding a get_user_pages_fast_flags(foll_flags) where KVM
    (and any other mmu notifier user) would call it without FOLL_GET (and if
    FOLL_GET isn't set we'd just BUG_ON if nobody registered itself in the
    current task mmu notifier list yet).  But O_DIRECT is fundamental for
    decent performance of virtualized I/O on fast storage so we can't avoid it
    to solve the race of put_page against split_huge_page_refcount to achieve
    a complete hugepage feature for KVM.

    Swap and oom works fine (well just like with regular pages ;).  MMU
    notifier is handled transparently too, with the exception of the young bit
    on the pmd, that didn't have a range check but I think KVM will be fine
    because the whole point of hugepages is that EPT/NPT will also use a huge
    pmd when they notice gup returns pages with PageCompound set, so they
    won't care of a range and there's just the pmd young bit to check in that
    case.

    NOTE: in some cases if the L2 cache is small, this may slowdown and waste
    memory during COWs because 4M of memory are accessed in a single fault
    instead of 8k (the payoff is that after COW the program can run faster).
    So we might want to switch the copy_huge_page (and clear_huge_page too) to
    not temporal stores.  I also extensively researched ways to avoid this
    cache trashing with a full prefault logic that would cow in 8k/16k/32k/64k
    up to 1M (I can send those patches that fully implemented prefault) but I
    concluded they're not worth it and they add an huge additional complexity
    and they remove all tlb benefits until the full hugepage has been faulted
    in, to save a little bit of memory and some cache during app startup, but
    they still don't improve substantially the cache-trashing during startup
    if the prefault happens in >4k chunks.  One reason is that those 4k pte
    entries copied are still mapped on a perfectly cache-colored hugepage, so
    the trashing is the worst one can generate in those copies (cow of 4k page
    copies aren't so well colored so they trashes less, but again this results
    in software running faster after the page fault).  Those prefault patches
    allowed things like a pte where post-cow pages were local 4k regular anon
    pages and the not-yet-cowed pte entries were pointing in the middle of
    some hugepage mapped read-only.  If it doesn't payoff substantially with
    todays hardware it will payoff even less in the future with larger l2
    caches, and the prefault logic would blot the VM a lot.  If one is
    emebdded transparent_hugepage can be disabled during boot with sysfs or
    with the boot commandline parameter transparent_hugepage=0 (or
    transparent_hugepage=2 to restrict hugepages inside madvise regions) that
    will ensure not a single hugepage is allocated at boot time.  It is simple
    enough to just disable transparent hugepage globally and let transparent
    hugepages be allocated selectively by applications in the MADV_HUGEPAGE
    region (both at page fault time, and if enabled with the
    collapse_huge_page too through the kernel daemon).

    This patch supports only hugepages mapped in the pmd, archs that have
    smaller hugepages will not fit in this patch alone.  Also some archs like
    power have certain tlb limits that prevents mixing different page size in
    the same regions so they will not fit in this framework that requires
    "graceful fallback" to basic PAGE_SIZE in case of physical memory
    fragmentation.  hugetlbfs remains a perfect fit for those because its
    software limits happen to match the hardware limits.  hugetlbfs also
    remains a perfect fit for hugepage sizes like 1GByte that cannot be hoped
    to be found not fragmented after a certain system uptime and that would be
    very expensive to defragment with relocation, so requiring reservation.
    hugetlbfs is the "reservation way", the point of transparent hugepages is
    not to have any reservation at all and maximizing the use of cache and
    hugepages at all times automatically.

    Some performance result:

    vmx andrea # LD_PRELOAD=/usr/lib64/libhugetlbfs.so HUGETLB_MORECORE=yes HUGETLB_PATH=/mnt/huge/ ./largep
    ages3
    memset page fault 1566023
    memset tlb miss 453854
    memset second tlb miss 453321
    random access tlb miss 41635
    random access second tlb miss 41658
    vmx andrea # LD_PRELOAD=/usr/lib64/libhugetlbfs.so HUGETLB_MORECORE=yes HUGETLB_PATH=/mnt/huge/ ./largepages3
    memset page fault 1566471
    memset tlb miss 453375
    memset second tlb miss 453320
    random access tlb miss 41636
    random access second tlb miss 41637
    vmx andrea # ./largepages3
    memset page fault 1566642
    memset tlb miss 453417
    memset second tlb miss 453313
    random access tlb miss 41630
    random access second tlb miss 41647
    vmx andrea # ./largepages3
    memset page fault 1566872
    memset tlb miss 453418
    memset second tlb miss 453315
    random access tlb miss 41618
    random access second tlb miss 41659
    vmx andrea # echo 0 > /proc/sys/vm/transparent_hugepage
    vmx andrea # ./largepages3
    memset page fault 2182476
    memset tlb miss 460305
    memset second tlb miss 460179
    random access tlb miss 44483
    random access second tlb miss 44186
    vmx andrea # ./largepages3
    memset page fault 2182791
    memset tlb miss 460742
    memset second tlb miss 459962
    random access tlb miss 43981
    random access second tlb miss 43988

    ============
    #include <stdio.h>
    #include <stdlib.h>
    #include <string.h>
    #include <sys/time.h>

    #define SIZE (3UL*1024*1024*1024)

    int main()
    {
            char *p = malloc(SIZE), *p2;
            struct timeval before, after;

            gettimeofday(&before, NULL);
            memset(p, 0, SIZE);
            gettimeofday(&after, NULL);
            printf("memset page fault %Lu\n",
                   (after.tv_sec-before.tv_sec)*1000000UL +
                   after.tv_usec-before.tv_usec);

            gettimeofday(&before, NULL);
            memset(p, 0, SIZE);
            gettimeofday(&after, NULL);
            printf("memset tlb miss %Lu\n",
                   (after.tv_sec-before.tv_sec)*1000000UL +
                   after.tv_usec-before.tv_usec);

            gettimeofday(&before, NULL);
            memset(p, 0, SIZE);
            gettimeofday(&after, NULL);
            printf("memset second tlb miss %Lu\n",
                   (after.tv_sec-before.tv_sec)*1000000UL +
                   after.tv_usec-before.tv_usec);

            gettimeofday(&before, NULL);
            for (p2 = p; p2 < p+SIZE; p2 += 4096)
                    *p2 = 0;
            gettimeofday(&after, NULL);
            printf("random access tlb miss %Lu\n",
                   (after.tv_sec-before.tv_sec)*1000000UL +
                   after.tv_usec-before.tv_usec);

            gettimeofday(&before, NULL);
            for (p2 = p; p2 < p+SIZE; p2 += 4096)
                    *p2 = 0;
            gettimeofday(&after, NULL);
            printf("random access second tlb miss %Lu\n",
                   (after.tv_sec-before.tv_sec)*1000000UL +
                   after.tv_usec-before.tv_usec);

            return 0;
    }
```
