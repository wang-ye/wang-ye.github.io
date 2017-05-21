---
layout: post
title:  "Git Workflow Made Simple For Beginners"
date:   2017-05-07 17:49:03 -0800
---
After using Git for several years, I realized that for our everyday work, we really only need to know ten Git commands. Of course, there are some fancy git tricks, but most of the beginners will rarely need them.
In this post I want to share my thoughts on simplifying the Git usages. I will start with basic git concepts, and then we will dive into a common workflow and introduce the absolutely necessary git commands to fullfill this flow.

## Git Basic Concepts
OK, let's start with the Git concepts. I will use painting concepts to help illustrate.

1. In git, all changes happen in **branches**. You can also view the branches as the canvas of the painter.
2. Once you finish all the changes, you **commit**, meaning that the change is ready to merge into production. Similarly, this means the painter finishes a draft, and it is ready to be published to the public!.
3. Git is a distributed system. There are two environments - **remote and local**. The commits are first made in local environment, and then you further submit the changes to remote environment. Similarly, when a painter's artwork is completed at your home (local environment), you send it to the art marketplace (remote environment).
4. [**Rebase**](https://git-scm.com/book/en/v2/Git-Branching-Rebasing) flow. Once the changes is in a remote develop branch, you will need to **merge** the branch to remote master branch. I personally prefer to rebase the changes, which moves the new commits on top of existing commits. This usually makes a cleaner, linear history. Whatever you like, just choose one and stick to it.
5. Each repository has its git change history, which is a directed graph with the commits being the nodes. You can move back and forth along the history, and create new branches based on any commit in the history.

## A Workflow That Works
In this section I will focus on a workflow that fits most of our needs. Basically, to contribute to codebase, you first create a local branch and make necessary changes, then create a local commit for your change, and finally merge the commit to the remote repositories, which stores the codebase and its change history.

### Create a local branch
In Git, work is done in *branches*. Say you want to add a new feature to your website. You will want to first create a local branch, which can be an exact copy of the production/develop code, and only visible to yourself. Then you make corresponding changes in this branch. Once you are done, you **commit** the changes. Some useful tips:

```
# Create a branch feature-x-branch, which is an exact copy 
# of the master branch.
git checkout -b feature-x-branch master  
# Open abc.py and make some changes.
vi abc.py
# Add abc.py to your local commit.
git add abc.py && git commit -m 'First version for X feature'
```

### Publish local commit to remote server
When you are confident the change is good, you can push it to remote server, which in our case is Github enterprise servers. Now, your code is available for peer reviews. Other engineers will give you comments and possibly request new changes. You will further update your code by addressing reviewers' comments and concerns. Some useful tips:

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
```

### Merge your changes to production
Finally, after several rounds of reviews, your code is ready to merge into production. And we are done!! Some useful tips:

```
If you use Github, simply click the merge button on
Github when the pull request is ready.
```
<!-- 
## Git Productivity and Toolkits
The above should be enough to get you started, but you still can get a big boost by having some configuration changes. This section I will discuss the shell and git related configurations.

Here are my shell shortcuts to save a couple of key strokes:

```shell
alias gs='git status '
alias ga='git add '
alias gb='git branch '
alias gc='git commit'
alias gd='git diff'
alias gco='git checkout '
alias gp='git push'
alias gpf='git push -f'
alias gac='git add . && git commit -am '
alias rbm="git checkout master && git pull --rebase origin master && git checkout - && git rebase master"
```

For rebase workflow, a frequent action is getting your branch to be in sync with master to pick up the latest changes and resolve the potential conflicts.
Another frequent action is picking up the latest changes for my current branch. This is especially useful to avoid merge conflict. The following composite command is used:

```
alias rbm="git checkout master && git pull --rebase origin master && git checkout - && git rebase master"
```

I instead always merge the add and commit into a single command

```
alias gac='git add . && git commit -am '
```

In .gitconfig, some important entries 

```shell
  [color]
    ui = auto
  [core]
    editor = vi
    excludesfile = ~/.gitignore
    precomposeunicode = true
  [push]
    default = current
  [rerere]
    enabled = 1
  [diff]
    tool = kdiff3
  [merge]
    tool = kdiff3
```

rerere is used for recording the previous rebase conflict resolution, and apply it directly.

push set up the remote tracking to the branch with the same name.
For .gitignore, you can find one online.
https://github.com/verekia/js-stack-from-scratch

In Mac, there is a [Github For Mac](). It has builtin workflow you can follow.

In all modern IDEs, there are git Plugins. 
To resolve the conflicts

For command line tools, you can try [legit](). This article is also deeply influenced by legit.
it support only a small set of instructions:

There are multiple ways to achieve the same goal.
Like, you can merge or rebase.
Use add or not
Whether to use stash.

Legit
https://github.com/kennethreitz/legit
Git for human

In Git, you can achieve the same goal through multiple approaches. This, for beginners, will often cause confusions.

If you read some other Git tutorials, 

you will realize the concept of staging. I rarely find it useful.
Is this really worth the dedicated command and extra typing?

Stashing changes can be a powerful tool for some users.

## Other Common Actions
In this section we discuss two frequent actions: undo the previous action, and clean up the local history before committing to remote repo.

### Undo Previous Action
You can also revert your changes easily, or modify your local history.
No matter which stages your change is in.

Changes not staged: Git checkout -- abc.py go back to the original file

Changes staged but not committed: 
# Git add abc.py; git reset HEAD abc.py
Git checkout HEAD abc.py

git reset -- files 用来撤销最后一次git add files，你也可以用git reset 撤销所有暂存区域文件。

Committed files: git reset --hard HEAD^

Checkout VS reset: http://www.liaoxuefeng.com/wiki/0013739516305929606dd18361248578c67b8067c8c017b000/001374831943254ee90db11b13d4ba9a73b9047f4fb968d000#0

### Clean Up Local History
There are still a couple of things to consider though. You realize that you have a very messy commit history containing many "addressing style changes" and "removing extra lines". It is useless to keep these commits in the history. You can merge or rename your local commits if needed.
An even more important question is conflict resolution. To resolve the conflict, you can use either merge-based approach or rebase-based approach. I use rebase-based approach, which produces a linear history.
Clean up local history? git rebase

Resolve conflict tools

## Stuff I Intentionally Omitted
If you are a experience Git user, you may notice that I omitted a lot of git commands, such as [stash]() and [cherry-pick](). Also, you can find my compond command gac, which simply skips the staging "git add" phase. 
There are multiple ways to achieve the same goal.
Like, you can merge or rebase.

git reflog

To resolve the conflicts

--Set-upstream to link local and remote
Origin

Stash really necessary?

## Summary
Trying to grasp all the useful commands all at once will only serve to confuse Git beginners. Instead, it is better to understand the basic Git concepts, and the typical workflow.
Later, you can tailer your Git toolbox to include commands like stash, cherrypick and many other things.
 -->