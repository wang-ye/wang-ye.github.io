Practical Rails Debugging Tips

Writing specs with transaction involved. 

Pry is an alternative to Ruby's interactive shell Irb,  Ipython to python shell. It provides a lot of features. 

Dynamic invocation with binding.pry

Navigation
standard navigation commands: next, step and continue
and a command to show the current execution line and its context

```
whereami or just use @
```

Show stack-trace


1. Set breakpoints
Conditional breakpoints


Live editing


## Pry Shortcuts
Start with the Pry cheetsheet: https://gist.github.com/lfender6445/9919357

  !!!                Alias for `exit-program`
  !!@                Alias for `exit-all`
  $                  Alias for `show-source`
  ?                  Alias for `show-doc`
  @                  Alias for `whereami`
  clipit             Alias for `gist --clip`
  file-mode          Alias for `shell-mode`
  history            Alias for `hist`
  jist               Alias for `gist`
  quit               Alias for `exit`
  quit-program       Alias for `exit-program`
  reload-method      Alias for `reload-code`
  show-method        Alias for `show-source`

play + wtf?
@ to show current code
$ to show source

## Setting breakpoints

cat --ex

## Showing DB Queries From ActiveRecord
Sometimes we want to know the actual DB queries ActiveRecord generates. This can be achieved in a simple command:

```
ActiveRecord::Base.logger = Logger.new(STDOUT)
```


## Understanding Rails Transactions

## What is Zeus