# Nextcloud

Easily deploy & configure NextCloud!


## 1. Configuration
Replace `mydomain.com` with your actual domain name:
```bash
sed -i 's/$DOMAIN/mydomain.com/g' docker-compose.yml
```

Replace database passwords with your own:
```bash
sed -i 's/$ROOT_MYSQL_PASWORD/my-mysql-root-pass/g' docker-compose.yml
sed -i 's/$MYSQL_PASSWORD/my-mysql-pass/g' docker-compose.yml
```
*I personally find that the `uuid` utility is quite nice for creating strong passwords.*

## 2. Setup a reverse-proxy

The goal is to forward `localhost:4999` -> `localhost:80`

An example NGINX proxy with HTTPS in `/etc/nginx/sites-available/nextcloud`

This config also assumes you have already LetsEncrypt certificates setup. Make adjustments if needed
```nginx
server {
    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/$DOMAIN/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/$DOMAIN/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;

    server_name cloud.$DOMAIN www.cloud.$DOMAIN;

    client_max_body_size 2048M;

    location / {
        proxy_pass http://localhost:4999;
        proxy_set_header Host $host;
        proxy_set_header X-Real-Ip $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_redirect off;
    }

}
server {
    if ($host = cloud.$DOMAIN) {
        return 301 https://$host$request_uri;
    }


    listen 80;
    server_name cloud.$DOMAIN www.cloud.$DOMAIN;
    return 404;
```

Where `mydomain.com` is your website, replace it with:

```bash
sed -i 's/$DOMAIN/mydomain.com/g' /etc/nginx/sites-available/nextcloud
```

Because this setup uses the older `sites-available` and `sites-enabled` nginx folder structure, add a symlink:
```bash
ln -s /etc/nginx/sites-available/nextcloud /etc/nginx/sites-enabled/
```

This config also set `client_max_body_size 2048M;` as the largest request size allowed to nginx, otherwise Nextcloud would not even let you change eg. the wallpaper due to size restrictions

### Test the config

```bash
sudo nginx -t
sudo systemctl reload nginx.service
```

## 3. Finish

Start it up
```bash
docker-compose up --build -d
```
Your Nextcloud instance will be available at `cloud.mydomain.com`

On the setup page, select MySQL and use the following:
```bash
DB User:        nextcloud
DB Password:    # $MYSQL_PASSWORD from step 1.
DB Name:        nextcloud_db
Host:           nextcloud
```
🎉 Set up a user & Done!

---

## Post-setup recommendations

I found that my emails did not load with a full height, these configuration lines in `./data/config/config.php` fixed it:
```php
  'overwrite.cli.url' => 'https://cloud.mydomain.com',
  'overwritehost' => 'cloud.mydomain.com',
  'overwriteprotocol' => 'https',
```


