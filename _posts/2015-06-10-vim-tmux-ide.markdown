---
layout: post
title: "Vim + Tmux = IDE"
date: 2015-06-10 20:00:00
categories: ide vim tmux
---

What defines a good IDE?  What setup will allow you to pair with coworkers?  How can one collaborate with others in a sane way?

When I started my career I was motivated to learn the ins and outs of system administration and systems programming.  So naturally I
made the effort to learn vim.  It was so cool to be able to open and edit files directly on the server without the need for a graphical
display.  Sure the key bindings were hard to learn when first starting out, but it was worth it to be able to walk up to any BSD/linux
server and have your text editor there.

While at college my professors were a mixed bag of advice regarding development environments.  A few "hard-core" ones promoted emacs.
Some promoted eclipse, others promoted Visual Studio, and one systems programming professor promoted vim.  As a good student I got in 
line with the majority (as who really wants to be different??) and started doing all my development in a big heavy IDE.

Years of subtle career shifts toward web application development happened, and I picked up plugin after plugin and used my mouse to select 
my auto-completions.  I still used vim on the rare occasion I needed to deploy something or manipulate files on a server, but for the most
part I used a graphical IDE.

A few years back I changed jobs, and started working for a West Coast based company from the East Coast.  I remember thinking to myself, how
am I going to be able to work closely with this new team when I can't just walk over and talk to them in person, or sit by their desk and pair 
together.  Turns out everyone uses vim and gnu-screen/tmux all day for development.

I brushed off the dust from my keyboard and started pairing with my teammates...

Years later and my preferred collaborative development environment is vim and tmux paired together on a remote server.  I feel like this solution
is superior to the following other schemes out there to do real remote collaborative development:

* Graphical Screen Broadcasting (_hangouts, skype, etc_)
  * Doesn't allow both participants to participate interactively
  * Resolution sucks, and usually the fine print font appearances are terrible
  * Very, Very difficult to multitask, i.e. There is no way for coworkers to divide the work if not pairing, and still be in the same environment
* Graphical Screen Sharing (_vnc, remote desktop_)
  * Resolution issues, bandwidth issues become very annoying
  * Very difficult to multitask on the same machine
* Using a web based collaborative environment (_atom pair, xpairtise_)
  * Hard to follow along/lead with the potential to have many cursors moving around randomly
  * Centralized third party controlled... scary...

I understand that people want to work in a comfortable environment for their day to day work.  I don't think people should be mandated to fit within
a certain box in order to collaborate, but then again it is hard to collaborate if you don't have a base tool-set that works for everyone on the team.

Here is my [.vimrc][vimrcgist] and my [.tmux.conf][tmuxgist]

Hope this was helpful to anyone.

[vimrcgist] https://gist.github.com/husobee/8d6ae09f79f4e584c8a2
[tmuxgist] https://gist.github.com/husobee/c20d17775db97045b123
