Think in Workflows, Not in Commands - A pragmatic Git Guide

When I started using Git, I got so confused. I have a svn background, and could not easily map the SVN concepts to git. As a retrospection, there are two reasons: a conceptual model for Git, and a simple workflow to follow.

After using Git for a while, I finally realized for the everyday work, we only need to know a couple of Git commands. I want to share my thoughts on simplifying the Git usages.

## Conceptual Model

Git has some basic concepts:

1. In git, all changes happen in Branches.
3. Once you make all the changes, you commit, meaning that the change is ready to merge into production.
2. Git is a distributed system. Remote and Local. A mapping branch. Tracking branch.
4. Git history is a directed graph, each node being a commit.

More specifically, a rebase workflow with a couple of steps, on top of Github.

## Basic Workflow

1. Create a local branch
In Git, work is done in *branches*. Say you want to add a new feature to your website. You will want to first create a local branch, which can be an exact copy of the production/develop code, and only visible to yourself. Then you make corresponding changes in this branch. Once you are done, you **commit** the changes. Note that, till this moment the changes are only visible to yourself.

Useful commands:

```
git rebase master
# Create a branch feature-x-branch, which is an exact copy 
# of the master branch.
git checkout -b feature-x-branch master  
# Open abc.py and make some changes.
vi abc.py
# Add abc.py to your local commit.
git add abc.py && git commit -m 'First version for X feature'
```

2. Publish your commit to remote server
When you are confident the code will make a positive impact to the product, you can push your changes to remote server, which in our case is Github enterprise servers. Now, your code is available for peer reviews. Other engineers will give you comments and request for new features. You will further improve your code by addressing their comments and concerns. You may also see some bullshit reviews full of adding or removing some blank lines. Working with these stupid reviewers is beyond the scope of this post though.

Useful commands:

```
# Assuming you have set up the remote tracking branch
# We will discuss more later.
git push
# Iterate on the code; address reviewer comments.
vi abc.py
# Commit again
git add abc.py && git commit -m 'Handle edge case Y'
# Push and request for reviews again, repeat the 
# change-push-review until we are all happy.
git push
...
```

3. Merge your changes to production
OK. After several rounds of reviews, your code is ready to merge into production! There are still a couple of things to consider though. You realize that you have a very messy commit history containing many "addressing style changes" and "removing extra lines". It is useless to keep these commits in the history. You can merge or rename your local commits if needed.
An even more important question is conflict resolution. To resolve the conflict, you can use either merge-based approach or rebase-based approach. I use rebase-based approach, which produces a linear history.

And we are done!!

Useful tips:

```
You can simply click the merge button on Github when the pull request is ready.
```

## Git Productivity Tips

There are multiple ways to achieve the same goal.

Like, you can merge or rebase.

Use add or not

Whether to use stash.

Git add & git commit

git reflog

To resolve the conflicts

What is git stash

staging helps you split up one large change into multiple commits
Let's say you worked on a large-ish change, involving a lot of files and quite a few different subtasks. You didn't actually commit any of these -- you were "in the zone", as they say, and you didn't want to think about splitting up the commits the right way just then. (And you're smart enough not to make the whole thing on honking big commit!)
Now the change is all tested and working, you need to commit all this properly, in several clean commits each focused on one aspect of the code changes.
With the index, just stage each set of changes and commit until no more changes are pending. Really works well with git gui if you're into that too, or you can use git add -p or, with newer gits, git add -e.

--Set-upstream to link local and remote
Origin

Stash really necessary?

## Other frequent actions

You can also revert your changes easily, or modify your local history.
No matter which stages your change is in.

Changes not staged: Git checkout -- abc.py go back to the original file

Changes staged but not committed: 
# Git add abc.py; git reset HEAD abc.py
Git checkout HEAD abc.py

git reset -- files 用来撤销最后一次git add files，你也可以用git reset 撤销所有暂存区域文件。

Committed files: git reset --hard HEAD^

Checkout VS reset: http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001374831943254ee90db11b13d4ba9a73b9047f4fb968d000#0

Clean up local history? git rebase

Resolve conflict tools

## Useful .gitconfig and Shell Config

Rerere config

Git rebase

For .gitignore, you can find one online.
https://github.com/verekia/js-stack-from-scratch

## Powerful Git Tools
Github for Mac

Legit
https://github.com/kennethreitz/legit
Git for human
