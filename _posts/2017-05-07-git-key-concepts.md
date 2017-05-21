Beginner Git Workflows Made Simple

When I started using Git, I got so confused. I have a svn background, and could not easily map the SVN concepts to git.

Instead of introducing a lot of Git commands, I will try to expose as few Git commands as possible. A simplified view of Git Workflow.

## Git Basic Concepts

After using Git for several years, I realized for our everyday work, we only need to know ten Git commands. Of course there are some fancy tricks, but you will very rarely need them. I want to share my thoughts on simplifying the Git usages.

Git Components - Branches, Commits and Repositories
In git, all changes happen in Branches.

Remote VS Local
Git is a distributed system. There is a central node - the remote repository. You can create a local repository. It talks with the remote repository.

Rebase VS Merge
What is rebase?
fast-forward to create a linear history.

One thing to remember is rebase usually make a cleaner, linear history. Whatever you like, just choose one and stick to it.
To contribute to the codebase, you first create a branch and make changes inside the branch, then create a commit for your change, and finally merge the commit to the repositories, which stores the codebase and its change history.

Commit History Graph
A rebase view:
How Git organize the commits? It can be viewed as a [graph](http://stackoverflow.com/questions/1057564/pretty-git-branch-graphs). Think it as a river. There is a main flow that keeps going forward, but occasionally, there could be branching, these branching can merge back to the main flow, or they can depart.



More specifically, a rebase workflow with a couple of steps, on top of Github.

## Simplified Git Workflow
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
You can simply click the merge button on Github
when the pull request is ready.
```

## Git Productivity Tips

The above should be enough to get you started. If you want to save a couple of key strokes, here is my shell shortcuts:

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

For .gitignore, you can find one online.
https://github.com/verekia/js-stack-from-scratch

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

## Making Workflow Even Simpler
In Git, you can achieve the same goal through multiple approaches. This, for beginners, will often cause confusions.

If you read some other Git tutorials, 

you will realize the concept of staging. I rarely find it useful.
Is this really worth the dedicated command and extra typing?

I instead always merge the add and commit into a single command

```
alias gac='git add . && git commit -am '
```

Another frequent action is picking up the latest changes for my current branch. This is especially useful to avoid merge conflict. The following composite command is used:

```
alias rbm="git checkout master && git pull --rebase origin master && git checkout - && git rebase master"
```

Stashing changes can be a powerful tool for some users.

## Git Productivity Tools

In Mac, there is a [Github For Mac](). It has builtin workflow you can follow.

In all modern IDEs, there are git Plugins. 

For command line tools, you can try [legit](). This article is also deeply influenced by legit.
it support only a small set of instructions:

There are multiple ways to achieve the same goal.
Like, you can merge or rebase.
Use add or not
Whether to use stash.

To resolve the conflicts

Legit
https://github.com/kennethreitz/legit
Git for human

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
