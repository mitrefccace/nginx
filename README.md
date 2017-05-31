![](images/adsmall.png)

# NGINX

NGINX is an open-source web-server, reverse-proxy, load-balancer and HTTP cache.
For the ACE Direct project, we are only using NGINX as a reverse proxy.

A reverse-proxy allow us to hide internally-used port numbers and script
names on the public-facing interface.

Full documentation and screenshots are here: [NGINX](https://www.nginx.com).

### NGINX Installation
1. Add the CentOS 7 EPEL repository

    ```
    sudo yum install epel-release
    ```

1. Install NGINX

    ```
    sudo yum install nginx
    ```

1. Configure and Start the NGINX Service

    ```
    sudo systemctl start nginx
    ```

1. Verify that the service is running (you should see two PIDs)

    ```
    pidof nginx
    ```

1. To enable NGINX to start on boot

    ```
    sudo systemctl enable nginx
    ```

At this point the service can be started or stopped with the following commands:
```
sudo service nginx start
sudo service nginx stop
```

### NGINX Configuration
1. Update the NGINX configuration (/etc/nginx/nginx.conf) as shown below
1. Verify that they key and cert are properly configured
1. Restart the service
1. Verify that the service is running (see pidof command above)


```
# For more information on configuration, see:
#   * Official English Documentation: http://nginx.org/en/docs/
#   * Official Russian Documentation: http://nginx.org/ru/docs/

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log;
pid /run/nginx.pid;

# Load dynamic modules. See /usr/share/nginx/README.dynamic.
include /usr/share/nginx/modules/*.conf;

events {
    worker_connections 1024;
}

http {
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';

    access_log  /var/log/nginx/access.log  main;

    sendfile            on;
    tcp_nopush          on;
    tcp_nodelay         on;
    keepalive_timeout   65;
    types_hash_max_size 2048;

    include             /etc/nginx/mime.types;
    default_type        application/octet-stream;

    # Load modular configuration files from the /etc/nginx/conf.d directory.
    # See http://nginx.org/en/docs/ngx_core_module.html#include
    # for more information.
    include /etc/nginx/conf.d/*.conf;

    server {
        listen       80 default_server;
        listen       [::]:80 default_server;
        listen       443 ssl;
        ssl on;

        # Specify the location of the cert and keyZZ
        ssl_certificate /etc/ssl/cert.pem;
        ssl_certificate_key /etc/ssl/key.pem;
        server_name  _/;
        root         /usr/share/nginx/html;

        # Load configuration files for the default server block.
        include /etc/nginx/default.d/*.conf;

        location / {
        }

        error_page 404 /404.html;
            location = /40x.html {
        }

        error_page 500 502 503 504 /50x.html;
            location = /50x.html {
        }

        location /ACEDirect/ {
		        proxy_pass https://localhost:8005/;
		        proxy_http_version 1.1;
		        proxy_set_header Upgrade $http_upgrade;
		        proxy_set_header Connection 'upgrade';
		        proxy_set_header Host $host;
		        proxy_cache_bypass $http_upgrade;
		        sub_filter /node /;
		        proxy_redirect https://localhost:8005/ACEDirect/ /ACEDirect/;
       }

       location /ManagementPortal/ {
		        proxy_pass https://localhost:8081/;
		        proxy_http_version 1.1;
		        proxy_set_header Upgrade $http_upgrade;
		        proxy_set_header Connection 'upgrade';
		        proxy_set_header Host $host;
		        proxy_cache_bypass $http_upgrade;
		        sub_filter /node /;
		        proxy_redirect https://localhost:8081/ManagementPortal/ /ManagementPortal/;
       }
    }
}
```

### NGINX Configuration Documentation
Configuration information can be found here: [NGINX Config](http://nginx.org/en/docs/http/ngx_http_proxy_module.html).

```
proxy_pass
```
Sets the protocol and address of a proxied server and an optional URI to which a location should be mapped. As a protocol, “http” or “https” can be specified. The address can be specified as a domain name or IP address, and an optional port:
proxy_pass http://localhost:8000/uri/;
or as a UNIX-domain socket path specified after the word “unix” and enclosed in colons:
proxy_pass http://unix:/tmp/backend.socket:/uri/;



```
proxy_http_version
```
Sets the HTTP protocol version for proxying. By default, version 1.0 is used. Version 1.1 is recommended for use with keepalive connections and NTLM authentication.


```
proxy_set_header
```
Allows redefining or appending fields to the request header passed to the proxied server. The value can contain text, variables, and their combinations. These directives are inherited from the previous level if and only if there are no proxy_set_header directives defined on the current level. By default, only two fields are redefined:
proxy_set_header Host       $proxy_host;
proxy_set_header Connection close;


```
proxy_cache_bypass
```
Defines conditions under which the response will not be taken from a cache. If at least one value of the string parameters is not empty and is not equal to “0” then the response will not be taken from the cache:
proxy_cache_bypass $cookie_nocache $arg_nocache$arg_comment;
proxy_cache_bypass $http_pragma    $http_authorization;


```
sub_filter
```
Sets a string to replace and a replacement string. The string to replace is matched ignoring the case. The string to replace (1.9.4) and replacement string can contain variables. Several sub_filter directives can be specified on one configuration level (1.9.4). These directives are inherited from the previous level if and only if there are no sub_filter directives defined on the current level.


```
proxy_redirect
```
Sets the text that should be changed in the “Location” and “Refresh” header fields of a proxied server response. Suppose a proxied server returned the header field “Location: http://localhost:8000/two/some/uri/”. The directive
proxy_redirect http://localhost:8000/two/ http://frontend/one/;
will rewrite this string to “Location: http://frontend/one/some/uri/”.
A server name may be omitted in the replacement string:
proxy_redirect http://localhost:8000/two/ /;
then the primary server’s name and port, if different from 80, will be inserted.

### Server Configuration
We have had issues connecting to localhost, errors similar to this would appear in the NGINX log file (/var/log/nginx/error.log):

connect() to 127.0.0.1:8005 failed (13: Permission denied) while connecting to upstream, client: 127.0.0.1, server: localhost, request: "GET / HTTP/1.1", upstream: "http://127.0.0.1:8005/", host: "localhost:8080"

 This appears to be caused by the default configuration for SELinux.  To resolve the problem, run the following command:

```
 sudo setsebool -P httpd_can_network_connect 1  
 ```

 A thread discussing this problem can be found [here](http://stackoverflow.com/questions/23948527/13-permission-denied-while-connecting-to-upstreamnginx?rq=1).
