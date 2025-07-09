# 🟢 Self-hosting Drizzle Studio using Nginx

I couldn't find any guide to self-host Drizzle Studio, so I figured it out myself—and now I'm sharing it with you pals.
Here is a step-by-step guide to host Drizzle Studio with:

* 🔒 Basic Auth
* 🌐 HTTPS using Let's Encrypt
* 🧍 UI served at `/studio/`
* ❌ No CORS issues
* 📀 API served from the same origin

---


## 0 · Prerequisites

Install the following dependencies:

### Node.js ≥ 20

```bash
curl -fsSL https://deb.nodesource.com/setup_20.x | sudo -E bash -
sudo apt install -y nodejs
```

### pm2

```bash
sudo npm i -g pm2
```

### Nginx

```bash
sudo apt install nginx
```

### Certbot (Let's Encrypt)

```bash
sudo apt install certbot python3-certbot-nginx
```

### Apache utils (for Basic Auth password generation)

```bash
sudo apt install apache2-utils
```

### DNS Setup

Set an A record in your DNS provider (e.g., Cloudflare, Namecheap):

```
example.com → YOUR_SERVER_IP
```
or you can bypass any external DNS/CDN and test directly against your origin
```
echo "127.0.0.1 example.com" | sudo tee -a /etc/hosts
```

---

## 1 · Run Drizzle Studio with PM2

```bash
pm2 start "npx drizzle-kit studio --host 127.0.0.1 --port 4983" \
  --name drizzle-studio --interpreter bash
pm2 save
```

Test it locally:

```bash
curl -k https://127.0.0.1:4983/init
# Expected: 404 or small JSON
```

---

## 2 · Create Basic Auth User

```bash
sudo htpasswd -c /etc/nginx/.htpasswd studioadmin
```

Enter and confirm a password.

---

## 3 · Set up HTTPS with Let's Encrypt

```bash
sudo certbot --nginx -d example.com
```

Certificates will be stored at:

```
/etc/letsencrypt/live/example.com/
```

---

## 4 · Configure Nginx

### 4.1 Site Config: `/etc/nginx/sites-available/drizzle.conf`

```nginx
# 0)  HTTP  →  HTTPS
server {
    listen 80;
    server_name example.com;
    return 301 https://$host$request_uri;
}

# 1)  HTTPS  (UI + API, same origin, Basic-Auth)
server {
    listen 443 ssl http2;
    server_name example.com;

    ssl_certificate     /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # ---------- Basic-Auth ----------
    auth_basic           "Drizzle Studio";
    auth_basic_user_file /etc/nginx/.htpasswd;

    # DNS for local.drizzle.studio (used by proxy_pass)
    resolver 1.1.1.1 8.8.8.8 ipv6=off valid=60s;

    # ───────────────────────────────── UI ───────────────────────────

    # A)  root path — GET/HEAD redirect, everything else → back-end
    location = / {
        if ($request_method ~ ^(GET|HEAD)$) {
            return 302 /studio/?host=$host&port=443;
        }

        # POST /, OPTIONS / … → Drizzle back-end
        proxy_pass https://127.0.0.1:4983;
        include snippets/drizzle-proxy-headers.conf;
    }

    # B)  every /studio/** file → CDN
    location ^~ /studio/ {
        rewrite ^/studio/(.*)$ /$1 break;     # strip /studio/ prefix
        proxy_pass https://local.drizzle.studio;
        include snippets/drizzle-cdn-headers.conf;
    }

    # C)  Cloudflare RUM ping
    location ^~ /cdn-cgi/ {
        proxy_pass https://local.drizzle.studio;
        include snippets/drizzle-cdn-headers.conf;
    }

    # ───────────────────────────── API ─────────────────────────────
    # everything NOT /studio/** and NOT /cdn-cgi/**  → back-end
    location / {
        proxy_pass https://127.0.0.1:4983;
        include snippets/drizzle-proxy-headers.conf;
    }
}
```

### 4.2 CDN Headers: `/etc/nginx/snippets/drizzle-cdn-headers.conf`

```nginx
proxy_ssl_verify       off;
proxy_ssl_server_name  on;
proxy_ssl_name         local.drizzle.studio;
proxy_set_header Host  local.drizzle.studio;
```

### 4.3 Backend Headers: `/etc/nginx/snippets/drizzle-proxy-headers.conf`

```nginx
proxy_ssl_verify       off;
proxy_ssl_server_name  on;
proxy_ssl_name         local.drizzle.studio;

proxy_http_version     1.1;
proxy_set_header Host              local.drizzle.studio;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header Upgrade           $http_upgrade;
proxy_set_header Connection        "upgrade";
```

### 4.4 Enable the Site

```bash
sudo ln -s /etc/nginx/sites-available/drizzle.conf /etc/nginx/sites-enabled/
sudo nginx -t && sudo systemctl reload nginx
```

---

## 5 · Test in Browser

1. Open an **incognito tab**
2. Visit `https://example.com`
3. Enter Basic Auth credentials
4. You should be redirected to:
   `https://example.com/studio/?host=example.com&port=443`
5. The Studio UI should appear

---

## 6 · Troubleshooting

| Problem                      | Fix                                                               |
| ---------------------------- | ----------------------------------------------------------------- |
| `502 Bad Gateway`            | Drizzle Studio not running, wrong port, or missing `proxy_ssl_*`  |
| Redirect loop                | Only GET/HEAD on `/` should redirect. Others must proxy.          |
| MIME error for JS            | Ensure `/studio/**` proxies to CDN, not redirected                |
| Requests to `localhost:4983` | Clear browser local storage, ensure URL has `?host=example.com` |

---

## 7 · Security Tips

* Only expose Studio with at least Basic Auth or IP allow-list
* Keep port 4983 bound to 127.0.0.1
* Certbot auto-renews your certs via systemd timer

---

## ✅ Done!

Your Drizzle Studio OSS is now available at:

```
https://example.com/studio/?host=example.com&port=443
```
