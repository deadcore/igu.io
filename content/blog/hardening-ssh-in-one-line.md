+++
author = "Jack Liddiard"
categories = ["Linux", "SSH"]
tags = ["tutorial"]
date = "2018-08-13"
description = "A one liner to harden SSH"
featuredpath = "date"
linktitle = ""
title = "Hardening SSH in One Line"
type = "post"
+++

Many moons ago I used to manage several hundred servers for a company I worked for. Whilst it was fun it also proved very tiresome as menial tasks, such as changing the port SSH listens on, becomes very tiresome moving server to server.

A more efficient way was needed and ideally a one liner. Tho in saying that SSH is managed from a file under `/etc/ssh/sshd_config` which having to `vim` into was not an option. As well I wanted to leave the existing file intact encase any magic was added

## Solution
```bash
sed -i.bak -Ee '
s/^#?Port\s+[0-9]*$/Port 2222/g;
s/^#?PermitRootLogin.*$/PermitRootLogin no/g;
s/^#?PasswordAuthentication.*$/PasswordAuthentication no/g' \
/etc/ssh/sshd_config
```
## What's Happening?

First off for those who have never heard of or used `sed` it is a very powerful tool. `sed` is a stream editor for filtering and transforming text used to perform basic transformations on an input stream (a file or input from a  pipeline). This means you can essentially transform text per line once a set of conditions have been met. Observe:

* `-i.bak` This will set `sed` to edit the file inplace **and** makes backup since we supplied `.bak`. Removing the `.bak` will not create a backup
* `-E` This will enable extended regular expressions like `+`
* `-e` The expression we wish to use, I'll explain this below

Now `sed` lets break down one of the expressions. For this lets pick `s/^#?Port\s+[0-9]*$/Port 2222/g;`:

* `s` - This stands for substitution and will replace matches defined in the first argument for the second
* `/` - Denotes the stand of the first parameter of the find.
* `^#?Port\s+[0-9]*$` - A regular expression we use to match a number of ways the `Port` parameter can be represented. These include active(`Port 1`) and commented out(`#Port 12`)
* `/` - Denotes the start of the second parameter to be substituted in
* `Port 2222` - Our port we wish to bind on
* `/` - End of the second parameter
* `g` - Sets the substitution to affect globally encase we have multiple instances of `Port` defined.

Finally we specify the file we wish to edit which is `/etc/ssh/sshd_config`
