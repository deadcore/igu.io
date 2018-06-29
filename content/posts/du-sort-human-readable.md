---
title: "du - Sort Human Readable"
date: 2017-09-26T07:07:00+00:00
draft: true
---

Quite a lot of the time we have to find the size of a number of directories to troubleshoot which one are taking up all the space. However how do you find the largest of directories if you have a fair few, sometimes seeing the woods through the trees can be very hard

## Solution
<pre>du -hd1 | sort --human-numeric-sort</pre>

## What's Happening?
So what an earth is happening?

`du` will get you the estimated file space usage:
* `-d1` will set the max depth to 1 (or the current directories space usage)
* `-h` will print the sizes in human readable format (e.g. 1K, 234M, 2G)

`sort` will sort lines of text from stdin:
* `--human-numeric-sort` will compare and sort human readable numbers
