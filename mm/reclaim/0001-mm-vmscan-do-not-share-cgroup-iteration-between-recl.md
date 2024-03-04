```diff
From 1ba6fc9af35bf97c84567d9b3eeb26629d1e3af0 Mon Sep 17 00:00:00 2001
From: Johannes Weiner <hannes@cmpxchg.org>
Date: Mon, 23 Sep 2019 15:35:01 -0700
Subject: [PATCH] mm: vmscan: do not share cgroup iteration between reclaimers

One of our services observed a high rate of cgroup OOM kills in the
presence of large amounts of clean cache.  Debugging showed that the
culprit is the shared cgroup iteration in page reclaim.

Under high allocation concurrency, multiple threads enter reclaim at the
same time.  Fearing overreclaim when we first switched from the single
global LRU to cgrouped LRU lists, we introduced a shared iteration state
for reclaim invocations - whether 1 or 20 reclaimers are active
concurrently, we only walk the cgroup tree once: the 1st reclaimer
reclaims the first cgroup, the second the second one etc.  With more
reclaimers than cgroups, we start another walk from the top.

This sounded reasonable at the time, but the problem is that reclaim
concurrency doesn't scale with allocation concurrency.  As reclaim
concurrency increases, the amount of memory individual reclaimers get to
scan gets smaller and smaller.  Individual reclaimers may only see one
cgroup per cycle, and that may not have much reclaimable memory.  We see
individual reclaimers declare OOM when there is plenty of reclaimable
memory available in cgroups they didn't visit.

This patch does away with the shared iterator, and every reclaimer is
allowed to scan the full cgroup tree and see all of reclaimable memory,
just like it would on a non-cgrouped system.  This way, when OOM is
declared, we know that the reclaimer actually had a chance.

To still maintain fairness in reclaim pressure, disallow cgroup reclaim
from bailing out of the tree walk early.  Kswapd and regular direct
reclaim already don't bail, so it's not clear why limit reclaim would have
to, especially since it only walks subtrees to begin with.

This change completely eliminates the OOM kills on our service, while
showing no signs of overreclaim - no increased scan rates, %sys time, or
abrupt free memory spikes.  I tested across 100 machines that have 64G of
RAM and host about 300 cgroups each.

[ It's possible overreclaim never was a *practical* issue to begin
  with - it was simply a concern we had on the mailing lists at the
  time, with no real data to back it up. But we have also added more
  bail-out conditions deeper inside reclaim (e.g. the proportional
  exit in shrink_node_memcg) since. Regardless, now we have data that
  suggests full walks are more reliable and scale just fine. ]

Link: http://lkml.kernel.org/r/20190812192316.13615-1-hannes@cmpxchg.org
Signed-off-by: Johannes Weiner <hannes@cmpxchg.org>
Reviewed-by: Roman Gushchin <guro@fb.com>
Acked-by: Michal Hocko <mhocko@suse.com>
Cc: Vladimir Davydov <vdavydov.dev@gmail.com>
Signed-off-by: Andrew Morton <akpm@linux-foundation.org>
Signed-off-by: Linus Torvalds <torvalds@linux-foundation.org>
---
 mm/vmscan.c | 22 ++--------------------
 1 file changed, 2 insertions(+), 20 deletions(-)

diff --git a/mm/vmscan.c b/mm/vmscan.c
index 8e03427cb64f..2ee4d9283738 100644
--- a/mm/vmscan.c
+++ b/mm/vmscan.c
@@ -2664,10 +2664,6 @@ static bool shrink_node(pg_data_t *pgdat, struct scan_control *sc)
 
 	do {
 		struct mem_cgroup *root = sc->target_mem_cgroup;
-		struct mem_cgroup_reclaim_cookie reclaim = {
-			.pgdat = pgdat,
-			.priority = sc->priority,
-		};
 		unsigned long node_lru_pages = 0;
 		struct mem_cgroup *memcg;
 
@@ -2676,7 +2672,7 @@ static bool shrink_node(pg_data_t *pgdat, struct scan_control *sc)
 		nr_reclaimed = sc->nr_reclaimed;
 		nr_scanned = sc->nr_scanned;
 
-		memcg = mem_cgroup_iter(root, NULL, &reclaim);
+		memcg = mem_cgroup_iter(root, NULL, NULL);
 		do {
 			unsigned long lru_pages;
 			unsigned long reclaimed;
@@ -2719,21 +2715,7 @@ static bool shrink_node(pg_data_t *pgdat, struct scan_control *sc)
 				   sc->nr_scanned - scanned,
 				   sc->nr_reclaimed - reclaimed);
 
-			/*
-			 * Kswapd have to scan all memory cgroups to fulfill
-			 * the overall scan target for the node.
-			 *
-			 * Limit reclaim, on the other hand, only cares about
-			 * nr_to_reclaim pages to be reclaimed and it will
-			 * retry with decreasing priority if one round over the
-			 * whole hierarchy is not sufficient.
-			 */
-			if (!current_is_kswapd() &&
-					sc->nr_reclaimed >= sc->nr_to_reclaim) {
-				mem_cgroup_iter_break(root, memcg);
-				break;
-			}
-		} while ((memcg = mem_cgroup_iter(root, memcg, &reclaim)));
+		} while ((memcg = mem_cgroup_iter(root, memcg, NULL)));
 
 		if (reclaim_state) {
 			sc->nr_reclaimed += reclaim_state->reclaimed_slab;
-- 
2.42.0
```
