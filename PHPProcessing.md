# PHP Processing

In the previous documents, NGINX was serving static assets of various types leaving the browser to handle the rendering based on the mime type. Most servers serve dynamic content that has been generated by a server-side language. Here, **php-fpm** will be used as a stand-alone service in which NGINX will pass a request for processing. NGINX will then take that response and return it to the client, i.e., act as a proxy server to the client :).

### Install the `php-fpm` Service
After install of php-fpm, you will be prompted with the version number, php7.1, as well as the fact that php-fpm was automatically installed as a systemd service.

```console
# apt-get install php-fpm

...

  The following NEW packages will be installed:
    php-common php-fpm php7.1-cli php7.1-common php7.1-fpm php7.1-json php7.1-opcache php7.1-readline

...

  Created symlink /etc/systemd/system/multi-user.target.wants/php7.1-fpm.service → /lib/systemd/system/php7.1-fpm.service.

...
```

Confirm the PHP systemd service exists with `grep` and then check the status to see one master process as well as two worker processes.
```console
# systemctl list-units | grep php

...

  php7.1-fpm.service                                                                                    loaded active running   The PHP 7.1 FastCGI Process Manager                                          
  phpsessionclean.timer                                                                                 loaded active waiting   Clean PHP session files every 30 mins
```

```console
# systemctl status php7.1-fpm

...

  ● php7.1-fpm.service - The PHP 7.1 FastCGI Process Manager
   Loaded: loaded (/lib/systemd/system/php7.1-fpm.service; enabled; vendor preset: enabled)
   Active: active (running) since Sun 2018-06-10 18:19:01 PDT; 10min ago
     Docs: man:php-fpm7.1(8)
  Main PID: 8755 (php-fpm7.1)
   Status: "Processes active: 0, idle: 2, Requests: 0, slow: 0, Traffic: 0req/sec"
    Tasks: 3 (limit: 4915)
   Memory: 30.6M
      CPU: 142ms
   CGroup: /system.slice/php7.1-fpm.service
           ├─8755 php-fpm: master process (/etc/php/7.1/fpm/php-fpm.conf)
           ├─8765 php-fpm: pool www
           └─8767 php-fpm: pool www

...
```
### Configure the Passing of PHP files to the NGINX Service
Consider the `nginx.conf` file below. A few things to note is the regular expression match, `~\.php$`, and the [index directive](http://nginx.org/en/docs/http/ngx_http_index_module.html). Most importantly for this configuration is understand the use of the [fastCGI](https://en.wikipedia.org/wiki/FastCGI) protocol.

```nginx
events {}

http {

  include mime.types;

  server {

    listen 80;
    server_name 167.99.93.26;
    root /sites/demo;

    index index.php index.html;

    location / {
      try_files $uri $uri/ =404;
    }

    location ~\.php$ {
      include fastcgi.conf;
      fastcgi_pass unix:/run/php/php7.1-fpm.sock;
    }

  }
}
```
`index` is a standard NGINX directive that tells NGINX where to go if the request points to a directory. Simply put, if  `index.php` does not exist in the directory, then `index.html` will be served as the default index instead. The first location block (prefix match) handles all possible requests for any static assets with a `try_files` directive. The last location block makes use of the fastCGI protocol. There are plenty of [fastCGI resources](https://www.digitalocean.com/community/tutorials/understanding-and-implementing-fastcgi-proxying-in-nginx) to get up-to-speed with this technology, but in short, it is how NGINX will communicate with php-fpm.  


### FastCGI and Unix Sockets
The second location block will include the `fastcgi.conf` file similar to how the http parent directive has included the `mime.types` file. Both files live in the same location hence their relative paths as the only argument for `include`. The fastCGI file was configured at the time of NGINX's installation and can be found as follows.

```console
# ls -l /etc/nginx/fastcgi.conf
  -rw-r--r-- 1 root root 1077 Jun  6 21:44 /etc/nginx//fastcgi.conf
```

Now consider the `fastcgi_pass` directive. It's job here is to pass the fastCGI file to a Unix socket. If you are unfamiliar with Unix sockets, think of it as an HTTP port where servers can listen for binary data. You can find the socket location as follows.

```console
# find / -name *fpm.sock
/run/php/php7.1-fpm.sock
```

### Create PHP File
Before restarting NGINX and testing out the configuration, create a very small PHP file to place in our application's root directory with one line of code. The only thing to keep in mind is the native PHP function that returns all of php's configuration information.

```console
echo '<?php phpinfo(); ?>' > /sites/nginx-demo/info.PHP
```

First, restart NGINX and point the browser to `http//<ip_address>` to render the original html file. Second, point the browser to `http://<ip_address>/info.php` to match the regex location block. Notice the 502 bad gateway. What do?

### User Permissions With Sockets

Check the error logs for a clue as to what went wrong as follows.

```console
# tail -n 1 /var/log/nginx/error.log
2018/06/10 20:18:38 [crit] 9202#0: *1 connect() to unix:/run/php/php7.1-fpm.sock failed (13: Permission denied) while connecting to upstream, client: 10.0.1.16, server: 192.168.86.31, request: "GET /info.php HTTP/1.1", upstream: "fastcgi://unix:/run/php/php7.1-fpm.sock:", host: "10.0.1.6"
```
The error gives away the fact that permissions are not configured correctly for the Unix socket. This could be handled by changing file permissions, but there is a more secure way.

Check the user the NGINX process is running as follows.

```console
# ps aux | grep nginx
root      9201  0.0  0.0  32144   784 ?        Ss   20:18   0:00 nginx: master process /usr/bin/nginx
nobody    9202  0.0  0.0  32304  2708 ?        S    20:18   0:00 nginx: worker process
root      9210  0.0  0.0  13068  1068 pts/0    S+   20:22   0:00 grep --color=auto nginx
```

The problem is `nobody` is running as the worker process. Compare this to the PHP fps process as follows.

```console
# ps aux | grep php
root      8755  0.0  0.4 282028 34476 ?        Ss   18:19   0:00 php-fpm: master process (/etc/php/7.1/fpm/php-fpm.conf)
www-data  8765  0.0  0.0 282028  6524 ?        S    18:19   0:00 php-fpm: pool www
www-data  8767  0.0  0.0 282028  6524 ?        S    18:19   0:00 php-fpm: pool www
root      9212  0.0  0.0  13068  1064 pts/0    S+   20:24   0:00 grep --color=auto php
```

Mystery solved. The `nobody` user does not have permission to access the php socket. The solution is to have NGINX run as the same user as the PHP process. Note the addition of `user` in the configuration file below.

```nginx
user www-data;

events {}

http {

...

```

### Conclusion
This document exemplifies the basic nature of NGINX – a proxy server. The browser sent a request to NGINX, which then communicated that to the PHP fpm service, the request was processed, sent back to NGINX, and then sent back to the client.