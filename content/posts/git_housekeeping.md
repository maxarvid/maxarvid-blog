+++
title = "Git Housekeeping 🧹"
date = 2024-10-15
+++

There are many instruction online outlining how to get rid of git branches that were merged to a remote. What I've not encountered however is how to get rid of local branches that lack a remote counterpart. On my local machine I have a tendency to experiment in a fresh branch, commit a couple of things, then get distracted and work on something else. The local branches proliferate, unmerged, unloved, and forgotten. 

My desire to go though all of these one-by-one and delete them by hand is extremely limited. Surely there is some command that can be run to list all the local branches not present on the remote and then pipe those into a `git branch -d` or `git branch -D`.

The `comm` utility is our friend here. It takes two files (in our case some process substitution with `<()`), sorted lexically. The output is three text columns, but we will suppress two of those with some options, namely `-2` to suppress lines only present in the second file and `-3` to suppress lines in both "files".

To format the rows we'll use the stream editor utility, `sed`. 

To appease the lexical sorting required by `comm`, we'll use the `sort` utility. Let's slap it all together with some pipes:
```sh
comm -23 <(git branch | sed 's/..//' | sort) <(git branch -r | sed 's/..//' | sed 's/origin\///' | sort)
```
Running this outputs the (in my case) considerable list of branches that I only have on my machine. Piping this into a `git branch -d` can be achieved by using the `xargs` utility:
```sh
comm -23 <(git branch | sed 's/..//' | sort) <(git branch -r | sed 's/..//' | sed 's/origin\///' | sort) | xargs -n 1 git branch -d
```
Now in my case, I need to run it with `-D` and not `—d` since I've never merged these branches. 
## TLDR;
List local branches not present on remote:
```sh
comm -23 <(git branch | sed 's/..//' | sort) <(git branch -r | sed 's/..//' | sed 's/origin\///' | sort)
```
Delete local branches not present on remote that have been merged:
```sh
comm -23 <(git branch | sed 's/..//' | sort) <(git branch -r | sed 's/..//' | sed 's/origin\///' | sort) | xargs -n 1 git branch -d
```
Delete local *unmerged* branches not present on remote:
```sh
comm -23 <(git branch | sed 's/..//' | sort) <(git branch -r | sed 's/..//' | sed 's/origin\///' | sort) | xargs -n 1 git branch -D
```