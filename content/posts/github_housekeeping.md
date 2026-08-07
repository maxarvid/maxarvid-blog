+++
title = "Github Housekeeping 🧹"
date = 2026-08-04
+++

Sometime ago I whined about keeping my local machine clean from git branches that were not present on a remote origin. I keep going back to [said post](https://maxarvid.com/posts/git-housekeeping/) because life is too short to memorize these types of scripts and creating some kind of util feels like overkill. And because I want to be able to run `git branch -a` and not cry blood.

Now I have cause to whine again. Imagine this scenario: You are the maintainer of some private codebase which is hosted on Github. Your workflow consists of raising Pull Requests and merging them, and it is your habit to delete the feature branches upon merge. But oh no, now it's 2026 and all of a sudden you are collaborating with various agentic contributors (human ones too), and sometimes people and agents forget to clean up after themselves. When you run `git branch -r` you find it difficult to understand what work is ongoing because the output is overwhelming. You are in need of some terminal commands to delete a bunch of branches on remote.

Prerequisite: In your desperation, you at some point installed the [Github-CLI](https://cli.github.com/) on your machine with the proper credentials.

Let's dive in. Let's be prudent and first get a list of branches associated with PRs either merged or closed (hint: they both have a state of "closed")
```sh
gh pr list --state closed --limit 100 --json headRefName --jq '.[].headRefName'
```
Github names the branch name property an intuitive `headRefName` (you know things are bad when you limit to 100 results, I hope you are not in the same boat). But wait a minute, what if we have a `develop` and a `main` branch? Won't this command return PRs that were releases to `main`? Yes. Yes it would, let's filter using `-- base`
```sh
gh pr list --state closed --base develop --limit 100 \
  --json headRefName --jq '.[].headRefName' \
  | sort -u
```
Switching over from the Github CLI to more OG git commands, let's normalize the `git branch -r` output and filter out some of the required branches (`main` and `develop` in the below example):
```sh
git branch -r \
  | sed 's|^[[:space:]]*origin/||' \
  | grep -v ' -> ' \
  | grep -Ev '^(develop|main)$' \
  | sort -u
```
Now it's just a matter of reaching for good old unix `comm`, which is so OG is has its own [Wikipedia page](https://en.wikipedia.org/wiki/Comm). Nice.
```sh
comm -12 \
  <(gh pr list --state closed --base develop --limit 500 --json headRefName --jq '.[].headRefName' | sort -u) \
  <(git branch -r | sed 's|^[[:space:]]*origin/||' | grep -v ' -> ' | grep -Ev '^(develop|main)$' | sort -u)
```
This will output an overlap list, that is to say, branches that are present on remote, that have been the compare branch for a *closed* PR. In the workflow described above, this is sufficient to delete these branches, but YMMV. It might be worthwhile to read through this list before going for the following steps. Now we need to reach for the scary looking `git push origin --delete branch-name`. To calm the nerves, a dry-run using `echo` is advised:
```sh
comm -12 \
  <(gh pr list --state closed --base develop --limit 500 --json headRefName --jq '.[].headRefName' | sort -u) \
  <(git branch -r | sed 's|^[[:space:]]*origin/||' | grep -v ' -> ' | grep -Ev '^(develop|main)$' | sort -u) \
| xargs -n1 echo git push origin --delete
```
This will print (echo) the commands tht we will pass to `xargs`. If you've come this far, please note that the following command **is a destructive action and cannot be undone**. Proceed at your own risk.
```sh
comm -12 \
  <(gh pr list --state closed --base develop --limit 500 --json headRefName --jq '.[].headRefName' | sort -u) \
  <(git branch -r | sed 's|^[[:space:]]*origin/||' | grep -v ' -> ' | grep -Ev '^(develop|main)$' | sort -u) \
| xargs -n1 git push origin --delete
```
And that's it! And if all this seems scary, just swap out the `--closed` with a `--merged` in the `gh pr list` command. Happy version control janitorial feelings all around.