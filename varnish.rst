Varnish
=======

Learn Varnish on Arch Linux, Nginx and Django

What is it?
-----------

Varnish Cache is a web application accelerator also known as a caching HTTP reverse proxy. You install it in front of any server that speaks HTTP and configure it to cache the contents.

https://www.varnish-cache.org/

Setup
-----

Install

.. code:: sh
    
    pacman -S varnish
    cp 'usr/lib/systemd/system/varnish.service' '/etc/systemd/system/varnish.service'
    cp /etc/varnish/default.vcl /etc/varnish/default.vcl.sample
    systemctl enable varnish
    systemctl start varnish

Generally speaking, you should restart nginx then varnish in this order when making changes.

Set port to 8080 in nginx config

.. code:: sh

    server {
        listen 8080;
        server_name www.awesomewebsite.com;
    }

Set the backend port to 8080 in VCL config. It's 8080 by default

.. code:: sh
    backend default {
        .host = "127.0.0.1";
        .port = "8080";
    }

Architecture
------------

All incoming requests will hit Varnish first. If it is unable to serve up a cached version, it will use the backend specified (you can define multiple backends). By default, we use nginx so nginx gets hit which in turn calls uwsgi. 

Varnish Gotcha
--------------

1. By default (and design) Varnish does not cache content with cookies in it. So you have to exclude/remove cookies in the VCL config file to bypass this.

2. Varnish also does not deal with HTTPS, because encryption makes it impossible for Varnish to distinguish what the objects are that it is passing back and forth from the user to the backend. There are workarounds (like installing Pound) in front of Varnish and terminating SSL/TLS there.

https://www.varnish-cache.org/trac/wiki/VCLExampleCacheCookies


Django Gotcha
-------------

http://stackoverflow.com/questions/12034242/is-varnish-compatible-with-django-csrf-protection

You also have to remove the sssionid and csrftoken set in the cookies. The VCL file below shows you how


VCL file
--------

The VCL file is the place where you control the rules.

Some basic settings

