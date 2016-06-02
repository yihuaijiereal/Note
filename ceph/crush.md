# Crush算法代码注释

## crush_rule

    struct crush_rule {
    	__u32 len;
    	struct crush_rule_mask mask;
    	struct crush_rule_step steps[0];
    };

crush_rule_step定义了crush算法的操作，其中的op可取如下值

    enum {
    	CRUSH_RULE_NOOP = 0,
    	CRUSH_RULE_TAKE = 1,          /* arg1 = value to start with */
    	CRUSH_RULE_CHOOSE_FIRSTN = 2, /* arg1 = num items to pick */
    				      /* arg2 = type */
    	CRUSH_RULE_CHOOSE_INDEP = 3,  /* same */
    	CRUSH_RULE_EMIT = 4,          /* no args */
    	CRUSH_RULE_CHOOSELEAF_FIRSTN = 6,
    	CRUSH_RULE_CHOOSELEAF_INDEP = 7,

    	CRUSH_RULE_SET_CHOOSE_TRIES = 8, /* override choose_total_tries */
    	CRUSH_RULE_SET_CHOOSELEAF_TRIES = 9, /* override chooseleaf_descend_once */
    	CRUSH_RULE_SET_CHOOSE_LOCAL_TRIES = 10,
    	CRUSH_RULE_SET_CHOOSE_LOCAL_FALLBACK_TRIES = 11,
    	CRUSH_RULE_SET_CHOOSELEAF_VARY_R = 12,
    	CRUSH_RULE_SET_CHOOSELEAF_STABLE = 13
    };

## Bucket
      Bucket Alg     Speed       Additions    Removals
       ------------------------------------------------
       uniform         O(1)       poor         poor
       list            O(n)       optimal      poor
       tree            O(log n)   good         good
       straw           O(n)       better       better
       straw2          O(n)       optimal      optimal
crush_bucket通用的属性如下，每种bucket还有各自独有的属性。

    struct crush_bucket {
    	__s32 id;        /* this'll be negative */
    	__u16 type;      /* non-zero; type=0 is reserved for devices */
    	__u8 alg;        /* one of CRUSH_BUCKET_* */
    	__u8 hash;       /* which hash function to use, CRUSH_HASH_* */
    	__u32 weight;    /* 16-bit fixed point */
    	__u32 size;      /* num items */
    	__s32 *items;

    	/*
    	 * cached random permutation: used for uniform bucket and for
    	 * the linear search fallback for the other bucket types.
    	 */
    	__u32 perm_x;  /* @x for which *perm is defined */
    	__u32 perm_n;  /* num elements of *perm that are permuted/defined */
    	__u32 *perm;
    };
## crush_map
    struct crush_map {
    	struct crush_bucket **buckets;
    	struct crush_rule **rules;

    	__s32 max_buckets;
    	__u32 max_rules;
    	__s32 max_devices;

    	/* choose local retries before re-descent */
    	__u32 choose_local_tries;
    	/* choose local attempts using a fallback permutation before
    	 * re-descent */
    	__u32 choose_local_fallback_tries;
    	/* choose attempts before giving up */
    	__u32 choose_total_tries;
    	/* attempt chooseleaf inner descent once for firstn mode; on
    	 * reject retry outer descent.  Note that this does *not*
    	 * apply to a collision: in that case we will retry as we used
    	 * to. */
    	__u32 chooseleaf_descend_once;

    	/* if non-zero, feed r into chooseleaf, bit-shifted right by (r-1)
    	 * bits.  a value of 1 is best for new clusters.  for legacy clusters
    	 * that want to limit reshuffling, a value of 3 or 4 will make the
    	 * mappings line up a bit better with previous mappings. */
    	__u8 chooseleaf_vary_r;

    	/* if true, it makes chooseleaf firstn to return stable results (if
    	 * no local retry) so that data migrations would be optimal when some
    	 * device fails. */
    	__u8 chooseleaf_stable;

    #ifndef __KERNEL__
    	/*
    	 * version 0 (original) of straw_calc has various flaws.  version 1
    	 * fixes a few of them.
    	 */
    	__u8 straw_calc_version;

    	/*
    	 * allowed bucket algs is a bitmask, here the bit positions
    	 * are CRUSH_BUCKET_*.  note that these are *bits* and
    	 * CRUSH_BUCKET_* values are not, so we need to or together (1
    	 * << CRUSH_BUCKET_WHATEVER).  The 0th bit is not used to
    	 * minimize confusion (bucket type values start at 1).
    	 */
    	__u32 allowed_bucket_algs;

    	__u32 *choose_tries;
    #endif
    };
## CrushWrapper
该类是对Crush的一个封装，包含了对Crush_map的各种操作。其中，do_rule方法返回一组OSD。

      void do_rule(int rule, int x, vector<int>& out, int maxout,
      	       const vector<__u32>& weight) const {
          Mutex::Locker l(mapper_lock);
          int rawout[maxout];
          int scratch[maxout * 3];
          int numrep = crush_do_rule(crush, rule, x, rawout, maxout, &weight[0], weight.size(), scratch);
          if (numrep < 0)
            numrep = 0;
          out.resize(numrep);
          for (int i=0; i<numrep; i++)
            out[i] = rawout[i];
        }

do_rule方法调用了mapper.h中的crush_do_rule方法

      /**
       * crush_do_rule - calculate a mapping with the given input and rule
       * @map: the crush_map
       * @ruleno: the rule id
       * @x: hash input
       * @result: pointer to result vector
       * @result_max: maximum result size
       * @weight: weight vector (for map leaves)
       * @weight_max: size of weight vector
       * @scratch: scratch vector for private use; must be >= 3 * result_max
       */
      int crush_do_rule(const struct crush_map *map,
      		  int ruleno, int x, int *result, int result_max,
      		  const __u32 *weight, int weight_max,
      		  int *scratch)；

crush_do_rule()方法调用crush_find_firstn()方法，crush_find_firstn()调用crush_bucket_choose()方法，该方法调用Robert Jenkins's hash 函数来选择一个osd。
