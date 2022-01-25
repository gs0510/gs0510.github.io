---
layout: post
title: Replacing PID bitmap implementation with IDR API- Final Stats
---

|  #     |        filename          | text | data | bss |  dec  |   hex    |
|:---    |:-------------------------|:-----|:-----|:----|:------|:---------|
|        |                          |      |      |     |       |          |
| BEFORE | kernel/pid.o             | 8447 | 3894 |  64 | 12405 | 3075     |
| AFTER  |                          | 3397 | 304  |   0 | 3701  |  e75     |
| BEFORE | kernel/pid_namespace.o   | 5692 | 1842 | 192 |  7726 | 1e2e     |
| AFTER  |                          | 2854 | 216  | 16  |  3086 | c0e      |

#### ps

|   #  | With IDR API  |  With bitmap |
|:-----|:--------------|:-------------|
|real  |  0m1.962s     |   0m2.319s |
|user  |  0m0.052s     |   0m0.060s |
|sys   |  0m0.392s     |   0m0.516s |

#### pstree

|    #  | With IDR API  |  With bitmap |
|:------|:--------------|:-------------|
|real   | 0m1.062s      |  0m1.794s |
|user   | 0m0.536s      |  0m0.612s |
|sys    | 0m0.184s      |  0m0.264s |

### proc

|  #    | With IDR API   | With bitmap |
|:------|:---------------|:------------|
|real   | 0m0.073s       | 0m0.074s |
|user   | 0m0.004s       | 0m0.004s |
|sys    | 0m0.012s       | 0m0.016s |
