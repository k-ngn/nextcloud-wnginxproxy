# Setup Nextcloud behind Nginx Reverse Proxy on Ubuntu 22.04

> **Installation on Ubuntu 22.04.03 LTS with Apache2, APCu, redis and mariadb behind a NGINX proxy, no Docker, no Snap**

## Nextcloud

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install unattended-upgrades -y
sudo dpkg-reconfigure unattended-upgrades
```

### Install packages
Nextcloud currently recommends PHP 8.2 but Ubuntu 22.04 comes with PHP8.1

```bash
sudo apt install apache2 mariadb-server exif imagemagick redis-server bzip2 -y
```

Install all php modules.
```bash
sudo apt install libapache2-mod-php php-gd php-posix php-mysql php-ctype php-curl php-mbstring php-gmp php-dom php-bcmath php-xml php-imagick php-zip php-bz2 php-intl php-imagick php-redis php-apcu -y
```
## MariaDB

Change the MariaDB settings to the recommended READ-COMITTED and binlog format ROW.
```bash
sudo nano /etc/mysql/conf.d/nextcloud.cnf
```
insert
```bash
[mysqld]
transaction_isolation = READ-COMMITTED
binlog_format = ROW 
```
exit and save.
Reload mariadb

```bash
sudo systemctl restart mariadb.service
```

Create the database
```bash
sudo mariadb
```
You should now see "`MariaDB [(none)]>`"

Check if the tx_isolation is "READ-COMITTED" and if binlog_format is "ROW".
```mysql
SELECT @@global.tx_isolation;
SELECT @@global.binlog_format;
```
If everything looks good, we can continue. 
Insert this to create a database called nextcloud. Replace all three x_ variables with your data.
```mysql
CREATE USER 'x_database_user'@'localhost' IDENTIFIED BY 'x_database_password';
CREATE DATABASE IF NOT EXISTS nextcloud CHARACTER SET utf8mb4 COLLATE utf8mb4_general_ci;
GRANT ALL PRIVILEGES ON nextcloud.* TO 'x_database_user'@'localhost';
FLUSH PRIVILEGES;
exit;
```
You should see 3 times a "Query OK" line and a "Bye" at the end.

Secure MariaDB
```bash
sudo mariadb-secure-installation
```

## Nextcloud
Download Nextcloud
```bash
wget https://download.nextcloud.com/server/releases/latest.tar.bz2
wget https://download.nextcloud.com/server/releases/latest.tar.bz2.asc
wget https://download.nextcloud.com/server/releases/latest.tar.bz2.md5
wget https://nextcloud.com/nextcloud.asc
gpg --import nextcloud.asc
```
verify
```bash
md5sum -c latest.tar.bz2.md5 < latest.tar.bz2
```
and
```bash
gpg --verify latest.tar.bz2.asc latest.tar.bz2
```
extract and move to the webroot. Change ownership and delete install files
```bash
tar -xjvf latest.tar.bz2
sudo cp -r nextcloud /var/www
sudo chown -R www-data:www-data /var/www/nextcloud
rm -r nextcloud
rm  latest.tar.bz2 latest.tar.bz2.asc  latest.tar.bz2.md5  nextcloud.asc
```
## PHP settings

We wanna change the PHP memory limit and upload filesize. Replace 8.1 if you have a newer version of PHP.
```bash
sudo nano /etc/php/8.1/apache2/php.ini
```

We search for these settings to change (use ctrl+W to search in nano).
```bash
memory_limit = 1G
upload_max_filesize = 50G
post_max_size = 0
max_execution_time = 3600
date.timezone = Europe/Amsterdam
```
Save and exit. Reload apache2
  
```bash
sudo systemctl reload apache2.service
```
create a PHP file so we can check the changes
```bash
sudo nano /var/www/html/phpinfo.php
```
and insert

```PHP
<?php phpinfo(); ?>
```

Now can see the default Apache2 page at 
http://x_nextcloud_host_IPv4.
To see the PHP settings go to 
http://x_nextcloud_host_IPv4/phphinfo.php
You should find the values we defined. 
To disable that page and delete the html folder, run 
```bash
sudo a2dissite 000-default.conf
sudo systemctl reload apache2
sudo rm -r /var/www/html
```
## Apache2
Create the data folder. You can also use a different location.
Just make sure to replace /var/www/nextcloud/data everywhere with your data path. 
```bash
sudo mkdir /var/www/nextcloud/data
sudo chown -R www-data:www-data /var/www/nextcloud/data
```

Configure Apache2
```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```
insert:
```bash
<VirtualHost *:80>
  DocumentRoot /var/www/nextcloud/
  ServerName  cloud.x_youromain.com

  <Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>
  </Directory>
