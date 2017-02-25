Rails Debugging Tips
Ruby on rails (ROR) is a powerful framework with limited documentations. In this post I wanted to share some useful tips when debugging ROR.

## Switch To Pry First
Pry is an alternative to Ruby's interactive shell, **Irb**. It provides a lot more features, including source code and documentation browsing, syntax highlighting, dynamic invocation, and most importantly, it contains various plugins for debugging Ruby and Rails programs. Switch to Pry early and enjoy these features! The tips here assumes you use Pry as your interactive shell.

## Debugging Environment Setup
For MRI version < 2.0, use [pry-debugger](https://github.com/nixme/pry-debugger). It is also recommended to use [pry-stack_explorer](https://github.com/pry/pry-stack_explorer). Otherwise, use [pry-byebug](https://github.com/deivid-rodriguez/pry-byebug).

## Starting a Debug Sessions and Breakpoints
Here we assume you will trigger a debugging session through rspec tests.
Setting a breakpoint in source code is simple - just call **binding.pry**! The role of "binding.pry" is the same as "debugger" in Javascript and "pdb.set_trace()" in Python.

On the other hand, one can also specify dynamic breakpoints using "break" command. You can break on a instance method in a class, or on a specific line of a file or break when certain condition meets. Here are some examples:

```
break SomeClass#run            Break at the start of `SomeClass#run`.
break Foo#bar if baz?          Break at `Foo#bar` only if `baz?`.
break app/models/user.rb:15    Break at line 15 in user.rb.
break 14                       Break at line 14 in the current file.
break --condition 4 x > 2      Change condition on breakpoint #4 to 'x > 2'.
break                          List all breakpoints. (Same as `breakpoints`)
```

Detailed instructions can be found [here](https://github.com/nixme/pry-debugger).

## Code Navigation
standard navigation commands: next, step and continue
and a command to show the current execution line and its context

Some of the useful commands:

```
whereami or just @               : Show the current code execution location
show-method User#name            : Show name method of User class
Show stack-trace                 : Print the stack-trace information
```

## Rerun Failed Tests
It is easy to rerun the failed test cases from command line. For example, you can call the following to run the failed tests starting froml line 4 of example_spec.rb.

```Ruby
rspec ./spec/example_spec.rb:4 
```

## Live Source Editing
Pry offers 'edit' command, where you can invoke your default editor. Some useful commands are:

```
edit hello.rb:10               : Editor goes to line 10 of hello.rb
edit Class#a_method            : Editor goes to a_method in Class
edit Class.a_class_method      : Editor goes to a_class_method in Class
edit --ex                      : Editor goes to the last exception place.
```

[Here](https://github.com/pry/pry/wiki/Editor-integration#Edit_command) you can find the detailed instructions.

## Showing More Logging in Pry Console
In Rails we often need to work with ActiveRecord. Sometimes we want to know the actual DB queries ActiveRecord executes. This can be achieved with a simple command in pry:

```
ActiveRecord::Base.logger = Logger.new(STDOUT)
```

## Summary
Pry offers powerful plugins for debugging. You can navigate code, set dynamic breakpoints, inspect variable and program states, rerun failed tests, check stack traces. You can even live editing the code. Enjoy it!