---
layout: post
title: Process ID Lookup
---

In this post, I’ll be explaining the functions used for PID lookup in the Linux kernel. The first function is next_pidmap(), which finds the 
first set bit in the current pidmap or the successor pidmaps corresponding to a namespace ns. Initially, there is a sanity check on the range to see if we have not exceeded the PID_MAX_LIMIT. Inside the for loop, we traverse from the current pidmap till the end of the pidmaps.

```C
int next_pidmap(struct pid_namespace *pid_ns, unsigned int last)
{
  int offset;
  struct pidmap *map, *end;
  if (last >= PID_MAX_LIMIT)
      return -1;
  offset = (last + 1) & BITS_PER_PAGE_MASK;
  map = &pid_ns->pidmap[(last + 1)/BITS_PER_PAGE];
  end = &pid_ns->pidmap[PIDMAP_ENTRIES];
  for (; map < end; map++, offset = 0) {
    if (unlikely(!map->page))
      continue;
    offset = find_next_bit((map)->page, BITS_PER_PAGE, offset);
    if (offset < BITS_PER_PAGE)
      return mk_pid(pid_ns, map, offset);
  }
  return -1;
}
```

The function find_next_bit returns the bit set as 1 in the current pidmap. If the offset is inside the BITS_PER_PAGE limit, meaning there is an entry in the pidmap which is set as 1, we return that pid. The unlikely provides hints to the compiler to adjust branch predictions.
find_pid_ns() is used to find the pid structure for the pid number nr. We iterate over the hash list whose head is returned by the pid_hash function. If found we return the struct pid that is related to the arguments nr and ns passed as parameters to the function.

```C
struct pid *find_pid_ns(int nr, struct pid_namespace *ns)
{
         struct upid *pnr;
 
         hlist_for_each_entry_rcu(pnr,
                        &pid_hash[pid_hashfn(nr, ns)], pid_chain)
                if (pnr->nr == nr && pnr->ns == ns)
                       return container_of(pnr, struct pid,
                                         numbers[ns->level]);
 
         return NULL;
}
find_vpid() finds the pid by its virtual id, i.e. in the current namespace.
struct pid *find_vpid(int nr)
{
         return find_pid_ns(nr, task_active_pid_ns(current));
}
find_get_pid looks up a PID in the hash table, and returns the struct pid with its count elevated. The count is incremented inside the get_pid function by calling atomic_inc(&pid->count);
struct pid *find_get_pid(pid_t nr)
{
         struct pid *pid;
 
         rcu_read_lock();
         pid = get_pid(find_vpid(nr));
         rcu_read_unlock();
 
         return pid;
}
```

find_ge_pid() returns the first allocated pid greater than or equal to. It uses two functions find_pid_ns and next_pidmap decribed above to do this. The function find_ge_pid() is used by fs/proc/base.c to find the pid greater than the pid “nr” passed as the argument to the function.
If there is a pid at nr this function is exactly the same as find_pid_ns.

```C
struct pid *find_ge_pid(int nr, struct pid_namespace *ns)
{
  struct pid *pid;
  do {
    pid = find_pid_ns(nr, ns);
    if (pid)
       break;
    nr = next_pidmap(ns, nr);
  } while (nr > 0);
  return pid;
}
```
