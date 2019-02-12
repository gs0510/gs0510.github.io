---
layout: post
title: git-bisect: A savior in times of trouble!
---

Things used to work, now they don't anymore. Somebody pushed a rogue commit, and you want to find the culprit. The solution to this problem is easier than you'd imagine. 

To put it simply, git-bisect performs a binary search across a good commit and a bad commit.  At each step it tries to reduce the number of revisions that are potentially bad by half.

To use git bisect:
```
$ git bisect start
$ git bisect bad //this means the current revision
$ git bisect good <good_hash>
```

After these steps, test your changes. Depending on the tests, mark the current revision as good/bad using:
```
git bisect good|bad
```

Eventually there will be no more commits to evaluate, and you'll find out the dodgy commit. Huzzah!

Here's a drawing that you can use as a cheat sheet. :)

![git bisect drawing](/images/git-bisect.png)