.. code:: sh

    sub vcl_recv {

      # Use anonymous, cached pages if all backends are down.
      if (!req.backend.healthy) {
        unset req.http.Cookie;
      }

      # Always cache the following file types for all users. This list of extensions
      # appears twice, once here and again in vcl_fetch so make sure you edit both
      # and keep them equal.
      if (req.url ~ "(?i)\.(pdf|asc|dat|txt|doc|xls|ppt|tgz|csv|png|gif|jpeg|jpg|ico|swf|css|js)(\?.*)?$") {
        unset req.http.Cookie;
      }

      # unless sessionid/csrftoken is in the request, don't pass ANY cookies (referral_source, utm, etc)  
      if (req.request == "GET" && (req.url ~ "^/static" || (req.http.cookie !~ "sessionid" && req.http.cookie !~ "csrftoken"))) {  
        remove req.http.Cookie;  
      } 


        if (req.http.Accept-Encoding) {
        if (req.url ~ "\.(jpg|png|gif|gz|tgz|bz2|tbz|mp3|ogg)$") {
            # No point in compressing these
            remove req.http.Accept-Encoding;
        } elsif (req.http.Accept-Encoding ~ "gzip") {
            set req.http.Accept-Encoding = "gzip";
        } elsif (req.http.Accept-Encoding ~ "deflate" && req.http.user-agent !~ "MSIE") {
            set req.http.Accept-Encoding = "deflate";
        } else {
            # unkown algorithm
            remove req.http.Accept-Encoding;
        }
        }

    }

    # Set a header to track a cache HIT/MISS.
    sub vcl_deliver {
      if (obj.hits > 0) {
        set resp.http.X-Varnish-Cache = "HIT";
      }
      else {
        set resp.http.X-Varnish-Cache = "MISS";
      }
    }

    # Code determining what to do when serving items from the Apache servers.
    # beresp == Back-end response from the web server.
    sub vcl_fetch {
      # We need this to cache 404s, 301s, 500s. Otherwise, depending on backend but 
      # definitely in Drupal's case these responses are not cacheable by default.
      if (beresp.status == 404 || beresp.status == 301 || beresp.status == 500) {
        set beresp.ttl = 10m;
      }
     
      # Don't allow static files to set cookies. 
      # (?i) denotes case insensitive in PCRE (perl compatible regular expressions).
      # This list of extensions appears twice, once here and again in vcl_recv so 
      # make sure you edit both and keep them equal.
      if (req.url ~ "(?i)\.(pdf|asc|dat|txt|doc|xls|ppt|tgz|csv|png|gif|jpeg|jpg|ico|swf|css|js)(\?.*)?$") {
        unset beresp.http.set-cookie;
      }

     # static files always cached  
      if (req.url ~ "^/static") {  
           unset beresp.http.set-cookie;  
           return (deliver);  
      } 

      # pass through for anything with a session/csrftoken set  
      if (beresp.http.set-cookie ~ "sessionid" || beresp.http.set-cookie ~ "csrftoken") {  
        return (hit_for_pass);  
      } else {  
        return (deliver);  
      }
     
      # Allow items to be stale if needed.
      set beresp.grace = 6h;
    }

    # In the event of an error, show friendlier messages.
    sub vcl_error {
      # Redirect to some other URL in the case of a homepage failure.
      #if (req.url ~ "^/?$") {
      #  set obj.status = 302;
      #  set obj.http.Location = "http://backup.example.com/";
      #}
     
      # Otherwise redirect to the homepage, which will likely be in the cache.
      set obj.http.Content-Type = "text/html; charset=utf-8";
      synthetic {"
    <html>
    <head>
      <title>Page Unavailable</title>
      <style>
        body { background: #303030; text-align: center; color: white; }
        #page { border: 1px solid #CCC; width: 500px; margin: 100px auto 0; padding: 30px; background: #323232; }
        a, a:link, a:visited { color: #CCC; }
        .error { color: #222; }
      </style>
    </head>
    <body onload="setTimeout(function() { window.location = '/' }, 5000)">
      <div id="page">
        <h1 class="title">Page Unavailable</h1>
        <p>The page you requested is temporarily unavailable.</p>
        <p>We're redirecting you to the <a href="/">homepage</a> in 5 seconds.</p>
        <div class="error">(Error "} + obj.status + " " + obj.response + {")</div>
      </div>
    </body>
    </html>
    "};
      return (deliver);
    }

The service file in /etc/systemd/system/varnish.service be defaults listens on port 80. To change the allocated ram, change 64M to whatever you wish.

.. code:: sh


    [Unit]
    Description=Web Application Accelerator
    After=network.target

    [Service]
    ExecStart=/usr/bin/varnishd -a 0.0.0.0:80 -f /etc/varnish/default.vcl -T localhost:6082 -s malloc,64M -u nobody -g nobody -F
    ExecReload=/usr/bin/varnish-vcl-reload

    [Install]
    WantedBy=multi-user.target

Commands
--------

To check config file. If everything is ok, there will be a printout

.. code:: sh

    sudo varnishd -C -f /etc/varnish/default.vcl

To ban a particular page (clear the cache for the page)

.. code:: sh

    varnishadm -T :6082 "ban.url /create"

Is Varnish Working?
-------------------

.. code:: sh

    [nai:~]$ curl --head http://dev.tripevent.co/
    HTTP/1.1 200 OK
    Server: nginx/1.4.1
    Content-Type: text/html; charset=utf-8
    Vary: Accept-Encoding, Cookie
    Date: Sun, 23 Jun 2013 03:56:49 GMT
    X-Varnish: 133995355 133995354
    Age: 2
    Via: 1.1 varnish
    Connection: keep-alive
    X-Varnish-Cache: HIT

or simply 

http://www.isvarnishworking.com/

Benchmarks (Exciting!)
----------------------

.. code:: sh

    (thack2012)[nai:~/Work/thack2012]$ siege -c50 -r10 -v http://dev.tripevent.co/map/pycon-sg
    With Varnish

    Transactions:                   998 hits
    Availability:                 99.90 %
    Elapsed time:                170.38 secs
    Data transferred:             2.01 MB
    Response time:                  7.50 secs
    Transaction rate:             5.86 trans/sec
    Throughput:                  0.01 MB/sec
    Concurrency:                 43.91
    Successful transactions:         998
    Failed transactions:                1
    Longest transaction:            15.43
    Shortest transaction:             0.16

    Without Varnish

    Transactions:                  1000 hits
    Availability:                100.00 %
    Elapsed time:                159.36 secs
    Data transferred:             2.02 MB
    Response time:                  6.93 secs
    Transaction rate:             6.28 trans/sec
    Throughput:                  0.01 MB/sec
    Concurrency:                 43.52
    Successful transactions:        1000
    Failed transactions:                0
    Longest transaction:            15.69
    Shortest transaction:             0.27

    (thack2012)[nai:~/Work/thack2012]$ siege -c50 -r10 -v http://dev.tripevent.co/

    With Varnish

    Transactions:                   500 hits
    Availability:                100.00 %
    Elapsed time:                  9.76 secs
    Data transferred:             0.54 MB
    Response time:                  0.20 secs
    Transaction rate:            51.23 trans/sec
    Throughput:                  0.06 MB/sec
    Concurrency:                 10.25
    Successful transactions:         500
    Failed transactions:                0
    Longest transaction:             1.45
    Shortest transaction:             0.12

    Without Varnish

    Transactions:                   500 hits
    Availability:                100.00 %
    Elapsed time:                 10.15 secs
    Data transferred:             0.54 MB
    Response time:                  0.24 secs
    Transaction rate:            49.26 trans/sec
    Throughput:                  0.05 MB/sec
    Concurrency:                 11.76
    Successful transactions:         500
    Failed transactions:                0
    Longest transaction:             1.73
    Shortest transaction:             0.14

What you don't see here is the CPU utilization. Nginx uses a little bit more but not significantly.

Alternatives
------------

1. http://www.squid-cache.org/

Useful Links
------------

1. https://www.varnish-cache.org/
2. http://chase-seibert.github.io/blog/2011/09/23/varnish-caching-for-unauthenticated-django-views.html
3. http://blog.bigdinosaur.org/adventures-in-varnish/
4. http://www.nedproductions.biz/wiki/a-perfected-varnish-reverse-caching-proxy-vcl-script
