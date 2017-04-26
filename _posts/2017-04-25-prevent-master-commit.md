---
layout: post
title:  "Prevent Committing To Github Master Branch"
date:   2017-04-23 19:59:03 -0800
---

I did something embarrassing today - during work I accidentally committed my local branch to master, and broke the build for about 20 minutes. After some search, there are some very easy-to-use [approaches](https://gist.githubusercontent.com/stefansundin/9059706/raw/pre-commit) that utilizes Github's pre-commit hook.

There is still one minor issue - in my working laptop, there are multiple Git repositories, and manually adding the pre-commit files is a boring and tedious task. I thus wrote a simple [Python script](https://github.com/wang-ye/code/blob/master/python/setup_pre_commit_hook.py) to automate the job. Enjoy!