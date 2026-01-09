
## Nginx Reverse Proxy, a simple config in multiple files
*2026-01-06 22:25:00-03:00*
*edited in 2026-01-09 00:37:00-03:00*


### Main Entry Point and HTTPS redirect (mainhost.mydomain.com)

I decided to just use nginx directly on my OrangePi host, as I was struggling with Docker and its local storage permissions.

The main entry point contains my SSL cert locations, a simple log format, and two servers, one that redirects all HTTP requests to HTTPS, and another one that currently displays nginx's default page, but will be the home of my host:

```bash
ssl_certificate /etc/porkcron/mydomain.com/certificate.pem;
ssl_certificate_key /etc/porkcron/mydomain.com/private_key.pem;

log_format upstream_logging '[$time_local] $remote_addr -> $scheme://$server_name to: $upstream_addr: $request $uri $server_name $request_uri $host';

server {
	listen 80 default_server;
	server_name _;
	access_log /var/log/nginx/root.log upstream_logging;
	return 301 https://$host$request_uri;
}
server {
	listen 443 ssl default_server;
	server_name _;
	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;
	location / {
		try_files $uri $uri/ =404;
	}
}
```

### Pi-hole webui on myhost.mydomain.com/pihole

Starting on [FTL v6.1](https://pi-hole.net/blog/2025/03/30/pi-hole-ftl-v6-1-web-v6-1-and-core-v6-0-6-released/) we can now set a `prefix` to use the Pi-Hole under a reverse proxy. I've set `webserver.paths.prefix="pihole"` and `webserver.paths.domain="host.mydomain.com"`. Then the nginx config looks like (according to [this forum thread](https://discourse.pi-hole.net/t/pihole-nginx-prefix-multiplexing-not-working/78511/5)):

```bash
server {
	listen 443 ssl;
	access_log /var/log/nginx/pihole.log upstream_logging;
	location /pihole/ {
		proxy_pass http://127.0.0.1:8080/;

		proxy_set_header Host $host;
		proxy_set_header X-Real-IP $remote_addr;
		proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
		proxy_set_header X-Forwarded-Proto $scheme;

		proxy_cookie_path / /pihole/;
	}
}
```
This is all on IPv4, that's why I used the IP instead of `localhost`. ~~The first redirect is for convenience, so that we can just type `pihole.mydomain.com` and it opens the webui. The second one can't be a path because the pihole uses `api` URIs.~~ No need for the redirect since we use the `prefix` now.

### Zabbix behind a proxy too

I used the same `location /` block on the Zabbix nginx config, that I copied from `/etc/zabbix/nginx.conf` into `/etc/nginx/sites-available/zabbix.mydomain.com.conf`, added that `server` block on top with `server_name zabbix.mydomain.com`, changed the log name, and changed the ports accordingly. In Zabbix's own server block, I just uncommented the `listen` line and set it to the same port. Now it also opens with HTTPS in the browser. It looks like so:
```bash
server {
	listen 443 ssl;
	server_name zabbix.mydomain.com;
	access_log /var/log/nginx/zabbix.log upstream_logging;
	location / {
		proxy_pass http://127.0.0.1:8090;
	}
}

server {
        listen          8090;
        root    /usr/share/zabbix/ui;

        index   index.php;

        client_max_body_size 5m;
        .
        .
        .
```


### Main takeouts from configuring Nginx on an OrangePi

So far I enjoyed the experience. It was a bit convoluted until I got the hang of how the nginx config hierarchy works, and learned that you need to test and reload the configs, and eventually restart it too. Sometimes I also had to reset the network connection on the clients, to avoid old cached redirects prevent me testing my new configs.

To test, reload and restart `nginx` using `systemd`, do:
```bash
sudo nginx -t && sudo systemctl reload nginx && sudo systemctl restart nginx
```
And that's it, I hope I took note of all the main points, as this even if entertaining was also quite painful and if I ever have to go through it again, at least I have these notes.

P.S.: Ah, and each one of those services I put in a separate file in `/etc/nginx/sites-available`, then symlinked each of them to `/etc/nginx/sites-enabled`. Once I deleted a file thinking I was deleting a link, and had to redo it from memory.




## Use porkcron to download your cert and key from Porkbun
*2025-12-30 23:35:00-03:00*
> **_NOTE:_** I am now using it via `systemd`, as I decided to use nginx directly on my OrangePI host.


I made PR [#15](https://github.com/tmzane/porkcron/pull/15) to `tmzane/porkcron` which fixes the issue where the docker container failed to access its parent folder. I added hardlinks to avoid too many complications, but there are probably a cleaner way to do this.

Anyway, as it is the container successfully downloaded my Porkbun cert and key, and just now I got `nginx` running in HTTPS on my workstation, accessible from the network. This will be very useful to manage my docker containers as now I can create a local docker registry and enable automatic updates using docker swarm.

On `/etc/nginx/nginx.conf` I uncommented the `443 ssl` `server` section following the [official guide on HTTPS](https://nginx.org/en/docs/http/configuring_https_servers.html) and then pointed the `ssl_certificate` and `ssl_certificate_key` vars to `/etc/ssl/certs/mydomain.com.pem` and `/etc/ssl/private/mydomain.com.key` respectively, that I'd copied from the docker volume created by `porkcron` (I still need to figure the nginx docker permissions, so it can use the same volume, but for now a local install running as root is enough to test this shit).


## Convert multiple ebooks from `.epub` to `.mobi` in parallel

Kindle accepts `.mobi` files directly, just copy them to the root `documents` folder. To convert from the more common `.epub`, we can use the `calibre` CLI utility `ebook-convert`.

To run multiple instances in parallel, converting all `.epub` files in current dir, and log each process into a `.log` with the same name, do:

```bash
foreach x (*.epub) {
    echo -e "\n$x";
    ebook-convert $x .mobi > "$x".log 2>&1 & 
}
```
In the 3rd line, `>` redirects `stdin` to the file `"$x".log`, and `2>&1` redirects stderr to stdin to log them together. The final `&` runs `ebook-convert` as a background process, that can be monitored in `htop` with the printed PID, or using the `jobs` utility.


## Connect to `ssh` host when it is online:
```bash
until nc -vzw 2 $host 22; do sleep 2; done && ssh $host
```
Very useful when rebooting hosts (VMs, IoT).


## Run `docker compose` on `git` hook `post-receive`:
In the git server (bare repo) write the script `post-receive` in the folder `hooks`, and make it executable and owned by `git:git`:
```bash
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
```bash
app.logger.warning(f'getting md files from {MARKDOWN_DIR}')
```
Any level lower that `warning` requires config.


## Access logs from docker container:
```bash
docker logs container_name
```