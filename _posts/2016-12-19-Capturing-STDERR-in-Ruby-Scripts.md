---
layout: post
title: Capturing STDERR in Ruby Scripts
---

In this post you'll learn how to capture the `stderr` of failed shell commands run from within Ruby programs.

### Background

In Ruby there are several ways to "shell out," i.e. to run a shell command from within your program. Recently I found myself at a loss when I was working on a Ruby script that needed to capture anything written to `stderr` from shell commands. More specifically, I was running `knife ssh -m "<server1> <server2> ..." "sudo chef-client"` and wanted to grab error messages from any servers that failed and store them in a CSV file.

Two of the most common methods of shelling out, surrounding the command with backticks (``) and using `system()`, won't help inthis situation. If these are successful, backticks captures `stdout` and `system()` just returns `true`. When there's an error, both return `nil`:

```ruby
[28] pry(main)> output = `fake-command`
Errno::ENOENT: No such file or directory - fake-command
from (pry):27:in ``'
[29] pry(main)> output
=> nil
[30] pry(main)> output = system('fake-command')
=> nil
[31] pry(main)> output
=> nil
```

As you can see, backticks still print the error to the console, but remains out of reach as far as being usable in a script.

### Solution

Thankfully through some Googling I found out about the `Open3` class, which is part of Ruby's standard library. It's very easy to use: just require the class, then call `Open3.capture3('command')`, like this:

```ruby
[1] pry(main)> require 'open3'
=> true
[2] pry(main)> Open3.capture3('date')
=> ["Sat Dec 17 13:37:20 EST 2016\n", "", #<Process::Status: pid 98035 exit 0>]
``` 
As you can tell from the output, `::capture3` returns an array with three elements: 

1. The content of `stdout` produced by the command (emptry string if none).
2. The content of `stderr` produced by the command (emptry string if none). 
3. A `Process::Status` object that contains the process ID (pid) of the subshell that was created for running the command, and the exit status code.

To capture all 3 items as their own variables, just use multiple assignment:

```ruby
[3] pry(main)> stdin, stderr, status = Open3.capture3('date')
=> ["Sat Dec 17 13:40:41 EST 2016\n", "", #<Process::Status: pid 98370 exit 0>]
[4] pry(main)> stdin
=> "Sat Dec 17 13:40:41 EST 2016\n"
[5] pry(main)> stderr
=> ""
[6] pry(main)> status
=> #<Process::Status: pid 98370 exit 0>
```
