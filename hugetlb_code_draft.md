The hugetlb analyze based on kernel 4.1.21 

===================================================================================================
The hugetlb key structure:


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
	


===================================================================================================
Hugetlb allocate memory via do_page_fault() function, it is why called hugetlb as a patch of MM.

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
		  __alloc_pages_slowpath /* No enough page in zonelist, try to allocate page from being preactive releasing some space */

===================================================================================================
Buddy system.

Main structure.

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


wrsadmin@pek-cdong-u145:~$ cat /proc/buddyinfo 
Node 0, zone      DMA      1      1      0      0      2      1      1      0     1      1      3 
Node 0, zone    DMA32   1476   1253    842    590    663    827    557    361   265      3    487 
Node 0, zone   Normal   6323   4159   2998   2031   2181   3306   2247   1442  1047     11   1720 

nr_free                    0      1      2      3      4      5      6      7     8      9     10
                       <---------------------------------------------------------------------------->
left                      1476  1253   842    590    663    827    557    361    265     3     487


===================================================================================================
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
   

===================================================================================================
Hugetlb Init:




















