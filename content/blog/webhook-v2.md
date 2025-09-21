+++
title = "Better and simpler server webhooks: nginx + Unix sockets + systemd"
date = "2025-09-06"
description = "Implement server webhooks by using nginx reverse proxy to Unix domain sockets with systemd socket activation."
tags = ["webhooks", "nginx", "systemd", "unix-sockets", "deployment", "ci-cd", "github-actions", "devops", "linux", "automation", "security"]
+++

In my [previous post about server webhooks](/blog/webhook/), I showed how to use nginx logging and systemd path units to trigger deployments. While that approach works, it has some limitations: file system polling, potential race conditions, and dependency on log file modifications. 

A more robust solution uses Unix domain sockets with systemd socket activation.

## nginx as a reverse proxy to Unix sockets

Instead of writing to log files, nginx forwards requests to Unix domain sockets:

```nginx
# /etc/nginx/conf.d/webhook.conf

upstream webhook {
    server unix:/run/webhook/webhook.sock;
}

server {
    server_name _;

    listen 80;

    root /var/www/html;
    access_log /var/log/nginx/webhook.log;

    location = /webhook/2ff16954429425ead56aaaacc01ec8c158fe57f04ce75a49 {
        limit_except POST { deny all; }
        proxy_pass http://webhook;
    }

    location / {
        return 404;
    }
}
```

## systemd socket unit

Socket unit creates and listens on the Unix domain socket:

```ini
# /etc/systemd/system/webhook.socket
[Unit]
Description=Webhook socket

[Socket]
ListenStream=/run/webhook/webhook.sock
SocketUser=www-data
SocketGroup=webhook
SocketMode=0660
Accept=yes

[Install]
WantedBy=sockets.target
```

## systemd service

We want a service instance to be spawned for each incoming connection.

Quoting [systemd.system(5)](https://man7.org/linux/man-pages/man5/systemd.socket.5.html):
> If Accept=yes is set, a service template foo@.service must exist
> from which services are instantiated for each incoming connection

```ini
# /etc/systemd/system/webhook@.service
[Unit]
Description=Webhook handler
After=network.target
PartOf=webhook.socket

[Service]
Type=simple
ExecStart=/usr/local/bin/webhook
User=webhook
Group=webhook
WorkingDirectory=/var/lib/webhook
StandardInput=socket
```

## webhook handler script

```bash
#!/usr/bin/env bash
# /usr/local/bin/webhook
set -euo pipefail

LOGTAG="webhook"
logger -t "${LOGTAG}" "Starting webhook handler"

failure() {
    echo "HTTP/1.0 500 Internal Server Error"
    echo "Content-Type: text/plain"
    echo "Content-Length: 14"
    echo ""
    echo "Request failed"
    logger -t "${LOGTAG}" "Error occurred during processing of request"
}

trap failure ERR

# Read HTTP request from stdin
read -r REQUEST_LINE
read -r # Skip the rest of the request line
logger -t "${LOGTAG}" "Received request: ${REQUEST_LINE}"
# read headers
while read -r line; do
   line=${line%%$'\r'}

   # If we've reached the end of the headers, break.
   [ -z "$line" ] && break

   REQUEST_HEADERS+=("$line")
done
printf -v headers_line '%s,' "${REQUEST_HEADERS[@]}"
logger -t "${LOGTAG}" "Request headers: ${headers_line%,}"

# Process the request (e.g., trigger deployment)

echo "HTTP/1.0 204 No Content"
echo "Content-Length: 0"
echo ""
```

## Activate webhook

```bash
sudo mkdir /var/lib/webhook /run/webhook
sudo useradd -r -s /usr/sbin/nologin -d /var/lib/webhook webhook
sudo chown -R webhook:webhook /var/lib/webhook
sudo chown -R www-data:webhook /run/webhook
sudo systemctl daemon-reload
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl reload nginx
chmod +x /usr/local/bin/webhook
```

## Test

To test the setup, you can use `curl` to send a POST request:

```bash
curl -X POST "https://yourdomain.com/webhook/2ff16954429425ead56aaaacc01ec8c158fe57f04ce75a49"
```

And check the logs:

```bash
sudo journalctl -u 'webhook*' -f
```