</VirtualHost>
```

Enable site and mods:
```bash
sudo a2ensite nextcloud.conf
sudo a2enmod rewrite headers env dir mime
sudo systemctl reload apache2
```


## NGINX - The Reverse Proxy

#### Setting the nginx proxy on another server 
If you already have a Nginx Proxy running, start from [[#Already have running nginx proxy server]]
My Nginx Proxy also run on Ubuntu 22.04 server (Nginx version 1.18)

```bash
sudo apt update && sudo apt upgrade -y
sudo apt install nginx -y
```

#### Already have running nginx proxy server
This guide assumes you already have a NGINX reverse Proxy up and running. 
Required a domain name for this and you have Certbot installed. 
Create an emtpy site without ssl.
```bash
sudo nano /etc/nginx/sites-available/cloud.x_youromain.conf
```

```NGINX
server {
    listen      80;
    listen      [::]:80;
    server_name cloud.x_yourdomain.com;
}
```
```bash
sudo nginx -t
sudo ln -s /etc/nginx/sites-available/cloud.x_yourdomain.conf /etc/nginx/sites-enabled/cloud.x_yourdomain.conf
sudo nginx -s reload
```
Now we let certbot create a cert. For certbot to be sucessfull, you need an A or AAAA record that points to your proxy with the open port 80.
```bash
sudo certbot
```
Follow the certbot instructions.
This will create a cert and also change your config to redirect all traffic to https.
To test if the automatic removal is working run
```bash
sudo certbot renew --dry-run
```

```bash
sudo nano /etc/nginx/sites-available/cloud.x_youromain.conf
```
At the time of writing this, there is still an open issue for certbot (https://github.com/certbot/certbot/issues/3646). 
To not get any warnings from NGINX, add 'http2' in the two ssl listen lines.
```NGINX
server {
    server_name cloud.x_youromain.com;


    listen [::]:443 ssl http2 ipv6only=on; # managed by Certbot
    listen 443 ssl http2; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/cloud.x_youromain.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/cloud.x_youromain.com/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    # security headers
#    add_header X-Content-Type-Options    "nosniff" always;
#    add_header X-Robots-Tag              "noindex, nofollow" always;
    add_header Referrer-Policy           "no-referrer" always;
    add_header Content-Security-Policy   "default-src 'self' http: https: ws: wss: data: blob: 'unsafe-inline'; frame-ancestors 'self';" always;
    add_header Permissions-Policy        "interest-cohort=()" always;
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # logging
    access_log              /var/log/nginx/access.log combined buffer=512k flush=1m;
    error_log               /var/log/nginx/error.log warn;

    # reverse proxy
    location / {
        proxy_pass            http://x_nextcloud_host_IPv4/;
        proxy_set_header Host $host;

        proxy_http_version                 1.1;
        proxy_cache_bypass                 $http_upgrade;

        # Proxy SSL
        proxy_ssl_server_name              on;

        # Proxy headers
        proxy_set_header Upgrade           $http_upgrade;
        #proxy_set_header Connection        $connection_upgrade;
        proxy_set_header X-Real-IP         $remote_addr;
        #proxy_set_header Forwarded         $proxy_add_forwarded;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Host  $host;
        proxy_set_header X-Forwarded-Port  $server_port;

        # Proxy timeouts
        proxy_connect_timeout              600s;
        proxy_send_timeout                 600s;
        proxy_read_timeout                 600s;
    }

    location /.well-known/carddav {
    return 301 $scheme://$host/remote.php/dav;
    }

    location /.well-known/caldav {
    return 301 $scheme://$host/remote.php/dav;
    }

}


server {
    if ($host = cloud.x_youromain.com) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen      80;
    listen      [::]:80;
    server_name cloud.x_youromain.com;
    return 404; # managed by Certbot


}

