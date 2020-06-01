---
layout: post
title: Replacing PID implementation with IDR API
---

My project involves replacing bitmap implementation of PID allocation with IDR implementation. In an effort to do this, I made an exhaustive list of all the functions inside pid.c which will need replacement/deletion because they use bitmap (or are no longer needed). While this is not an ideal way to go about this problem, it helps me isolate what needs to be changed. In a while I will also have to figure out how to break my patches logically so that every patch in the series ensures that the kernel will not break. For now, here is the list:

- mk_pid() → Uses pidmap, to be removed
    — alloc_pidmap()
    — next_pidmap()
- free_pidmap() -> to be removed
    — alloc_pid()
    — free_pid()
- pid_before() → to be removed
    — set_last_pid()
- set_last_pid() → can be removed (Do not require to set last since IDR is used)
    — alloc_pidmap()
- alloc_pidmap() → to be removed
    — alloc_pid()
- next_pidmap() → to be removed
    — find_ge_pid()
    — zap_pid_ns_processes()
- put_pid() → does not use bitmap, can stay as it is.
    — delayed_put_pid()
    — Referenced in a lot of other driver and fs files.
- delayed_put_pid() → does not use bitmap, can stay as it is.
    — free_pid()
- free_pid() → Uses bitmap, replace implementation with IDR
    — __change_pid()
    — Referenced in fork.c and arch fiels
- alloc_pid() → Uses bitmap, replace implementation with IDR
    — kernel/fork.c
- disable_pid_allocation() → does not use bitmap, can stay as it is.
    — zap_pid_ns_processes()
    — alloc_pid()
- find_pid_ns() → Uses hlist, can stay as it is.
    — fs/fuse/file.c
    — find_vpid()
    — find_task_by_pid_ns()
    — find_ge_pid()
- find_vpid() → calls find_pid_ns()[uses hlist], can stay as it is
    — find_get_pid()
    — zap_pid_ns_processes
    — Used by other files in the kernel.
- attach_pid() -> Uses hashlist/rcu, can stay as it is
    — change_pid()
    — fork.c
- __change_pid() → Uses hashlist/rcu, can stay as it is
    — detach_pid()
    — change_pid()
- detach_pid() → Uses __change_pid which in turn uses hash list, can stay as it is
    — kernel/exit.c
- change_pid() → Uses hash list, can stay as it is
    — kernel/sys.c
    — fs/exec.c
- transfer_pid() → Uses hash list, can stay as it is
    — fs/exec.c
- pid_task() → Uses hlist, can stay as it is
    — find_task_by_pid_ns()
    — get_pid_task()
- find_task_by_pid_ns() → Uses rcu_list, can stay as it is
    — find_task_by_vpid()
- find_task_by_vpid() -> simply calls find_task_by_pid_ns(), can stay as it is
    — Used in a lot of places in the kernel
    — kernel/sys.c etc.
- get_task_pid() → uses rcu, can stay as it is
    — Used by a few driver files
    — kernel/exit.c
    — kernel/fork.c
- get_pid_task() → Can stay as it is.
    — Used by a few driver files
- find_get_pid() → Calls get_pid(), can stay as it is.
    — kernel/sysctl.c
    — A few other kernel files
- pid_nr_ns() , can stay as it is
    — pid_vnr()
    — __task_pid_nr_ns()
    — task_tgid_nr_ns()
    — A few other files in the kernel
- pid_vnr() → calls pid_nr_ns(), can stay as it is
- __task_pid_nr_ns() → can stay as it is
- task_tgid_nr_ns() → calls pid_nr_ns() can stay as it is
    — Used by fs/ and kernel/ files.
- task_active_pid_ns() -> calls ns_of_pid() can stay as it is
- find_ge_pid() → calls next_pidmap(), needs IDR implementation
- pidhash_init() → Uses hlist, can stay as it is
- pidmap_init() → Initialises pidmap, needs replacement with IDR API

I will list a few of the changes I made here. You will be able to see how and why using the IDR API is more effective as far the reducing the kernel size is concerned.
A few examples of how IDR is helping in reducing the size of the kernel:

``C
static void destroy_pid_namespace(struct pid_namespace *ns)
 {
-       int i;
-
        ns_free_inum(&ns->ns);
-       for (i = 0; i < PIDMAP_ENTRIES; i++)
-               kfree(ns->pidmap[i].page);
+
+       idr_destroy(ns->idr);
        call_rcu(&ns->rcu, delayed_free_pidns);
 }
```

Instead of individually removing pages from pidmap, we can simply destroy the idr struct associated with that namespace.
Inside the `find_ge_pid()` function, where we want to find the next greatest pid, earlier we used to traverse the pages in the pidmap to see if there is an allocated integer in the map, now a simple call to idr_get_next() would suffice.

```C
struct pid *find_ge_pid(int nr, struct pid_namespace *ns)
 {
-       struct pid *pid;
-
-       do {
-               pid = find_pid_ns(nr, ns);
-               if (pid)
-                       break;
-               nr = next_pidmap(ns, nr);
-       } while (nr > 0);
-
-       return pid;
+       return idr_get_next(ns->idr, &nr);
 }
```

We also got rid of functions like `alloc_pidmap()`, `free_pidmap()` etc. which managed the pidmap and allocations of the pages. IDR takes away this complexity by exposing the API which takes care of all of this.
In theory, using IDR should also be faster, since pidmap is linear time whereas IDR should be log time in the worst case. If this will be true in practice is something to be figured out as the next step.

As for now, here are the size metrics before and after IDR implementation. The kernel size seems to have shrunk nicely! :D

+--------+------------------------+------+------+-----+-------+----------+
|        |        filename        | text | data | bss |  dec  |   hex    |
+--------+------------------------+------+------+-----+-------+----------+
|        |                        |      |      |     |       |          |
| BEFORE | kernel/pid.o           | 8447 | 3894 |  64 | 12405 | 3075     |
| AFTER  |                        | 7667 | 2018 |  64 |  9749 | 2615     |
| BEFORE | kernel/pid_namespace.o | 5692 | 1842 | 192 |  7726 | 1e2e     |
| AFTER  |                        | 5682 | 1842 | 192 |  7716 | 1.00E+24 |
+--------+------------------------+------+------+-----+-------+----------+
