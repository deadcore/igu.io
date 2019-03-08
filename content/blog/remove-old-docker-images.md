+++
author = "Jack Liddiard"
categories = ["Docker"]
tags = ["tutorial"]
date = "2017-09-26"
description = "One of the quirks I've found is Docker never cleans up it's self"
featured = "pic03.jpg"
featuredalt = "Pic 3"
featuredpath = "date"
linktitle = ""
title = "Remove Old Docker Images"
type = "post"
+++

Don't get me wrong I like [Docker](https://www.docker.com/), but my god it has some flaws.

One of them is it keeps taking down our continues integration server ([gocd](https://www.gocd.org/)) every few months because it hogs all the disk with unused and old images.

One of the quirks I've found is Docker never cleans up it's self

## Solution
```bash
docker images -q | xargs -r docker rmi
```

## What's Happening?
So what an earth is happening?

`docker images` will list you all the images in your docker instance:
* `-q` will only show numeric IDs

`xargs` is one of my favourite, it will build and execute a command lines from standard input
* `-r` will not run the command if the standard input does not contain any nonblanks.

`docker rmi` will remove one or more images
