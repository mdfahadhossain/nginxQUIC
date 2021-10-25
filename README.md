# How to setup Nginx with QUIC/HTTP3 support

You're about to build and install Nginx server with QUIC/HTTP3 support. I am assuming that you already know what it is and how it works. Also keep in mind that, this build is not suppose to run on a production server.

### Either you build one or you can download which is already built and tested for you from the [release](https://github.com/fhdaax/nginxQUIC/releases) of this repository, and follow only the setup guide down below.

Follow up this step by step. Raise issue if you face any problem.

## Environment Setup

First of all, login to your server and make sure that the system is up to date.

```bash
sudo apt update && sudo apt upgrade -y
```

Get the necessary tools for the build.

```bash
sudo apt install -y dpkg-dev uuid-dev mercurial golang libunwind-dev unzip cmake
```

## NGINX Source Code

Navigate to the folder where you want to build `nginx`, in our case, it is `~/nginx`.

```bash
mkdir -p ~/nginx; cd $_
```

We need to create a key signature so we can download repo from NGINX packages.

```bash
wget https://nginx.org/keys/nginx_signing.key
sudo apt-key add nginx_signing.key
```

After that, we have to edit the `/etc/apt/sources.list` and add the NGINX repositories.
```bash
sudo nano /etc/apt/sources.list
```

Add these two lines at very bottom of the file.

```shell
deb https://nginx.org/packages/mainline/ubuntu focal nginx
deb-src https://nginx.org/packages/mainline/ubuntu focal nginx
```
Update the version name according to your system version name. In our case, it's `focal`.
- Ubuntu 20.10: `groovy`
- Ubuntu 20.04.1 LTS: `focal`
- Ubuntu 18.04.5 LTS: `bionic`
- Ubuntu 16.04.7 LTS: `xenial`

Update your system once again.

```bash
sudo apt update
```

Now, build dependencies for NGINX and pull the source code.

```bash
sudo apt build-dep nginx -y
sudo apt source nginx
```
After that, you should see a folder named nginx with version on the current.

## Clone Nginx-QUIC

We already got the source code of nginx and now we need to get nginx-quic and sync with the source code.

```bash
hg clone -b quic https://hg.nginx.org/nginx-quic
```

Get the ownership of the `nginx` directory.

```bash
sudo chown -R username:username ~/nginx
```
Replace `username` with your username.

Overwrite all content of the nginx-quic directory to the nginx source code folder.

```bash
rsync -r nginx-quic/ nginx-x.x.x
```

## BoringSSL module

We will need a BoringSSL library that provides QUIC support.

cd to the following.

```bash
cd nginx-x.x.x/debain/
```

Clone BoringSSL repo.

```bash
git clone https://github.com/google/boringssl
```

Create and `cd` to `boringssl/build` directory.

```bash
mkdir -p boringssl/build; cd $_
```

Compile it to make it ready to use with nginx source code.

```bash
cmake ../
make -j 8
```
This may take a few moment.

## Additional modules

We will add the following two additional modules with nginx.

