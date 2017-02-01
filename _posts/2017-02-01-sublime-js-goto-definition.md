---
layout: post
title:  "Sublime Goto Definition Hack for Javascript"
date:   2017-02-01 07:40:01 -0800
---
I used Sublime when developing Javascript. One big plus for coding in Sublime is it has a builtin support for `goto definition`. You can find the option in both the context menu and the Goto dropdown menu. This feature works pretty well for Python, but not so in Javascript. Every time, my "goto_definition"  command in Javascript gives me everything except the function definition. 
I spent a long time and still could not find the exact reason. Maybe this is a bug, or maybe this was caused by conflicts of plugins. As a quick hack for this, I enabled `goto_symbol_in_project`

```Json
  { "keys": ["ctrl+alt+r"], "command": "goto_symbol_in_project"},
```

This will perform a generic search including both the usages and the function definition. What's more, it also supports fuzzy search. 

![]({{ site.url }}/assets/goto_symbol_project.png)
