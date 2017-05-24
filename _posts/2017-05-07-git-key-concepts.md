---
layout: post
title:  "Git Workflow Made Simple For Beginners"
date:   2017-05-07 17:49:03 -0800
---
After using Git for several years, I realized that for everyday work, we really only need to know ten Git commands. Of course, there are some fancy git tricks, but the beginners will rarely need them.
In this post I want to share my thoughts on learning git from *workflows*, a.k.a. the git commands you really need. I will start with basic git concepts, and then dive into a common workflow which covers only the absolutely necessary Git commands.

## Git Basic Concepts
First, some Git concepts. I will sometimes use a painting example to help understand.

1. In git, all changes happen in **branches**. You can also view them as the canvas of the painter.
2. Once you finish all the changes, you **commit**, meaning that the change is ready to merge into production. Similarly, this means the painter finishes a draft, and it is ready to be sold to the public!.
3. Git is a distributed system. There are two environments - **remote** and **local**. The commits are first made in local environment, and then you further push them to remote environment. Similarly, after a painter completes the work in the studio (local environment), he sends it to the art marketplace (remote environment).
4. [**Rebase**](https://git-scm.com/book/en/v2/Git-Branching-Rebasing) flow. Once your commit is in a remote develop branch, you will need to **merge** the branch to remote master branch. I personally prefer to rebase the changes, which moves the new commits on top of existing commits. This usually makes a cleaner, linear history. Whatever you like, just choose one and stick to it.
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

## Git Command Line Productivity
The above should be enough to get you started, but you still can get a big boost by having some more configurations. This section I will talk about the shell and git related configurations.

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

There are a couple of shortcuts to emphasize. For rebase workflow, a frequent action is getting your branch in sync with master by calling `git rebase`.  In the above we designed a composite command:

```shell
alias rbm="git checkout master && git pull --rebase origin master && git checkout - && git rebase master"
```

Most of the time `git add` is just an intermediate command towards committing the changes.  The following `gac` combines `add` and `commit` commands together:

```shell
alias gac='git add . && git commit -am '
```

*.gitconfig* contains the git specific settings: 

```.gitconfig
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

To illustrate, *rerere* is used for recording the previous rebase conflict resolution results, and apply them directly in the future. The *push* setting dictates that by default `git push` pushes local branch to a remote one with the same name. `kdiff3` is applied for resolving merge conflicts locally.

A brief note about other powerful git tools: in Mac, there is a [Github For Mac](). It has a built  -in workflow you can follow. In all modern IDEs, there are git Plugins. For command line tools, you can try [legit](https://github.com/kennethreitz/legit). BTW, this article is also deeply influenced by legit.

## Undo Changes, and Clean Up History
To work fluently with git, you also need to know  a couple more actions. One is undo the previous action, and the other is cleaning up the local history before committing to remote repository.

### Undo Previous Changes
You can also revert your changes easily, or modify your local history.
No matter which stages your change is in.

1. Existing file changes; not staged: `git checkout -- abc.py` to go back to the original file.
2. Changes staged but not committed: `git add abc.py; git reset HEAD abc.py`
3. Discard committed changes: `git reset --hard HEAD^`

### Clean Up Local History
Assume you make a very messy commit history containing many "style changes" and "removing extra lines". It is useless to keep these commits in the final git history. Here is the trick: you can merge or rename your local commits using `git rebase`. 
<!-- Clean up local history. -->

<!-- ## Stuff I Intentionally Omitted
If you are a experience Git user, you may notice that I omitted a lot of git commands, such as [stash]() and [cherry-pick](). Also, you can find my compond command gac, which simply skips the staging "git add" phase. 
There are multiple ways to achieve the same goal.
Like, you can merge or rebase.

## Summary
Trying to grasp all the useful commands all at once will only serve to confuse Git beginners. Instead, it is better to understand the basic Git concepts, and the typical workflow.
Later, you can tailer your Git toolbox to include commands like stash, cherrypick and many other things.
 -->