---
layout: post
title: "Linux Tutorials"
date: 2016-01-17 08:00:00
categories: linux tutorials
---

## Linux Tutorials

I am asked often "how to learn linux" and more often than not I just find myself
saying "try it out" which is horrible.  How can you "try it out" if you don't
even have a baseline of information about what is happening?  The person would
be forced to read many volumes of words just to get a grasp on what they want
to learn.

This is a living document of my attempt to explain linux for new linux users.  I
am trying explain as if the user has zero background in unix/linux variants but 
has a willingness to learn.  If you find discrepancies in what I have written 
please by all means contact me and I will address the mistakes.

### The Command Line Interface

When people think about Linux/Unix they immediately are put off by the seemingly
complicated command line interface.  I mean, since the 1980's we have become 
accustom to learning computers on a graphical user interface, where we can 
point a mouse and click on things.  It isn't that hard to understand the why this
window of black text is so intimidating.  

But, when you learn that basically computer programs have inputs, and outputs,
and the shell is there to facilitate the inputs and outputs of programs to the 
human interacting with the interface, it all makes a lot of sense.

#### The Shell

When you are at a command line interface, you are actually running a program, called 
a shell.  A shell is the primary interface between you as a user, and the computer.
You provide the shell text as input, specifying programs to run and data to give to 
programs, and the programs give their output to the shell after running, which in 
turn the shell gives back to the user.  Very simply put the shell is an interface
by which you interact with the computer.

A very popular shell is [Bash][bash]. Shells, being interpreters ( the shell 
interprets what the user wants to do, and then performs that interpretation)
allow for various control-flow capabilities as well.  Such control-flow 
capabilities include loops and conditionals.

#### Programs

So, we know now that we can run programs on a computer through the shell.  We 
should talk a little bit about programs.  Programs are, at a high level, a grouping
of instructions for a CPU.  The format of the instructions depends highly on the 
type of processor you have, but lets not get bogged down with that.

Programs being composed of instructions and data can be "executed" or "run" by the
shell by the user telling the shell the location of the file which contains said 
instructions (the program file location)

For example, if I wanted to run a program called `my-useful-program` in the shell
all I need to do is the following:

{% highlight bash %}
/path/to/my-useful-program # where /path/to/ is the directory location that my-useful-program is located
{% endhighlight %}

Programs can have the following _needs_:
1. Programs potentially need [stdin or Standard Input][stdin_man]
2. Programs potentially need command options
3. Programs potentially need command arguments

In linux/unix variants input to programs is typically provided via [stdin][stdin]
which means the program is handed a "file descriptor" by which the application
can read (buffered) input to a program.  This is a bit different than a graphical 
user interface in that inputs to programs can be outputs from other programs, which
could form a chain, or pipeline of processing.  For example:

{% highlight bash %}
cat file.txt | grep "something" # we "cat" a file and send that output to "grep"
{% endhighlight %}

Program command options typically come in the form of `-flag` and you can think 
of this as almost configuration for the command.  This allows you to fine tune
the program configuration to give you what you want, for example:

{% highlight bash %}
ls -alh # we are telling the ls command we want the a, l and h options configured
{% endhighlight %}

Programs can also need arguments which are text tokens that the program uses to 
do something.  Typically this is used to specify files to perform operations on
but could be for anything really, for example:

{% highlight bash %}
ls -alh *.txt # we are passing *.txt as a command argument to the ls command
{% endhighlight %}

With that basic understanding of what programs are, and what programs may or 
may not need, lets look at some very commonly used programs

#### Various Useful Programs

Here is a listing of useful programs that typically come standard on linux and unix
variants, with a short description, and a link to the `man` or manual pages:

0. `man` - Man is a manual for commands
  * Example: 
{% highlight bash %}
    user@localhost ~]$ man ls
LS(1)                                 User Commands                                LS(1)

NAME
       ls - list directory contents

       SYNOPSIS
              ls [OPTION]... [FILE]...
...
{% endhighlight %}
  * [man man][man_man]

1. `echo` - Echo prints it's arguments to the output stream
  * Example: 
{% highlight bash %}
    user@localhost ~]$ echo "hello world"
    hello world
{% endhighlight %}
  * [man echo][echo_man]
2. `cat` - Output the contents of a file
  * Example: 
{% highlight bash %}
    user@localhost ~]$ cat file.txt # file.txt contains "hello world"
    hello world
{% endhighlight %}
  * [man cat][cat_man]
3. `grep` - Search Input for an expression
  * Example: 
{% highlight bash %}
    user@localhost ~]$ echo "hi there" | grep "hi"
    hi there
{% endhighlight %}
  * [man grep][grep_man]
4. `ls` - Print out the directories and files contained within the current working directory
  * Example: 
{% highlight bash %}
    user@localhost ~]$ ls
    file.txt
{% endhighlight %}
  * [man ls][ls_man]

## Definitions:

1. [POSIX][posix] - Portable Operating System Interface.  A group of standards 
for maintaining compatibility between operating systems (unix variants)
2. [bash][bash] - Bourne-Again SHell, a shell program that takes commands and 
executes said commands, an attempt at a [POSIX][posix] compatible shell.
3. [stdin][stdin_man] - Standard Input is a "file handle" that allows a program to take input
3. [stdout][stdout_man] - Standard Input is a "file handle" that allows a program to write normal output to
3. [stderr][stderr_man] - Standard Error is a "file handle" that allows a program to write errors to

[posix]: https://en.wikipedia.org/wiki/POSIX
[bash]: https://en.wikipedia.org/wiki/Bash_%28Unix_shell%29
[echo_man]: http://linux.die.net/man/1/echo
[stdin_man]: http://linux.die.net/man/3/stdin
[stdout_man]: http://linux.die.net/man/3/stdout
[stderr_man]: http://linux.die.net/man/3/stderr
[cat_man]: http://linuxcommand.org/man_pages/cat1.html
[grep_man]: http://linuxcommand.org/man_pages/grep1.html
[ls_man]: http://linux.die.net/man/1/ls
[man_man]: http://linux.die.net/man/1/man
