Every time I started the Pycharm debugger, I wanted to understand how it works internally. So finally after some Googling I found the series of posts Eli Bendersky [https://eli.thegreenplace.net/2011/01/23/how-debuggers-work-part-1]. These posts answered a lot of my questions I am just going to write a short note to summarize my understanding at a high level.

### A bit of theory
Ptrace: Used solely in Linux systems. Mac OS also uses it.
Ptrace can control child process executions. It provides ways for the main process to interact with the child process running the actual program you want to debug.
Interrupts:

### How to set breakpoints

### Inspect vars

### Program flow control

## Summary
Please read the original posts for more info.