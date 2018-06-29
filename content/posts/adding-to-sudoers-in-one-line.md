---
title: "Adding to Sudoers in One Line"
date: 2017-09-13T07:11:00+00:00
draft: false
---

Recently I had to build a startup script which added a user to the sudoers file. I figured doing something like `echo "something here" >> /etc/sudoers` would be enough, but how wrong was I.

After some time looking around Stack Overflow and general Google I found this was the worst idea possible. Even the `README` in `sudoers.d` hints to only use a tool called `visudo`. Looking at the man page for `visudo` it is obvious why:

> visudo edits the sudoers file in a safe fashion, analogous to vipw(8). visudo locks the sudoers file against multiple simultaneous edits, provides basic sanity checks, and checks for parse errors.  If the sudoers file is currently being edited you will receive a message to try again later.

## Solution
<pre>echo "{{username}} ALL=(ALL) NOPASSWD: ALL" | (sudo su -c 'EDITOR="tee" visudo -f /etc/sudoers.d/{{username}}')</pre>

## What's Happening?
So what an earth is happening?

We set the output file `/etc/sudoers.d/{{username}}` with the `-f` flag.

`visudo` can accept an `EDITOR` reference variable which we set to `tee` allowing us to pipe into `visudo`.

Then finally `tee` is a utility which

> Read from standard input and write to standard output and files

### Reference
1) [https://xkcd.com/838/](https://xkcd.com/838/)
