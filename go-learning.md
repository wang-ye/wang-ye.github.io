---
layout: post
title:  "Start Go Learning"
date:   2017-08-25 19:59:03 -0800
---

As a 2017 New Year resolution, I planned to get familiar with [Golang](). This week I finally started my Go journey!

Instead of reading the syntax one by one, I jumped into something a bit more interesting - [Go by Examples](www.gobyexample.com). In this blog, I want to talk about some initial feelings when learning Go. For me, it seems like an enhancement of C language with concurrency in mind. It also shares some of the designs from Python. Some of the most surprising features to me are:

## Go routines and Channels
It arguably is the most powerful feature in Go. As an example from *Go By Examples*:

```go
// From https://gobyexample.com/channels
package main

import "fmt"

func main() {
    messages := make(chan string)
    go func() { messages <- "ping" }()
    msg := <-messages
    fmt.Println(msg)
}
```

Go routines and channels provide a concurrency model using the channel abstraction.

## Defer
One big step to make sure we clean up properly. Go advocates a pattern to free resources immediately after resource allocation.

```go 
package main

func main() {
    f := createFile("/tmp/defer.txt")
    defer closeFile(f)
    writeFile(f)
}
```

3. Confusing Command Line Parsing
I am really confused when reading the command line parsing for Go. As an example:

```go 
// From https://play.golang.org/p/NASEOq2R3n
import "flag"
...
func main() {
    wordPtr := flag.String("word", "foo", "a string")
    flag.Parse()
    fmt.Println("word:", *wordPtr)
}
```

The above code looks so magical to me - what does *flag.Parse()* do? What is *wordPtr*? It turned out it does tons of things in the background. Inside the *flag* package, a variable called CommandLine is created and holds all the flags.

```go
// A FlagSet represents a set of defined flags. The zero value of a FlagSet
// has no name and has ContinueOnError error handling.
type FlagSet struct {
    // Usage is the function called when an error occurs while parsing flags.
    // The field is a function (not a method) that may be changed to point to
    // a custom error handler.
    Usage func()
  
    name          string
    parsed        bool
    actual        map[string]*Flag
    formal        map[string]*Flag
    args          []string // arguments after flags
    errorHandling ErrorHandling
    output        io.Writer // nil means stderr; use out() accessor
}
var CommandLine = NewFlagSet(os.Args[0], ExitOnError)
```

When ``String`` method is called, a flag is created. *CommandLine* now knows what to expect when parsing the arguments.

```go 
// String defines a string flag with specified name, default value, and usage string.
// The return value is the address of a string variable that stores the value of the flag.
func (f *FlagSet) String(name string, value string, usage string) *string {
    p := new(string)
    f.StringVar(p, name, value, usage)
    return p
}

// StringVar defines a string flag with specified name, default value, and usage string.
// The argument p points to a string variable in which to store the value of the flag.
func StringVar(p *string, name string, value string, usage string) {
    CommandLine.Var(newStringValue(value, p), name, usage)
}

// Var defines a flag with the specified name and usage string. The type and
// value of the flag are represented by the first argument, of type Value, which
// typically holds a user-defined implementation of Value. For instance, the
// caller could create a flag that turns a comma-separated string into a slice
// of strings by giving the slice the methods of Value; in particular, Set would
// decompose the comma-separated string into the slice.
func (f *FlagSet) Var(value Value, name string, usage string) {
    // Remember the default value as a string; it won't change.
    flag := &Flag{name, usage, value, value.String()}
    _, alreadythere := f.formal[name]
    if alreadythere {
        var msg string
        if f.name == "" {
            msg = fmt.Sprintf("flag redefined: %s", name)
        } else {
            msg = fmt.Sprintf("%s flag redefined: %s", f.name, name)
        }
        fmt.Fprintln(f.out(), msg)
        panic(msg) // Happens only if flags are declared with identical names
    }
    if f.formal == nil {
        f.formal = make(map[string]*Flag)
    }
    f.formal[name] = flag
}
```

The real magic is the *Parse* method. It utilizes *os* package to get all the command line arguments, and parse them one by one and store them inside CommandLine.

```go 
// Parse parses flag definitions from the argument list, which should not
// include the command name. Must be called after all flags in the FlagSet
// are defined and before flags are accessed by the program.
// The return value will be ErrHelp if -help or -h were set but not defined.
func (f *FlagSet) Parse(arguments []string) error {
    f.parsed = true
    f.args = arguments
    for {
        seen, err := f.parseOne()
        if seen {
            continue
        }
        if err == nil {
            break
        }
        switch f.errorHandling {
        case ContinueOnError:
            return err
        case ExitOnError:
            os.Exit(2)
        case PanicOnError:
            panic(err)
        }
    }
    return nil
}

// NewFlagSet returns a new, empty flag set with the specified name and
// error handling property.
func NewFlagSet(name string, errorHandling ErrorHandling) *FlagSet {
    f := &FlagSet{
        name:          name,
        errorHandling: errorHandling,
    }
    f.Usage = f.defaultUsage
    return f
}

// Inside package flag
// Parse parses the command-line flags from os.Args[1:].  Must be called
// after all flags are defined and before flags are accessed by the program.
func Parse() {
    // Ignore errors; CommandLine is set for ExitOnError.
    CommandLine.Parse(os.Args[1:])
}
```

## Ticker: What is Python equivalent?
I am thinking about the Python equivalent of Go's ticker utility. Here is one way to simulate the ticker in Python that uses Queue for message passing.

```python
# package main
# import "time"
# import "fmt"
# func main() {
#     ticker := time.NewTicker(time.Millisecond * 500)
#     go func() {
#         for t := range ticker.C {
#             fmt.Println("Tick at", t)
#         }
#     }()
#     time.Sleep(time.Millisecond * 1600)
#     ticker.Stop()
#     fmt.Println("Ticker stopped")
# }

class Ticker:
  def __init__(self, sleep_interval):
    self.running = True
    self.sleep_interval = sleep_interval
    self.q = Queue()

  def start(self):
    print('ticker starting!')

    def ticking():
      while self.running:
        self.q.put(datetime.now())
        time.sleep(self.sleep_interval)

    threading.Thread(target=ticking).start()

  def stop(self):
    self.running = False

  def restart(self):
    self.running = True


def print_ticker(ticker):
  print('entering print_ticker')
  ticker.start()
  while ticker.running:
    try:
      data = ticker.q.get(block=True, timeout=2)
    except queue.Empty as e:
      print(e)
    else:
      print('ticking {}'.format(data))


def main():
  ticker = Ticker(0.5)
  print("start print_ticker")
  threading.Thread(target=print_ticker, args=(ticker,)).start()
  time.sleep(5)
  ticker.stop()
  print('stop print_ticker')
```

## Thoughts
Go has the origin of 