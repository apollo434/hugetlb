The hugetlb analyze based on kernel 4.1.21

==========================================================================
The hugetlb key structure:
```c

super_block
{
...
	void *s_fs_info  /* Filesystem private info */
		|
		|
		 \___ >>> hugetlbfs_sb_info
		     {
		      ...
			struct hstate *hstate /* Defines one hugetlb page size */
			{
				int next_nid_to_alloc;
				int next_nid_to_free;
				unsigned int order;
				unsigned long mask;
				unsigned long max_huge_pages;
				unsigned long nr_huge_pages;
				unsigned long free_huge_pages;
				unsigned long resv_huge_pages;
				unsigned long surplus_huge_pages;
				unsigned long nr_overcommit_huge_pages;
				struct list_head hugepage_activelist;
				struct list_head hugepage_freelists[MAX_NUMNODES];
				unsigned int nr_huge_pages_node[MAX_NUMNODES];
				unsigned int free_huge_pages_node[MAX_NUMNODES];
				unsigned int surplus_huge_pages_node[MAX_NUMNODES];
			#ifdef CONFIG_CGROUP_HUGETLB
				/* cgroup control files */
				struct cftype cgroup_files[5];
			#endif
				char name[HSTATE_NAME_LEN];
			}
		      ...

		     }
...
}

```

==========================================================================
Hugetlb allocate memory via do_page_fault() function, it is why called hugetlb as a patch of MM.
```c
COW:

do_page_fault		      ----|
  __do_page_fault		  |>( all of these operations are standerd do_page_fault path )
   handle_mm_fault		  |
    __handle_mm_fault         ----|
     hugetlb_fault  	      ====> ( At this point, the allocate page go through hugetlb path )
      hugetlb_no_page
       alloc_huge_page
	dequeue_huge_page_vma ====> ( get page from hugepage_freelists[] )
	alloc_buddy_huge_page ====> ( get page via buddy system )
	  alloc_pages	      ----> ( At this point, the allocate page get back to standerd alloc_pages() )
	    alloc_pages_node
	      __alloc_pages
		__alloc_pages_nodemask   /* This is the 'heart' of the zoned buddy allocator */
		  get_page_from_freelist /* goes through the zonelist trying to allocate a page */
			 buffered_rmqueue /* allocate from buddy system */
		  __alloc_pages_slowpath /* No enough page in zonelist, try to allocate page from being preactive releasing some space */
```
buffered_rmqueue code flow:
![Alt text](/buffered_rmqueue.png)

```c
buffered_rmqueue()
{
	...
	if (likely(order == 0))
	{
		...
		/* Obtain a specified number of elements from the buddy allocator */
		pcp->count += rmqueue_bulk()
		...
		/* Attempt to get the one page from per-CPU cache for enhancement */
		if (cold)
			page = list_entry(list->prev, struct page, lru);
		else
			page = list_entry(list->next, struct page, lru);
		}
		else
		{
			...
			/* Do the hard work of removing an element from the buddy allocator */
			page = __rmqueue(zone, order, migratetype);
			...
		}
		...
}

```