```
Check your NGINX config and reload
```bash
sudo nginx -t
sudo nginx -s reload
```


## Install Nextcloud 
You can install Nextcloud with that command or by using the WebPage.

```bash
cd /var/www/nextcloud/
sudo -u www-data php occ  maintenance:install \
--database='mysql' --database-name='nextcloud' \
--database-user='x_database_user' --database-pass='x_database_password' \
--admin-user='x_nextcloud_admin_user' --admin-pass='x_nextcloud_admin_user_password' \
--data-dir='/var/www/nextcloud/data'
```

If we navigate now to https://cloud.x_youromain.com, we should see a warning that we are not trusted, because we have not set up the proxy configs yet.


## PHP config settings
Edit config.php file. 
```bash
sudo nano /var/www/nextcloud/config/config.php
```
Set the trusted_domains array
```bash
  0 => 'cloud.x_youromain.com',
```
also change or add these settings:
```bash
'trusted_proxies'   => ['x_NGINX_IPv4'],
'default_language' => 'de',
'default_locale' => 'de_DE',
'default_phone_region' => 'DE',
'overwrite.cli.url' => 'https://cloud.x_youromain.com',
'overwriteprotocol' => 'https',
'overwritewebroot' => '/',
'overwritecondaddr' => 'x_NGINX_IPv4',
'htaccess.RewriteBase' => '/',
```
save and exit

update the settings by
```bash
cd /var/www/nextcloud/
sudo -u www-data php occ maintenance:update:htaccess
```

## Set up crontab
We wanna use crontab instead of AJAX.
```bash
sudo crontab -u www-data -e
```
press 1 to use nano and insert at the end
```bash
*/5  *  *  *  * php -f /var/www/nextcloud/cron.php
```
Change the settings by
```bash
sudo -u www-data php /var/www/nextcloud/occ background:cron
```
## Mail
In the web GUI, go to user settings and insert the mail address for the x_nextcloud_admin_user you created earlier. 
Naviate to Administration -> Basic settings to set up outgoing mail. In my example, I am using Office365 with an AppPassword created in the account settings of MS365.
```config
Servername: smtp.office365.com
Port: 587
Encryption: STARTTLS
Needs authentification, sender and user is me@mydomain.com 
AppPasswort
```

## Caching
Check if Opcache is working
```bash
php -r 'phpinfo();' | grep opcache.enable
```
### Redis
Add redis to the www-data group
```bash
sudo usermod -a -G redis www-data
```
Configure Redis server
```bash
sudo nano /etc/redis/redis.conf
```
uncomment 
unixsocket  /var/run/redis/redis.sock
Set 
unixsocketperm to 770
Exit and save.
Restart redis
```bash
sudo service redis-server restart
```
Check output of redis
```bash
ls -lh /var/run/redis
```
Change nextcloud PHP config. While we are in this file, we also add the memcache.local for APCu.

```bash
sudo nano /var/www/nextcloud/config/config.php
```
Add:
```PHP
  'memcache.local' => '\OC\Memcache\APCu',
  'memcache.locking' => '\OC\Memcache\Redis',
  'redis' => array(
     'host' => 'localhost',
     'port' => 6379,
     'timeout' => 1,
     'password' => '',  
      ),
```

### APCu
Change apcu.ini. Watch out for the PHP version
```bash
sudo nano /etc/php/8.1/apache2/conf.d/20-apcu.ini
```
Change it to: 
```config
extension=apcu.so
apc.enabled=1
apc.enable_cli=1
```
To start APCu automatically use this command and replace the PHP version 8.1 if needed
```bash
sudo -u www-data php8.1 --define apc.enable_cli=1  /var/www/nextcloud/occ  maintenance:repair
```

## Configure Apache2 HSTS
Probably only cosmetics, because it is already done by the proxy. Anyway setting this will remove the warning in the admin center.
```bash
sudo nano /etc/apache2/sites-available/nextcloud.conf
```
insert mod_headers.c     

```bash
<VirtualHost *:80>
  DocumentRoot /var/www/nextcloud/
  ServerName  cloud.salzmann.solutions

  <Directory /var/www/nextcloud/>
    Require all granted
    AllowOverride All
    Options FollowSymLinks MultiViews

    <IfModule mod_dav.c>
      Dav off
    </IfModule>

    <IfModule mod_headers.c>
      Header always set Strict-Transport-Security "max-age=63072000; includeSubDomains; preload"
    </IfModule>

  </Directory>
</VirtualHost>
```
save and exit. Reload
```bash
sudo systemctl reload apache2
```

Congrats! You should no have no warnings in the admin center and a perfect score on scan.nextcloud.com. 
