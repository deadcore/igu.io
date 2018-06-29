---
title: "Binding Ghost to a Unix Domain Socket"
date: 2017-09-29T13:32:37+00:00
draft: true
---

So while I was setting up [igu](https://igu.io), like many people I needed to setup which port [Ghost](https://ghost.org/) would listen on. While digging through the documentation I found that Ghost could bind to a Unix Domain Socket, so lets give that a go instead

## Solution
`config.production.json`
<pre>
"server": {
    "socket": {
	    "path": "/tmp/igu-io.sock",
	    "permissions": "0666"
	}
}
</pre>

`/etc/nginx/sites-available/igu.io`
<pre>
server {
    listen 80;
    server_name igu.io;
    location / {
        proxy_pass http://igu;
  }
}

upstream igu {
    server unix:/tmp/igu-io.sock fail_timeout=0;
}
</pre>

## What's Happening?
Lets break this down
### Unix Domain Socket
First off we have to ask the question what is a Unix domain socket? Well a Unix domain socket is a data communications endpoint for exchanging data between processes executing on the same host. Effectively they support transmission of a reliable stream of bytes using `SOCK_STREAM` instead of `TCP`

The first thing to note is that even though we can see the socket on the file system by running `ls` against it

<pre>
root@ghost:~$ ls -lash /tmp/igu-io.sock
0 srw-rw-rw- 1 ghost ghost 0 Sep 28 07:11 /tmp/igu.io.socket
</pre>

Nothing is written to disk. Processes reference Unix domain sockets as file system inodes, so the two processes can communicate by opening the same socket.


Calling `file` on our socket furthers the existance of the socket
<pre>
jackliddiard@ghost-001:~$ file /tmp/igu-io.sock
/tmp/igu.io.socket: socket
</pre>

### Ghost
The ghost config is super simple:
* **path**: Just where the socket description file will be located
* **permissions**: We have to set this so everyone can read this because in my case Nginx runs under a separate user.

### Nginx
The Nginx config again is super simple:

* **proxy_pass**: We set this to the name of the `upstream` container
* **upstream**: The `ngx_http_upstream_module` module is used to define groups of servers that can be referenced by the proxy_pass (among other things). It gives us a named reference to the socket
* **server unix:/tmp/igu-io.sock fail_timeout=0;**
    * **fail_timeout=0**: sets the time during which the specified number of unsuccessful attempts to communicate with the server should happen to consider the server unavailable. By default, the parameter is set to 10 seconds so we just zero it

## Why Bother?
Well it's quite simple, no more figuring out which ports are free if you are running multiple instance of Ghost on the same machine (for different websites). You also remove the TCP overhead for communicating between Nginx and Ghost