👉 [Pagespeed](https://developers.google.com/speed)
👉 [Brotli](https://www.nginx.com/products/nginx/modules/brotli)

### Pagespeed

Back to the debain directory, make a `modules` directory and cd into it.

```bash
cd ../..
mkdir -p modules; cd $_
```

Download and extract pagespeed to the appropriate directory.

```bash
wget https://github.com/apache/incubator-pagespeed-ngx/archive/v1.13.35.2-stable.zip
unzip v1.13.35.2-stable.zip
mv incubator-pagespeed-ngx-1.13.35.2-stable ngx_pagespeed
```

Add PSOL library for ngx_pagespeed

```bash
cd ngx_pagespeed
wget https://dl.google.com/dl/page-speed/psol/1.13.35.2-x64.tar.gz
tar -xzvf 1.13.35.2-x64.tar.gz
```

### Brotli

Back to the `debain/modules` directory and clone Brotli repo.

```bash
cd ..
git clone --recursive https://github.com/google/ngx_brotli
```

## Configure the QUIC and the modules

Get back to the `dabain` directory. Sit back, we need to make a few updates to the `rules` file.

```bash
cd ..
nano -l rules
```
Follow these steps for the both lines that says `config.env.nginx` and `config.env.nginx_debug`, you might find these at line of 41 and 46 respectively.

Add the following after `–with-stream_ssl_preread_module`

```
--with-http_v3_module --with-http_quic_module --with-stream_quic_module
```

Then we need to add `-Wno-ignored-qualifiers` at the `CFLAGS=""` to disable compiler from throwing qualifiers error.

```
CFLAGS="-Wno-ignored-qualifiers"
```

Now, let's add BoringSSL. You might already see these options `--with-cc-opt` and `--with-ld-opt`. Overwrite them with the following.

```
--with-cc-opt="-I../boringssl/include $(CFLAGS)" --with-ld-opt="-L../boringssl/build/ssl -L../boringssl/build/crypto $(LDFLAGS)"
```

### Add modules to the config

At the `./configure` of *config.env.nginx* and *config.env.nginx_debug*, add the following lines right after `--sbin-path=/usr/sbin/nginx`

```
--add-module="$(CURDIR)/debian/modules/ngx_pagespeed" --add-module="$(CURDIR)/debian/modules/ngx_brotli"
```

Make sure you do the same changes on both line `config.env.nginx` and `config.env.nginx_debug`.

It's time to compile.

## Conpile Nginx QUIC

Before we compile, we have to add some finishing touch so we distinguish we are using mod build for NGINX.
First, we have to edit the `changelog` file.

```bash
nano changelog
```

Then add the following at the very beginning.

```
nginx (x.x.x-1~focal+pagespeed+brotli+http3+quic) focal; urgency=low

  * x.x.x-1

 -- YOUR_NAME <your_name@domain.com>  Fri, 15 Oct 2021 22:30:00 +0600
 ```
 
 In above to the value, change the following
 - `x.x.x` to the right version of the nginx source code
 - `focal` with the right system version name if it's not a `focal`
 - your name and email
 - time log

### Compile time

Now everything is ready to go. Let's navigate to the source code directory and compile...

```bash
cd ..
sudo dpkg-buildpackage -b
```

Once it started compiling, it will asked if you want to include PSOL debug version, just type `yes`. After it completes, it will create the deb file at the parent directory.

Let's use that `.deb` package to install nginx.

```bash
cd ..
sudo dpkg -i nginx_x.x.x-1~focal+pagespeed+brotli_amd64.deb
```

Check nginx version

```bash
nginx -v
```

## Setting up Nginx QUIC

At this point, our nginx quic is installed. Let's configure the server. We will set things up for the public directory first.

```bash
sudo mkdir -p /var/www; cd $_
sudo mkdir -p html; cd $_
sudo nano index.html
```

Paste the following.

```html
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="UTF-8" />
    <meta http-equiv="X-UA-Compatible" content="IE=edge" />
    <meta name="viewport" content="width=device-width, initial-scale=1.0" />
    <title>Nginx QUIC</title>
    <style>
      body {
        margin: 0;
        padding: 0;
        height: 100vh;
        display: flex;
        align-items: center;
        justify-content: center;
      }
      h1 {
        margin: 0;
      }
    </style>
  </head>
  <body>
    <h1>Nginx QUIC</h1>
  </body>
</html>
```

Change `/var/www` directory owner to `www-data`.

```bash
sudo chown -R www-data:www-data /var/www
```

Let's go configure nginx server

```bash
cd /etc/nginx
sudo rm -f nginx.conf conf.d/default.conf
sudo nano nginx.conf
```

Paste the following

```
user www-data;
worker_processes auto;
include /etc/nginx/modules-enabled/*.conf;

error_log  /var/log/nginx/error.log warn;
pid        /var/run/nginx.pid;


events {
	worker_connections  1024;
}


http {
	include /etc/nginx/mime.types;
	default_type  application/octet-stream;
	server_tokens off;

	add_header Set-Cookie "Path=/; HttpOnly; Secure";

	##
	# PageSpeed Settings
	##
	pagespeed on;
	pagespeed FileCachePath /var/ngx_pagespeed_cache;
    
	##
	# Access/Error Log Settings
	##
	log_format quic '$remote_addr - $remote_user [$time_local] '
                        '"$request" $status $body_bytes_sent '
                        '"$http_referer" "$http_user_agent" "$quic"';
	access_log  /var/log/nginx/access.log quic;
	error_log /var/log/nginx/error.log;

	##
	# Http Core Module Settings
	##
	sendfile on;
	tcp_nopush on;
	tcp_nodelay on;
	keepalive_timeout 65;
	types_hash_max_size 2048;

	##
	# Gzip Settings
	##
	gzip on;
	gzip_comp_level 5;
	gzip_min_length 256;
	gzip_proxied any;
	gzip_vary on;
	pagespeed FetchWithGzip off;
	pagespeed HttpCacheCompressionLevel 0;
	gzip_types text/plain text/css application/json application/javascript text/xml application/xml application/x-font-ttf application/x-web-app-manifest+json application/xml+rss text/javascript image/svg+xml image/x-icon;

	##
	# Brotli Settings
	##
	brotli on;
	brotli_comp_level 6;
	brotli_static on;
	brotli_types application/octec-stream text/xml image/svg+xml application/x-font-ttf image/vnd.microsoft.icon application/x-font-opentype application/json font/eot application/vnd.ms-fontobject application/javascript font/otf application/xml application/xhtml+xml text/javascript application/x-javascript text/plain application/x-font-trutype application/xml+rss image/x-icon font/opentype text/css image/x-win-bitmap application/x-web-app-manifest+json;
    
	##
	# SSL Configuration
	##
	quic_retry on;
	ssl_early_data on;
	ssl_session_timeout 1d;
	ssl_session_cache shared:SSL:10m;
	ssl_session_tickets off;
	#ssl_stapling on; # not supported by boringssl
	ssl_stapling_verify on;
	#http3_max_field_size 5000;
	http3_max_table_capacity 50;
        http3_max_blocked_streams 30;
        http3_max_concurrent_pushes 30;
        http3_push 10;
        http3_push_preload on;
	ssl_protocols TLSv1.2 TLSv1.3;
	ssl_prefer_server_ciphers on;
	ssl_ciphers TLS_AES_128_GCM_SHA256:TLS_AES_256_GCM_SHA384:TLS_CHACHA20_POLY1305_SHA256;

	##
	# FastCGI Cache Settings
	##
	fastcgi_cache_path /etc/nginx-cache levels=1:2 keys_zone=phpcache:100m inactive=60m;
	fastcgi_cache_key "$scheme$request_method$host$request_uri";
	fastcgi_ignore_headers Cache-Control Expires;

	include /etc/nginx/conf.d/*.conf;
	include /etc/nginx/sites-enabled/*;
}
```

Let's configure individual site. Starting with linking

```bash
sudo mkdir -p /etc/nginx/sites-available
sudo ln -s /etc/nginx/sites-available /etc/nginx/sites-enabled
```

Now, create a site with the following

```bash
sudo nano sites-available/yourdomain.com
```
And paste the following
```
server{
    listen 80;
    index index.html;
    server_name yourdomain.com;

    root /var/www/html;
}
```

We are good to go. Before restarting nginx, check if the configuration is ok.

```bash
sudo nginx -t
```

Restart nginx

```bash
sudo systemctl restart nginx
```

If everything is ok, your site should go live.

## SSL Certificate

Let's install `Certbot`. Since our system is `focal`, we can install certbot from *snap* package.

```bash
sudo snap install certbot --classic
```

> Run the following if you are using below 20.04 LTS.
> ```bash
> sudo add-apt-repository ppa:certbot/certbot
> sudo apt update
> sudo apt install -y certbot
> ```

Issue an SSL certificate for your site.

```bash
sudo certbot --nginx -d yourdomain.com
```

Answer the following questions and certbot will issue and configure a certificate for your site.

Here's comes the fun part setting up HTTP2 and QUIC/HTTP3

```bash
sudo nano sites-available/yourdomain.com
```

Replace the content with the following. (Remember the certificate path and change it after the replacement)

```
server{
    listen 443 http3 quic reuseport;
    listen 443 ssl http2;

    add_header alt-svc 'h3-29=":443"; ma=3600' always;

    index index.html index.nginx-debian.html;
    server_name yourdomain.com;

    root /var/www/html;

    ssl_certificate /etc/letsencrypt/live/yourdomain.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/yourdomain.com/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; 

}
server{
    listen 80;
    server_name yourdomain.com;
    if ($host = yourdomain.com) {
        return 301 https://$host$request_uri;
    }

    return 404;
}
```

Hurrah! we are done. Before restarting nginx, check if the configuration is ok.

```bash
sudo nginx -t
```

Restart nginx.

```bash
sudo systemctl restart nginx
```

**Enjoy HTTP3**




# $100 free credit in DigitalOcean

DigitalOcean is providing you $100 free credit for two months to use their products including VPS droplets starting from $5/m. Signup with my referral image to redeem the offer.

[![DigitalOcean Referral Badge](https://web-platforms.sfo2.cdn.digitaloceanspaces.com/WWW/Badge%201.svg)](https://www.digitalocean.com/?refcode=93f03a33b121&utm_campaign=Referral_Invite&utm_medium=Referral_Program&utm_source=badge)
