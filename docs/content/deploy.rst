Deployment
==========

.. _deploy:

Nginx configuration
~~~~~~~~~~~~~~~~~~~

Minimal Nginx version - 1.3.13

Here is an example Nginx configuration to deploy Centrifuge.

.. code-block:: bash

    #user nginx;
    worker_processes 4;

    #error_log /var/log/nginx/error.log;

    events {
        worker_connections 1024;
        #use epoll;
    }

    http {
        # Enumerate all the Tornado servers here
        upstream centrifuge {
            #sticky;
            ip_hash;
            server 127.0.0.1:8000;
            #server 127.0.0.1:8001;
            #server 127.0.0.1:8002;
            #server 127.0.0.1:8003;
        }
        include mime.types;
        default_type application/octet-stream;

        keepalive_timeout 65;
        proxy_read_timeout 200;
        sendfile on;
        tcp_nopush on;
        tcp_nodelay on;
        gzip on;
        gzip_min_length 1000;
        gzip_proxied any;

        # Only retry if there was a communication error, not a timeout
        # on the Tornado server (to avoid propagating "queries of death"
        # to all frontends)
        proxy_next_upstream error;

        map $http_upgrade $connection_upgrade {
            default upgrade;
            ''      close;
        }

        server {
            listen 8081;
            server_name localhost;

            location ^~ /static/ {
                root /var/www/different/python/centrifuge/src/src/centrifuge/frontend;
                if ($query_string) {
                    expires max;
                }
            }
            location = /favicon.ico {
                rewrite (.*) /static/favicon.ico;
            }
            location = /robots.txt {
                rewrite (.*) /static/robots.txt;
            }
            location / {
                proxy_pass_header Server;
                proxy_set_header Host $http_host;
                proxy_redirect off;
                proxy_set_header X-Real-IP $remote_addr;
                proxy_set_header X-Scheme $scheme;
                proxy_pass http://centrifuge;
            }
            location /socket {
                proxy_buffering off;
                proxy_pass http://centrifuge;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
            }
            location /connection/websocket {
                proxy_buffering off;
                proxy_pass http://centrifuge;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
            }
            location /connection {
                proxy_buffering off;
                proxy_pass http://centrifuge;
                proxy_http_version 1.1;
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection $connection_upgrade;
            }
        }
    }


Look carefully at commented ``sticky;`` directive in upstream section.

In this configuration example we use ``ip_hash`` directive to proxy client with the same ip
address to the same backend process. This is very important when we have several processes.

When client connects to Centrifuge - session created - and to communicate those client must
send all next requests to the same backend process. But ``ip_hash`` is not the best choice
in this case, because there could be situations where a lot of different browsers are coming
with the same IP address (behind proxies) and the load balancing system won't be fair.
Also fair load balancing does not work during development - when all clients connecting from
localhost.

So best solution would be using something like `nginx-sticky-module <http://code.google.com/p/nginx-sticky-module/>`_
which uses a cookie to track the upstream server for making each client unique.


Supervisord configuration example
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

.. _supervisord_configuration:

In 'deploy' folder of Centrifuge's repo you can find supervisord configuration
example. Something like this:

supervisord.conf:

.. code-block:: bash

    [unix_http_server]
    file=/tmp/supervisor.sock

    [inet_http_server]
    port=127.0.0.1:9001

    [supervisord]
    logfile=/tmp/supervisord.log
    pidfile=/tmp/supervisord.pid

    [rpcinterface:supervisor]
    supervisor.rpcinterface_factory = supervisor.rpcinterface:make_main_rpcinterface

    [include]
    files = *.supervisor


centrifuge.supervisor

.. code-block:: bash

    [program:centrifuge]
    process_name = centrifuge-%(process_num)s
    command = centrifuge --conf=%(here)s/../src/config.json --port=%(process_num)s --log_file_prefix=/tmp/%(program_name)s-%(process_num)s.log
    numprocs = 1
    numprocs_start = 8000