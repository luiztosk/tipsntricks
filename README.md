

## Use porkcron to download your cert and key from Porkbun
*2025-12-30 23:35:00-03:00*


I made PR [#15](https://github.com/tmzane/porkcron/pull/15) to `tmzane/porkcron` which fixes the issue where the docker container failed to access its parent folder. I added hardlinks to avoid too many complications, but there are probably a cleaner way to do this.

Anyway, as it is the container successfully downloaded my Porkbun cert and key, and just now I got `nginx` running in HTTPS on my workstation, accessible from the network. This will be very useful to manage my docker containers as now I can create a local docker registry and enable automatic updates using docker swarm.

On `/etc/nginx/nginx.conf` I uncommented the `443 ssl` `server` section following the [official guide on HTTPS](https://nginx.org/en/docs/http/configuring_https_servers.html) and then pointed the `ssl_certificate` and `ssl_certificate_key` vars to `/etc/ssl/certs/mydomain.com.pem` and `/etc/ssl/private/mydomain.com.key` respectively, that I'd copied from the docker volume created by `porkcron` (I still need to figure the nginx docker permissions, so it can use the same volume, but for now a local install running as root is enough to test this shit).


## Convert multiple ebooks from `.epub` to `.mobi` in parallel

Kindle accepts `.mobi` files directly, just copy them to the root `documents` folder. To convert from the more common `.epub`, we can use the `calibre` CLI utility `ebook-convert`.

To run multiple instances in parallel, converting all `.epub` files in current dir, and log each process into a `.log` with the same name, do:

```
foreach x (*.epub) {
    echo -e "\n$x";
    ebook-convert $x .mobi > "$x".log 2>&1 & 
}
```
In the 3rd line, `>` redirects `stdin` to the file `"$x".log`, and `2>&1` redirects stderr to stdin to log them together. The final `&` runs `ebook-convert` as a background process, that can be monitored in `htop` with the printed PID, or using the `jobs` utility.


## Connect to `ssh` host when it is online:
```
until nc -vzw 2 $host 22; do sleep 2; done && ssh $host
```
Very useful when rebooting hosts (VMs, IoT).


## Run `docker compose` on `git` hook `post-receive`:
In the git server (bare repo) write the script `post-receive` in the folder `hooks`, and make it executable and owned by `git:git`:
```
#!/bin/bash
TARGET="/srv/git/flaskwiki/flaskwiki"
GIT_DIR="/srv/git/flaskwiki/flaskwiki.git"

echo "--- Deploying $GIT_DIR to Orange Pi ---"

git --work-tree=$TARGET --git-dir=$GIT_DIR checkout -f

cd $TARGET

docker compose up -d --build

echo "--- Deployment Complete ---"
```


## Log from Flask app:
```
app.logger.warning(f'getting md files from {MARKDOWN_DIR}')
```
Any level lower that `warning` requires config.


## Access logs from docker container:
```
docker logs container_name
```