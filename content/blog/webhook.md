+++
title = "Server webhooks with nginx and systemd"
date = "2025-08-23"
description = "Use nginx and systemd to receive webhooks and trigger deployments from GitHub Actions. Secret URLs, per-env logs, systemd path units, and practical security tips."
tags = ["webhooks", "nginx", "systemd", "deployment", "ci-cd", "github-actions", "devops", "linux", "automation", "security"]
+++

Webhooks let one service notify another when something happens, for example, triggering a deploy right after you push code to your Git repository. Many tools (and SaaS platforms) exist to receive and process webhooks, such as [adnanh/webhook](https://github.com/adnanh/webhook). But if you already run nginx, you can build a lightweight, reliable webhook receiver without adding another daemon.

In this post, we’ll use nginx and systemd to:
- Expose a secret, exact-match URL that returns 204 on success.
- Capture the `version` query parameter into a per-environment log.
- Have systemd react to that log change and run a oneshot deploy script.

I’ll also show a GitHub Actions example for triggering the endpoint, plus a few notes on log rotation and security.

Below is an example with `staging` and `release` environments.

## nginx: minimal webhook endpoints

- One access log for audit.
- One per-environment access log that contains only the `version` query string.
- Exact-match locations with a long, unguessable secret segment (you can use, for example, `openssl rand -hex 24`).
- 204 response on success, 404 otherwise.

Note: Make sure the directory for logs exists and nginx can write to it.

```
# /etc/nginx/conf.d/webhook.conf
log_format request '$request $status';
log_format version $arg_version;

server {
  server_name _;

  listen [::]:443 ssl http2;
  listen 443 ssl http2;

  root /var/www/html;
  access_log /var/log/nginx/webhook/webhook.log request;

  location = /2ff16954429425ead56aaaacc01ec8c158fe57f04ce75a49/deploy-app/staging {
    access_log /var/log/nginx/webhook/app/staging version;
    return 204;
  }

  location = /edbf95c935a1fb9f0566ec15f618a8b4b9d94e7fe0e4b831/deploy-app/prod {
    access_log /var/log/nginx/webhook/app/prod version;
    return 204;
  }

  location / {
    return 404;
  }
```

Create log directories and reload nginx:
```bash
sudo mkdir -p /var/log/nginx/webhook/app
sudo nginx -t &amp;&amp; sudo systemctl reload nginx
```

## systemd service: do the work

- Use tail -n1 so if multiple hits land before your service runs, you deploy the most recent version.
- Truncate the per-environment log after a successful run to make the next trigger clean.
- Keep it oneshot and small. Consider adding sandboxing options later.

```
# /lib/systemd/system/deploy-app@.service
[Unit]
Description=Deploy app (%i) triggered by nginx webhook
After=network-online.target
Wants=network-online.target

[Service]
Type=oneshot

# Ensure there's something in the per-env log before proceeding
ExecStartPre=/usr/bin/test -s /var/log/nginx/webhook/app/%i
# Capture the last requested version into a place your deploy script reads
ExecStart=/usr/bin/sh -c 'tail -n1 /var/log/nginx/webhook/app/%i > /etc/app/%i/version';
# Run your deploy; receives 'staging' or 'release' as $1
ExecStart=/usr/local/bin/deploy-app %i
# Clear the trigger log for the next event
ExecStopPost=/usr/bin/truncate -s 0 /var/log/nginx/webhook/app/%i
# Recommended hardening (tune to your environment)
#User=app
#Group=app
#WorkingDirectory=/etc/app/%i
#ProtectSystem=strict
#ProtectHome=true
#PrivateTmp=true
#NoNewPrivileges=true
```

## systemd path: trigger on log change

- The path unit watches the per-environment log for modifications.
- Instantiate one path unit per environment.

```
# /lib/systemd/system/deploy-app@.path
[Unit]
Description=Watch webhook log and trigger deploy for %i

[Path]
PathModified=/var/log/nginx/webhook/app/%i
Unit=deploy-app@%i.service

[Install]
WantedBy=multi-user.target
```

Enable and start the watchers:

```bash
sudo systemctl daemon-reload
sudo systemctl enable --now deploy-app@staging.path
sudo systemctl enable --now deploy-app@release.path
```

## GitHub Actions workflow

- Keep the endpoint names consistent with nginx: staging and release.
- The version is either a branch name or a tag value.
- Alternative github action you may consider: [fjogeleit/http-request-action](https://github.com/fjogeleit/http-request-action).

```yaml
# .github/workflows/deploy.yml
name: Trigger deploy
on:
  push:
    branches:
      - main
    tags:
      - 'v*'

deploy:
  name: Trigger deploy
  runs-on: ubuntu-latest

  steps:
  - name: Deploy staging
    if: github.ref == 'refs/heads/main'
    uses: satak/webrequest-action@v1.2.4
    with:
      url: "https://webhook.app.io/${{ secrets.WEBHOOK_SECRET }}/deploy-app/staging?version=main"
      method: GET

  - name: Get release version
    if: startsWith(github.ref, 'refs/tags/v')
    id: version
    shell: bash
    run: echo "version=${GITHUB_REF#refs/tags/v}" >> $GITHUB_OUTPUT
    env:
      GITHUB_REF: ${{ github.ref }}

  - name: Deploy release
    if: startsWith(github.ref, 'refs/tags/v')
    uses: satak/webrequest-action@v1.2.4
    with:
      url: "https://webhook.app.io/${{ secrets.WEBHOOK_SECRET }}/deploy-app/release?version=${{ steps.version.outputs.version }}"
      method: GET
```

## deploy script example

Example skeleton expected by the service above.

```bash
#!/usr/bin/env bash
set -euo pipefail

ENVIRONMENT="${1:?usage: deploy-app <staging|release>}"
VERSION_FILE="/etc/app/${ENVIRONMENT}/version"

if [[ ! -s "$VERSION_FILE" ]]; then
  echo "No version provided" >&2
  exit 1
fi

VERSION="$(tail -n1 "$VERSION_FILE")"
echo "Deploying ${ENVIRONMENT} version: ${VERSION}"

# Example:
# git -C /srv/app fetch origin
# git -C /srv/app checkout --force "refs/heads/${VERSION}"   # or refs/tags/${VERSION}
# docker compose -f /srv/app/compose.${ENVIRONMENT}.yml pull
# docker compose -f /srv/app/compose.${ENVIRONMENT}.yml up -d --remove-orphans

echo "Done."
```

## Testing

- Staging test
  - `curl -i "https://webhook.app.io/SECRET/deploy-app/staging?version=test-123"`
  - Check: `systemctl status deploy-app@staging` and `/etc/app/staging/version`
- Release test
  - `curl -i "https://webhook.app.io/SECRET/deploy-app/release?version=1.2.3"`
  - Check: `systemctl status deploy-app@release` and `/etc/app/release/version`
- Endpoints should return HTTP 204.

## Log rotation

The per-environment logs are truncated after each run, but the audit log grows.

```
# /etc/logrotate.d/nginx-webhook
/var/log/nginx/webhook/webhook.log {
  rotate 12
  weekly
  missingok
  compress
  delaycompress
  notifempty
  create 0640 www-data adm
  postrotate
    [ -s /run/nginx.pid ] &amp;&amp; kill -USR1 $(cat /run/nginx.pid)
  endscript
}
```

## Security notes

- Keep the secret path long and random; rotate if exposed.
- Serve only over TLS.
- Optionally:
  - Add basic IP rate limiting on the server block.
  - Restrict to GET as shown (or switch to POST if you prefer).
  - If you need HMAC validation or more complex checks, consider adding an njs/lua validation step or move to a small webhook verifier service.

That’s it: nginx writes the version to a tiny file, systemd notices and runs your deploy. No extra daemon needed.
