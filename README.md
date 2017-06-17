# Optimizations for [tneu.edu.ua](http://www.tneu.edu.ua/)

> Make tneu.edu.ua fast again

## Motivation

Sometimes you urgently need look up teacher's name, phone number or faculty info.

But it takes extremely long to load TNEU website over slow 3G connection. 😔

So the goal is to make it faster! ⚡️

## How and what?

### 1. Register a domain name

It's nice to have something shorter than www.tneu.edu.ua.

Free for 1 year. Here: [dot.tk](http://dot.tk)

Now we have tneu.ml

### 2. Setup a reverse proxy

A server with nginx will do all the magic proxying requests to tneu.edu.ua.

Server is on DigitalOcean, smallest $5 droplet with 512MB RAM is more than we need.

Looks simple in the beginning:

```conf
server {
    listen      80;
    server_name tneu.ml;

    location / {
      proxy_pass http://www.tneu.edu.ua;
      proxy_set_header X-Real-IP $remote_addr;
      proxy_set_header Host www.tneu.edu.ua;
      proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    }
}
```

`docker run nginx` and we have a copy of tneu website.

### 3. Setup Cloudflare

For free it will provide a shared SSL certificate, https://tneu.ml.

So we could even setup force SSL - to redirect users from http to https automatically:

```conf
if ($http_x_forwarded_proto = "http") {
    return 301 https://$server_name$request_uri;
}
```

Furthermore, Cloudflare will play as a CDN for static assets + HTTP/2 + SPDY and some other things as a bonus.

**Before**

![image](https://user-images.githubusercontent.com/3817380/27256784-76cf42b2-53c9-11e7-9ae8-f41911054257.png)

**After**

![image](https://user-images.githubusercontent.com/3817380/27256787-94f7d86c-53c9-11e7-9ebd-24c225fdd369.png)


### 4. Setup reverse proxy so it actually works

So it occurred some of the links are hard-coded to use tneu.edu.ua domain.
Which obviously doesn't work when you view them from tneu.ml.

So this is a fix:

```conf
sub_filter_once off;
sub_filter_types *;
sub_filter 'www.tneu.edu.ua' 'tneu.ml';
```

### 5. Setup Google's PageSpeed nginx module

See `_tneu.nginx.conf` file in the repo for more details.

But in essence, it compresses images, converts some of them to WebP format, minifies HTML and more.

Also it sets a fixed size for images, so page doesn't "jumps" when images are loading.

And most importantly, it defers all the synchronous JavaScript on the page.

1. tneu.ml
2. tneu.edu.ua

More than 2x improvement over a slow 3G connection with 0 cache for the 1st load.

![image](https://user-images.githubusercontent.com/3817380/27256930-a61cd828-53cd-11e7-9bcd-af1caf56b78d.png)

## Run

Docker image is based on nginx + [pagespeed module](https://developers.google.com/speed/pagespeed/module/)

Runs as a simple reverse proxy on port 80.

```sh
$ docker run -d -p 80:80 \
  -v ~/_tneu.nginx.conf:/etc/nginx/sites-enabled/_tneu.nginx.conf \
  --name tneu funkygibbon/nginx-pagespeed
```
