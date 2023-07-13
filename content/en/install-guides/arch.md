+++
title = "Deploying Pixelfed on Arch Linux"
summary = "End-to-end guide for installing Pixelfed on Arch Linux"
[menu]
[menu.docs]
identifier = "install-guides/arch"
parent = "install-guides"
name = "Arch Linux"is 
+++

## Assumptions
These instructions will install Pixelfed with the following:
- Nginx (instead of Apache)
- MariaDB (instead of PostgreSQL)
- PHP-FPM (latest version)
- Redis and PHP-FPM running via sockets instead of TCP (same machine)
- `pixelfed` user for running Horizon queues, `http` user for running web processes (Arch default)
- Repo cloned at `/srv/http/pixelfed`
- No other sites/services running on this machine

## Preparing a machine

You will need a machine running Arch Linux with access to the root account.

1. Login as `root`.
2. Create the `pixelfed` user and group:
```bash
useradd -rU -s /bin/bash -d /srv/http/pixelfed pixelfed
```
3. Install dependencies:
```bash
pacman -S --needed nginx mariadb redis git php-fpm php-intl php-gd php-imagick php-redis composer jpegoptim optipng pngquant imagemagick ffmpeg unzip certbot certbot-nginx
```
4. Setup database. During `mysql_secure_installation`, hit Enter to use the default options. Make sure to set a password for the SQL user `root` (as by default, there is no password).
```bash
mysql_install_db --user=mysql --basedir=/usr --datadir=/var/lib/mysql
systemctl enable --now mariadb
mysql_secure_installation
mysql -u root -p
```
```sql
create database pixelfed;
grant all privileges on pixelfed.* to 'pixelfed'@'localhost' identified by 'strong_password';
flush privileges;
exit
```
5. Edit `/etc/php/php.ini` and uncomment the following lines:
```
extension=bcmath
extension=exif
extension=gd
extension=iconv
extension=intl
extension=mysqli
extension=pdo_mysql
```
Edit the following lines to your desired upload limits:
```
post_max_size = 8M
upload_max_filesize = 2M
max_file_uploads = 20
```
Edit `/etc/php/conf.d/imagick.ini` and uncomment:
```
extension=imagick
```
Edit `/etc/php/conf.d/redis.ini` and uncomment:
```
extension=redis
```
Edit `/etc/php/conf.d/igbinary.ini` and uncomment:
```
extension=igbinary
```
Create a PHP-FPM pool for Pixelfed:
```bash
cd /etc/php/php-fpm.d/
cp www.conf pixelfed.conf
$EDITOR pixelfed.conf
```
Make the following changes to the PHP-FPM pool:
```
;     use the username of the app-user as the pool name, e.g. pixelfed
[pixelfed]
user = pixelfed
group = pixelfed
;    to use a tcp socket, e.g. if running php-fpm on a different machine than your app:
;    (note that the port 9001 is used, since php-fpm defaults to running on port 9000;)
;    (however, the port can be whatever you want)
; listen = 127.0.0.1:9001;
;    but it's better to use a socket if you're running locally on the same machine:
listen = /run/php-fpm/pixelfed.sock
listen.owner = http
listen.group = http
listen.mode = 0660
[...]
```
6. Edit `/etc/redis.conf` and edit the following lines:
```
port 6379                           # change this to "port 0" to disable network packets
unixsocket /run/redis/redis.sock    # 
unixsocketperm 770                  # give permission to "redis" user and group
```
7. Edit `/etc/nginx/nginx.conf`:
```nginx
worker_processes 1;    # change to auto, or 1 x your CPU cores, but 1 is enough
events {
    worker_connections 1024;    # 512-1024 is fine for a small site, but you may want to use up to 10k or more, if running in production with many users
}
http {
    # [...]
    client_max_body_size 9m; # add this line to configure client upload file size
    
    gzip on;    # uncomment this line
    server {    # delete this entire block
        # [...]
    }

    include /srv/http/pixelfed/nginx.conf;    # we will make this file later
}
```
Generate SSL cert:
```bash
mkdir /etc/nginx/ssl
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout /etc/nginx/ssl/server.key -out /etc/nginx/ssl/server.crt
```
8. Add users to groups:
```bash
usermod -aG redis pixelfed # give app user access to redis for queues
```
9. Enable services:
```bash
systemctl enable {nginx,redis,php-fpm}
systemctl start {redis,php-fpm} # nginx will fail if started now
```

## Pixelfed setup
1. Clone the repo:
```
cd /srv/http
git clone -b dev https://github.com/pixelfed/pixelfed.git pixelfed
```
2. Setup environment variables and nginx:
```bash
cd pixelfed
cp contrib/nginx.conf nginx.conf
$EDITOR nginx.conf
## in particular, set:
### - the correct domain name
### - client_max_body_size to your desired upload limit
### - fastcgi_pass correct path (e.g. unix:/run/php-fpm/pixelfed.sock;)

cp .env.example .env
$EDITOR .env
## in particular, set:
### DB_SOCKET = /run/mysqld/mysqld.sock
###
### REDIS_HOST = /run/redis/redis.sock
### REDIS_PORT = null
###
### IMAGE_DRIVER = imagick

$EDITOR config/database.php
## change predis to phpredis
```
3. Create the following file at `/etc/systemd/system/pixelfed.service`:
```
[Unit]
Description=Pixelfed task queueing via Laravel Horizon
After=network.target
Requires=mariadb
Requires=php-fpm
Requires=redis
Requires=nginx

[Service]
Type=simple
ExecStart=/usr/bin/php /srv/http/pixelfed/artisan horizon
User=pixelfed
Restart=on-failure

[Install]
WantedBy=multi-user.target
```
4. Set permissions:
```bash
chown -R pixelfed:pixelfed .
find . -type d -exec chmod 755 {} \;
find . -type f -exec chmod 644 {} \;
```
5. Switch to the `pixelfed` user:
```bash
su - pixelfed
```
6. Deploy:
```bash
$EDITOR composer.json
## change require php to >= instead of ^
## remove beyondcode/laravel-self-diagnosis
## change league/iso3166 to >= instead of ^
## change spatie/laravel-image-optimizer to >= instead of ^
## change laravel/ui to >= instead of ^
composer update
composer install --no-ansi --no-interaction --no-progress --no-scripts --optimize-autoloader
php artisan key:generate
php artisan storage:link
php artisan horizon:install
php artisan horizon:publish
php artisan migrate --force
```
Optionally, use cache [NOTE: if you run these commands, you will need to run them every time you change .env or update Pixelfed]:
```bash
php artisan config:cache
php artisan route:cache
php artisan view:cache
```
Import Places data:
```bash
php artisan import:cities
``` 
7. Start web server and Horizon task queue:
```bash
exit
systemctl enable --now {nginx,pixelfed}
```
