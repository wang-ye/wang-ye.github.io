Think in Workflows, Not in Commands - A pragmatic Git Guide

When I started using Git, I got so confused. I have a svn background, and could not map the Git concepts to SVN.

In git, changes happen in Branches.


To resolve the conflicts

0. Make a local branch

1. Make changes locally
Say you want to add a new feature to your website. The new feature is showing the FB trending news when your user visits your website.

2. Publish your changes to remote server

3. Merge your changes to production
Merge may causes conflict. To resolve the conflict, you can use either merge-based approach or rebase-based approach. I use rebase-based approach, which produces cleaner history. 

And you are done!!


Git add & git commit

git reflog

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

cherry-pick: apply one code snippet to another branch. copy paste

Resolve conflict tools

## Useful .gitconfig

Rerere config

Git rebase

For .gitignore, you can find one online.
https://github.com/verekia/js-stack-from-scratch

https://github.com/kennethreitz/legit
Git for human

## Powerful Git Tools
Github for Mac