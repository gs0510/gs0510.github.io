---
layout: post 
title: PID Allocation in Linux Kernel
---

### Process ID

Every process has a unique identifier which it is represented by, called as the process ID(pid). The first process that the kernel runs is called the idle process and has the pid 0. The first process that runs after booting is called the init process and has the pid 1.
The default maximum value of the pid on a linux 32678. This can be checked by running the command:

`$cat /proc/sys/kernel/pid_max`

This default value ensures compatibility with older systems which used 16 bit types for process IDs. One can increase the maximum pid value by writing the the number to /proc/sys/kernel/pid_max. This ensures larger pid space at the expense of reduced compatibility. On 64-bit systems, pid_max can be set to any value up to 2²² (PID_MAX_LIMIT, approximately 4 million).

### Process ID Namespaces

PID Namespaces isolate the process ID number space, meaning that processes in different PID namespaces can have the same PID. PID namespaces allow containers to provide functionality such as suspending/resuming the set of processes in the container and migrating the container to a new host while the processes inside the container maintain the same PIDs. PID namespaces are hierarchically nested in parent-child relationships. Within a PID namespace, it is possible to see all other processes in the same namespace, as well as all processes that are members of descendant namespaces. Processes in a child PID namespace cannot see processes that exist (only) in the parent PID namespace (or further removed ancestor namespaces). A process will have different PID in each of the layers of the PID namespace hierarchy starting from the PID namespace in which it resides through to the root PID namespace. Calls to getpid() always report the PID associated with the namespace in which the process resides.

### Process ID Structure

```C
struct upid {
 int nr;
 struct pid_namespace *ns;  /* the namespace this value
                             * is visible in
                             */
 struct hlist_node pid_chain;  
};
struct pid {
         atomic_t count;
         unsigned int level;
         /* lists of tasks that use this pid */
         struct hlist_head tasks[PIDTYPE_MAX];
         struct rcu_head rcu;
         struct upid numbers[1];
 };
```

This structure contains the ID value, the list of tasks having this ID, the reference counter and the hashed list node to be stored in the hash table for a faster search.

### How are the process IDs allocated?

The two main functions called inside the function alloc_pid(namespaces) are kmem_cache_alloc(…) and alloc_pidmap(…)

```C
...
pid = kmem_cache_alloc(ns->pid_cachep, GFP_KERNEL);
...
```

The alloc_pid function calls the kmem_cache_alloc function, with the cache for the namespace as the parameter. kmem_cache_alloc keeps a cache of pre-allocated structures. Since, the pid allocation is frequently done, instead of allocating pid struct from the main memory(kmalloc), we keep a cache, and when a new pid struct is required, kmem_cache_alloc returns the address of a block that was already allocated.

```C
struct pid *alloc_pid(struct pid_namespace *ns)
{
    ...
    for (i = ns->level; i >= 0; i--) {
        nr = alloc_pidmap(tmp);
        if (nr < 0)
            goto out_free;
        pid->numbers[i].nr = nr;
        pid->numbers[i].ns = tmp;
        tmp = tmp->parent;
    }
    ...
}
```

A process will have one PID in each of the layers of the PID namespace hierarchy starting from the PID namespace in which it resides through to the root PID namespace. Hence, once alloc_pid gets the pid structure, we are required to assign the process pid in all namespaces. We iterate through all namespaces and call alloc_pidmap with the namespace as the parameter.
struct upid is used to get the id of the struct pid, as it is seen in particular namespace. Each upid instance is put on the PID hash list.

```C
struct pid *alloc_pid(struct pid_namespace *ns)
{
    ...
    for ( ; upid >= pid->numbers; --upid) {
        hlist_add_head_rcu(&upid->pid_chain                                                                                          ,                          &pid_hash[pid_hashfn(upid->nr
,                           upid->ns)]);
    upid->ns->nr_hashed++;
}
```

alloc_pid function, which is tasked with allocating the pids, makes a call to the function alloc_pidmap which searches for the next pid in the bitmap.

```C
static int alloc_pidmap(struct pid_namespace *pid_ns)
{
        int i, offset, max_scan, pid, last = pid_ns->last_pid;
        struct pidmap *map;
        pid = last + 1;
        if (pid >= pid_max)
                pid = RESERVED_PIDS;
        ...
}
```

last is the last pid that was allocated in this namespace. We assign the next pid serially. PIDs are allocated in the range of (RESERVED_PIDS, PID_MAX_DEFAULT).

```C
if (unlikely(!map->page)) {
    void *page = kzalloc(PAGE_SIZE, GFP_KERNEL);
    /*
     *Free the page if someone raced with us
     *installing it:
     */
     spin_lock_irq(&pidmap_lock);
     if (!map->page) {
         map->page = page;
         page = NULL;
     }
     spin_unlock_irq(&pidmap_lock);
     kfree(page);
     if (unlikely(!map->page))
         return -ENOMEM;
}
```

If no memory has been allocated to the bitmap, allocate it the memory using kzalloc. If the bitmap is still not allocated memory (checked using unlikely function), return a No Memory Available(ENOMEM) error.
```C
if (likely(atomic_read(&map->nr_free))) {
    for ( ; ; ) {
        if (!test_and_set_bit(offset, map->page)) {
                atomic_dec(&map->nr_free);
                set_last_pid(pid_ns, last, pid);
                return pid;
            }
            offset = find_next_offset(map, offset);
            if (offset >= BITS_PER_PAGE)
                 break;
            pid = mk_pid(pid_ns, map, offset);
            if (pid >= pid_max)
                 break;
    }
}
```

Here, we keep iterating through the bitmap until we find a pid or the offset exceeds the pagesize or if pid reaches the PID_MAX_DEFAULT limit.

```C
static int alloc_pidmap(struct pid_namespace *pid_ns)
{
    ...
    if (map < &pid_ns->pidmap[(pid_max-1)/BITS_PER_PAGE]) {
        ++map;
        offset = 0;
    } else {
        map = &pid_ns->pidmap[0];
        offset = RESERVED_PIDS;
        if (unlikely(last == offset))
            break;
    }
    pid = mk_pid(pid_ns, map, offset);
    ...
}ea
```

In the previous section, when the offset exceeds the page size, we move onto a new bitmap with the offset 0. Otherwise, if the pid reached the maximum default limit, pid assignment wraps around. Finally, we get the pid value from the bitmap using the function mk_pid.