# Health status of Elastic Beanstalk changed to 'Serve' due to the http code 101

## Environment

- Python3.8 running on 64bit Amazon Linux 2 version 3.2.0
- Nginx

## Issue

- Health check turns to Serve if the Elastic Beanstalk is hosting a websocket.

## Root Cause

- Http code 101 represents the request is a websocket. Elatic Beanstalk is using a so-called healthd daemon running on the EC2 instance to monitor cluster status and save logs to /var/log/nginx/healthd/application.log.$year-$month-$day-$hour. If the logs contain the http code other than 200. The status of the cluster will change to Serve.

## Solution

1. Create following folder under the working directory.

    ```bash
    mkdir .platform/nginx/conf.d/elasticbeanstalk/
    ```

2.  Save following `nginx.conf` to .platform/nginx/. The configuration file added mapping $status to $logflag, and change the http code from 101 to 0.

    ```bash
    cat <<EoF > .platform/nginx/nginx.conf
    #Elastic Beanstalk Nginx Configuration File

    user                    nginx;
    error_log               /var/log/nginx/error.log warn;
    pid                     /var/run/nginx.pid;
    worker_processes        auto;
    worker_rlimit_nofile    32634;

    events {
        worker_connections  10240;
    }

    http {
        include       /etc/nginx/mime.types;
        default_type  application/octet-stream;

        log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                        '$status $body_bytes_sent "$http_referer" '
                        '"$http_user_agent" "$http_x_forwarded_for"';

        include       conf.d/*.conf;
        map $status $logflag {
            101 0;
            default 1;
        }
        map $http_upgrade $connection_upgrade {
            default     "upgrade";
        }

        server {
            listen        80 default_server;
            access_log    /var/log/nginx/access.log main;

            client_header_timeout 60;
            client_body_timeout   60;
            keepalive_timeout     60;
            gzip                  off;
            gzip_comp_level       4;
            gzip_types text/plain text/css application/json application/javascript application/x-javascript text/xml application/xml application/xml+rss text/javascript;

            # Include the Elastic Beanstalk generated locations
            include conf.d/elasticbeanstalk/*.conf;
        }
    }
    EoF
    ```

3. Change `healtd.conf` and add if=$logflag to the end of access_log line.

    ```bash
    cat <<EoF > .platform/nginx/conf.d/elasticbeanstalk/healthd.conf
    if ($time_iso8601 ~ "^(\d{4})-(\d{2})-(\d{2})T(\d{2})") {
        set $year $1;
        set $month $2;
        set $day $3;
        set $hour $4;
    }

    access_log /var/log/nginx/healthd/application.log.$year-$month-$day-$hour healthd if=$logflag;
    EoF
    ```

4. compress the code and upload to Elastic Beanstalk environment.