```c
Buddy system.

Main structure.

On NUMA machines, each NUMA node would have a pg_data_t to describe it's memory layout
typedef struct pglist_data {
        struct zone node_zones[MAX_NR_ZONES];
        struct zonelist node_zonelists[MAX_ZONELISTS];
        int nr_zones;
...
}pg_data_t;


Each Node has only two zonelist:
1)one for all zones with memory
2)one containing just zones from the node the zonelist belongs to.

This zone list contains a maximum of MAXNODES*MAX_NR_ZONES zones
(include/linux/mmzone.h)
struct zonelist {
        struct zonelist_cache *zlcache_ptr;                  // NULL or &zlcache
        struct zoneref _zonerefs[MAX_ZONES_PER_ZONELIST + 1];
#ifdef CONFIG_NUMA
        struct zonelist_cache zlcache;                       // optional ...
#endif
};

(include/linux/mmzone.h)
struct zone {
...
  /* free areas of different sizes */
  struct free_area free_area[MAX_ORDER];
...
}

(include/linux/mmzone.h)
struct free_area {
          struct list_head        free_list[MIGRATE_TYPES];
          unsigned long           nr_free;
};


The organization of "ZONE" in memory:
 _________
|____0____|           free_list  ---------> free_list
|____1____|          _______________       _______________
|____2____| <=====> |_1_|_2_|_3_|_4_|<===>|_1_|_2_|_3_|_4_|   (1,2....nr_free)
    . . ^^                                 ^^
    . . ||=================================||             
    . .
|MAX_ORDER|


/*
 * This is the 'heart' of the zoned buddy allocator.
 */
struct page *
__alloc_pages_nodemask(gfp_t gfp_mask, unsigned int order,
                        struct zonelist *zonelist, nodemask_t *nodemask)
{
        struct zoneref *preferred_zoneref;
        struct page *page = NULL;
        unsigned int cpuset_mems_cookie;
        int alloc_flags = ALLOC_WMARK_LOW|ALLOC_CPUSET|ALLOC_FAIR;
        gfp_t alloc_mask; /* The gfp_t that was actually used for allocation */
        struct alloc_context ac = {
                .high_zoneidx = gfp_zone(gfp_mask),
                .nodemask = nodemask,
                .migratetype = gfpflags_to_migratetype(gfp_mask),
        };

        gfp_mask &= gfp_allowed_mask;

        lockdep_trace_alloc(gfp_mask);

        might_sleep_if(gfp_mask & __GFP_WAIT);

        if (should_fail_alloc_page(gfp_mask, order))
                return NULL;

        /*
         * Check the zones suitable for the gfp_mask contain at least one
         * valid zone. It's possible to have an empty zonelist as a result
         * of __GFP_THISNODE and a memoryless node
         */
        if (unlikely(!zonelist->_zonerefs->zone))
                return NULL;

        if (IS_ENABLED(CONFIG_CMA) && ac.migratetype == MIGRATE_MOVABLE)
                alloc_flags |= ALLOC_CMA;

retry_cpuset:
        cpuset_mems_cookie = read_mems_allowed_begin();

        /* We set it here, as __alloc_pages_slowpath might have changed it */
        ac.zonelist = zonelist;
        /* The preferred zone is used for statistics later */
        preferred_zoneref = first_zones_zonelist(ac.zonelist, ac.high_zoneidx,
                                ac.nodemask ? : &cpuset_current_mems_allowed,
                                &ac.preferred_zone);
        if (!ac.preferred_zone)
                goto out;
        ac.classzone_idx = zonelist_zone_idx(preferred_zoneref);

        /* First allocation attempt */
        alloc_mask = gfp_mask|__GFP_HARDWALL;
        page = get_page_from_freelist(alloc_mask, order, alloc_flags, &ac);
        if (unlikely(!page)) {
                /*
                 * Runtime PM, block IO and its error handling path
                 * can deadlock because I/O on the device might not
                 * complete.
                 */
                alloc_mask = memalloc_noio_flags(gfp_mask);

                page = __alloc_pages_slowpath(alloc_mask, order, &ac);
        }

        if (kmemcheck_enabled && page)
                kmemcheck_pagealloc_alloc(page, order, gfp_mask);

        trace_mm_page_alloc(page, order, alloc_mask, ac.migratetype);

out:
        /*
         * When updating a task's mems_allowed, it is possible to race with
         * parallel threads in such a way that an allocation can fail while
         * the mask is being updated. If a page allocation is about to fail,
         * check if the cpuset changed during allocation and if so, retry.
         */
        if (unlikely(!page && read_mems_allowed_retry(cpuset_mems_cookie)))
                goto retry_cpuset;

        return page;
}
EXPORT_SYMBOL(__alloc_pages_nodemask);


```

![Alt text](/overview.jpeg)

```c

wrsadmin@pek-cdong-u145:~$ cat /proc/buddyinfo
Node 0, zone      DMA      1      1      0      0      2      1      1      0     1      1      3
Node 0, zone    DMA32   1476   1253    842    590    663    827    557    361   265      3    487
Node 0, zone   Normal   6323   4159   2998   2031   2181   3306   2247   1442  1047     11   1720

nr_free                    0      1      2      3      4      5      6      7     8      9     10
                       <---------------------------------------------------------------------------->
left                      1476  1253   842    590    663    827    557    361    265     3     487

```
==========================================================================
```c
1.Hugetlb has no read/write operation, all of the action via MMAP completed.
2.MMAP does not allocate physical page, only set the offset in VMA.

MMAP:

const struct file_operations hugetlbfs_file_operations = {
        .read_iter              = hugetlbfs_read_iter,
        .mmap                   = hugetlbfs_file_mmap,
        .fsync                  = noop_fsync,
        .get_unmapped_area      = hugetlb_get_unmapped_area,
        .llseek         = default_llseek,
};

hugetlbfs_file_mmap
  hugetlb_reserve_pages

```
==========================================================================
Hugetlb Init:

```c
Huge page allocation:

hugetlb_init
 hugetlb_init_hstates
  hugetlb_hstate_alloc_pages
   alloc_fresh_huge_page
    alloc_fresh_huge_page_node
     alloc_pages_exact_node
      __alloc_pages
       __alloc_pages_nodemask ====> allocate the huge page
    prep_new_huge_page
     put_page ===============> put the huge page into hugepage_freelists[]


Huge FS Init:

```
