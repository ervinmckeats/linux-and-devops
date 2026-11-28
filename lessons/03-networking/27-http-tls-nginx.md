# Lesson 27. HTTP, TLS, nginx reverse proxy

## Goal
Serve an app behind nginx with HTTPS, and debug HTTP and certificate problems.

## HTTP basics

A request: method, path, headers, optional body. A response: status code, headers, body.

```
GET /api/users HTTP/1.1          HTTP/1.1 200 OK
Host: example.com                Content-Type: application/json
User-Agent: curl/8.5             Cache-Control: no-cache
Accept: */*
                                 [{"id":1}]
```

| Code | Meaning | Usually |
|---|---|---|
| 2xx | success | 200 OK, 201 Created, 204 No Content |
| 3xx | redirect | 301 permanent, 302/307 temporary, 304 not modified |
| 4xx | client error | 400 bad request, 401 no auth, 403 forbidden, 404 not found, 429 rate limited |
| 5xx | server error | 500 app crashed, **502** proxy got a bad/no answer from upstream, **503** unavailable/overloaded, **504** upstream timeout |

502/504 at nginx almost always mean "the app behind nginx is down, slow, or listening elsewhere".

Methods: `GET` read, `POST` create, `PUT`/`PATCH` update, `DELETE`. `GET`, `PUT`, `DELETE` should be idempotent (safe to retry): important for retries (lesson 60).

HTTP/2 multiplexes many requests in one TCP connection. HTTP/3 runs over QUIC (UDP).

## TLS

TLS encrypts the connection and proves the server's identity.

1. Client connects and says which hostname it wants (**SNI**).
2. Server sends its **certificate** + intermediate chain.
3. Client checks: signed by a trusted CA? hostname matches (SAN)? not expired?
4. They agree on keys; the rest is encrypted.

Common failures: expired certificate, missing intermediate chain, wrong hostname, clock skew on the client.

```bash
openssl s_client -connect example.com:443 -servername example.com </dev/null | head -20
echo | openssl s_client -connect example.com:443 -servername example.com 2>/dev/null \
  | openssl x509 -noout -subject -issuer -dates -ext subjectAltName
openssl x509 -in cert.pem -noout -text
curl -vI https://example.com 2>&1 | grep -E 'expire|subject|issuer'
```

## nginx

```bash
sudo apt install nginx

# test config - ALWAYS before reload
sudo nginx -t

sudo systemctl reload nginx

# Debian layout
ls /etc/nginx/sites-available /etc/nginx/sites-enabled

tail -f /var/log/nginx/access.log /var/log/nginx/error.log
```

### Reverse proxy for an app on port 3000

`/etc/nginx/sites-available/app.conf`:

```nginx
upstream app_backend {
    server 127.0.0.1:3000;
    keepalive 32;
}

server {
    listen 80;
    server_name app.example.com;

    location / {
        proxy_pass http://app_backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    location /static/ {
        alias /var/www/app/static/;
        expires 7d;
    }

    client_max_body_size 20m;
}
```

```bash
sudo ln -s /etc/nginx/sites-available/app.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

### Free HTTPS with Let's Encrypt

DNS must point to the server and port 80 must be open.

```bash
sudo apt install certbot python3-certbot-nginx
sudo certbot --nginx -d app.example.com
sudo certbot renew --dry-run

# auto-renewal
systemctl list-timers | grep certbot
```

certbot edits the server block: adds `listen 443 ssl`, certificate paths and a redirect from HTTP to HTTPS.

### Useful extras

```nginx
# load balancing across instances
upstream app_backend {
    least_conn;
    server 10.0.1.11:3000 max_fails=3 fail_timeout=10s;
    server 10.0.1.12:3000 max_fails=3 fail_timeout=10s;
}

# rate limiting (in http {} block)
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
# in location:
limit_req zone=api burst=20 nodelay;

# security headers
add_header Strict-Transport-Security "max-age=31536000" always;
add_header X-Content-Type-Options nosniff always;
```

## Debugging 502

1. `tail /var/log/nginx/error.log`: "connect() failed (111: Connection refused)" = app not listening; "upstream timed out" = app too slow.
2. `ss -tlnp | grep 3000`: is the app listening, on which address?
3. `curl -v http://127.0.0.1:3000/` from the server: does the app answer directly?
4. `systemctl status app`, `journalctl -u app`.
5. SELinux on RHEL blocks nginx → network: `setsebool -P httpd_can_network_connect 1`.

## Practice

1. Run `python3 -m http.server 3000`, put nginx in front of it, open it through port 80.
2. Stop the app and look at the 502 in the browser and in `error.log`.
3. If you have a domain and a cloud VM, get a Let's Encrypt certificate. Otherwise make a self-signed one with `openssl req -x509` and configure `listen 443 ssl`.
4. Check the certificate expiry date with `openssl`.
5. Load-balance across two app instances on ports 3001 and 3002, stop one, watch nginx route around it.

## Check yourself

- What is the difference between 502, 503 and 504?
- What is SNI and why does `openssl s_client` need `-servername`?
- Why run `nginx -t` before every reload?

---

<p align="left">
  <a href="media/27-http-tls-nginx.mp4"><img width="640" src="media/27-http-tls-nginx.gif" alt="Lesson 27 animation: HTTP, TLS, nginx reverse proxy"></a>
</p>

## Next
[Lesson 28. Advanced networking](28-advanced-networking.md)
