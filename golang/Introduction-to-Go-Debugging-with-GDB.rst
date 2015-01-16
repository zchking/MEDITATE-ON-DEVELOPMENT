Introduction to Go Debugging with GDB
=====================================

*Jan 16th,2016*

From https://lincolnloop.com/blog/introduction-go-debugging-gdb/

 
I spent the vast majority of my time in the last 4 years writing, reading and debugging Python or JavaScript code. The process of learning Go was like a beautiful hike in the mountains with a small rock in my shoe. A lot of things impressed me, but using println to debug my code was travelling too far into the past. In Python we have pdb/ipdb to debug the code while running it, JavaScript offers similar tools. Over the years this pattern became a very important part of my development workflow.

Today I realized that Go has builtin support for the Gnu debugger (aka GDB).

For the sake of this article we are going to use the simple program below

::

    package main

    import (
        "fmt"
        "time"
    )

    func counting(c chan<- int) {
        for i := 0; i < 10; i++ {
            time.Sleep(2 * time.Second)
            c <- i
        }
        close(c)
    }

    func main() {
        msg := "Starting main"
        fmt.Println(msg)
        bus := make(chan int)
        msg = "starting a gofunc"
        go counting(bus)
        for count := range bus {
            fmt.Println("count:", count)
        }
    }

To use GDB you need to compile your program with the options -gcflags ”-N -l”. These options prevent the compiler from using inline functions and variables.

::

    go build -gcflags "-N -l" gdbsandbox.go

Here is an example of an interactive debugging GDB session:

::

    yml@simba$  gdb gdbsandbox 
    GNU gdb (Ubuntu/Linaro 7.4-2012.04-0ubuntu2) 7.4-2012.04
    Copyright (C) 2012 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>

First we run our program:

::

    (gdb) run

    Starting program: /home/yml/Developments/go/src/gdbsandbox/gdbsandbox 

    Starting main

    count: 0

    count: 1

    count: 2

    [...]

    count: 9

    [Inferior 1 (process 13507) exited normally]

Now that we know how to run our program we probably want to set a breakpoint:

::

    (gdb) help break 

    Set breakpoint at specified line or function.
    break [LOCATION] [thread THREADNUM] [if CONDITION]
    LOCATION may be a line number, function name, or "*" and an address.

    [...]

    (gdb) break 22

    Breakpoint 1 at 0x400d7a: file /home/yml/Developments/go/src/gdbsandbox/gdbsandbox.go, line 22.

    (gdb) run

    Starting program: /home/yml/Developments/go/src/gdbsandbox/gdbsandbox 

    Starting main
    [New LWP 13672]
    [Switching to LWP 13672]

    Breakpoint 1, main.main () at /home/yml/Developments/go/src/gdbsandbox/gdbsandbox.go:22

    22              for count := range bus {

    (gdb) 

Once GDB stops at your breakpoint you view the context:

::

    (gdb) help list

    List specified function or line.
    With no argument, lists ten more lines after or around previous listing.
    "list -" lists the ten lines before a previous ten-line listing.
    [...]

    (gdb) list
    msg := "Starting main"

    fmt.Println(msg)

    bus := make(chan int)

    msg = "starting a gofunc"

    go counting(bus)

    for count := range bus {

           fmt.Println("count:", count)

     }

    }

You can also inspect the variables:

::

    (gdb) help print
    Print value of expression EXP.
    Variables accessible are those of the lexical environment of the selected
    stack frame, plus all those whose scope is global or an entire file.
    [...]
    (gdb) print msg
    $1 = "starting a gofunc"

Earlier in the code we started a goroutine. I want to introspect this part of my program next time we execute the line 10.

::

    (gdb) break 10
    Breakpoint 3 at 0x400c28: file /home/yml/Developments/go/src/gdbsandbox/gdbsandbox.go, line 10.
    (gdb) help continue
    Continue program being debugged, after signal or breakpoint.
    If proceeding from breakpoint, a number N may be used as an argument,
    which means to set the ignore count of that breakpoint to N - 1 (so that
    the breakpoint won't break until the Nth time it is reached).
    [...]
    (gdb) continue
    Continuing.

The last thing we are going to demo today is how to change the value of a variable at runtime.

::

    Breakpoint 3, main.counting (c=0xf840001a50) at /home/yml/Developments/go/src/gdbsandbox/gdbsandbox.go:10
    10                      time.Sleep(2 * time.Second)
    (gdb) help whatis
    Print data type of expression EXP.
    Only one level of typedefs is unrolled.  See also "ptype".
    (gdb) whatis count
    type = int
    (gdb) print count
    $3 = 1
    (gdb) set variable count=3
    (gdb) print count
    $4 = 3
    (gdb) c
    Continuing.
    count: 3

We only covered the following commands:

    - list
    - next
    - print
    - continue
    - break <line number=“number”>
    - whatis
    - set variable <var>=<value>

and this barely scratches the surface of what you can do with GDB, here are some links if you want to learn more:

    http://sourceware.org/gdb/current/onlinedocs/gdb/
    http://golang.org/doc/gdb


