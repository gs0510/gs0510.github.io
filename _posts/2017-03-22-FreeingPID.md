---
layout: post
title: Freeing Process ID
---

Allocating a free PID is essentially looking for the first bit in the bitmap whose value is 0; this bit is then set to 1. Conversely, freeing a PID can be implemented by ‘toggling’ the corresponding bit from 1 to 0.

A PID is generally visible in multiple namespaces. The kernel has to iterate through the different namespaces and “free” the PID individually from all the namespaces. Also, during this operation, we would want a lock on the pidmap so as to ensure atomic updates.

```C
spin_lock_irqsave(&pidmap_lock, flags);
```

`spin_lock_irq` does two things. It takes the lock (protecting against activities on other codes) and it turns off interrupts (protecting against activities on the same core in interrupt handlers). Somewhere later the lock will be released and interrupts will be restored. But it could be that interrupts were already turned off at the point of the spin_lock call. In that case, when we release the lock, we don’t want to turn interrupts on. spin_lock_irqsave protects against this by storing the incoming interrupt status in the flags argument. The corresponding unlock function checks the flags argument to decide whether to turn interrupts back on or leave them off.

```C
for (i = 0; i <= pid->level; i++) {
  struct upid *upid = pid->numbers + i;
  struct pid_namespace *ns = upid->ns;
  hlist_del_rcu(&upid->pid_chain);
  switch(--ns->nr_hashed) {
    case 2:
    case 1:
      wake_up_process(ns->child_reaper);
      break;
    case PIDNS_HASH_ADDING:
      /* Handle a fork failure of the first process */
      WARN_ON(ns->child_reaper);
      ns->nr_hashed = 0;
      /* fall through */
    case 0:
      schedule_work(&ns->proc_work);
      break;
  }
}
```

In the above code snippet, we iterate through all the PID levels, and at each level, we delete the pid hash from the rcu hash list. RCU(Read-Copy-Update) is a synchronization mechanism that allows reads to occur concurrently with updates. This hash was added in the `alloc_pid()` function using the following:

```C
hlist_add_head_rcu(&upid->pid_chain,
                   &pid_hash[pid_hashfn(upid->nr, upid->ns)]);
```

If the namespace has only one or two PIDs left, we want to clear the namespace. For every namespace, we have a field type `nr_hashed` which stores the number of PIDs that have been hashed to a particular namespace. If the number of PIDs that have been hashed to the namespace are one or two: We wake up the reaper. The reaper does memory management.

We wake up the reaper even when the number of hashed ids are 2, because one of the processes that might be exiting may be `init()`. It’s possible to get into a situation where a pid->ns reaper is `<defunct>`, re-parented to host pid 1, but never reaped. The `wake_up` call to the reaper will be spurious if init is not being exited, and is a waste of time but not a correctness issue.

If the number of PIDs hashed is 0 or the `nr_hashed` has the same value as when it was initialized(`PIDNS_HASH_ADDING`), we schedule cleaning up of the namespace which after a few function calls, does a `kern_unmount()`.

```C
for (i = 0; i <= pid->level; i++)
  free_pidmap(pid->numbers + i);
```

The next step is to call `free_pidmap` which simply clears the bit pertaining to that pid using the `clear_bit()` function. The code inside `free_pidmap()` is listed below.

```C
struct pidmap *map = upid->ns->pidmap + upid->nr / BITS_PER_PAGE;
int offset = upid->nr & BITS_PER_PAGE_MASK;
clear_bit(offset, map->page);
```

The function `free_pid()` finally calls `call_rcu(&pid->rcu, delayed_put_pid);`

The use of `call_rcu()` permits the caller of `free_pid()` to immediately regain control, without needing to worry further about the old version of the newly updated element. In other words, use of `call_rcu()` to ensure that any readers that might have references to the old pid structure complete before freeing the pid.
