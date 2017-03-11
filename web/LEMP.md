## Установка LEMP и не только

/var/www/html - тут поднимаем сайт на LEMP
/var/www/ghost - тут поднимаем блог на ghost
Сайт - твой домен или ip-адрес
Юзер - пользователь с sudo
db - база
userdb - пользователь базы данных
pwddb - пароль базы данных

* LEMP
    * [NGINX](#nginx) + [Let's Encrypt](#letsencrypt)
    * [MySQL](#mysql)
    * [PHP-FPM](#php-fpm)

* CMS
    * [x] [OpenCart](#opencart) *не заходит в админку*
    * [ ] [Joomla](#joomla) *не работает*

    * [x] [Ghost](#ghost)
    * [x] [WordPress](#wordpress)
    * [ ] [MODX](#modx) *не работает*

* Miscellaneous
    * [Pictures optimization](#pictures)

## NGINX
<a id="nginx"></a>

По установке и настройке nginx на RHEL/CentOS см. http://idoka.ru/nginx-tips-and-tricks/

```bash
sudo apt install -y nginx letsencrypt
sudo cp /etc/nginx/sites-available/default /etc/nginx/sites-available/default.bak
sudo sed -i "s/server_name _;/server_name САЙТ;/" /etc/nginx/sites-available/default
sudo sed -i "s/# server_names_hash_bucket_size 64;/server_names_hash_bucket_size 64;/" /etc/nginx/nginx.conf
sudo sed -i "s/# server_tokens off;/server_tokens off;/" /etc/nginx/nginx.conf
sudo sed -i 's/# gzip/gzip/g' /etc/nginx/nginx.conf
sudo sed -i 's/gzip_comp_level 6;/gzip_comp_level 1;/' /etc/nginx/nginx.conf
sudo systemctl reload nginx
```
## Let's Encrypt
<a id="letsencrypt"></a>

```bash
sudo mkdir -p /var/www/html/.well-known/
sudo nano /etc/nginx/sites-available/default
```
Добавляем

```nginx
	location ~ /.well-known {
		allow all;
	}
```
```bash
sudo nginx -t
sudo systemctl restart nginx
sudo letsencrypt certonly -a webroot --webroot-path=/var/www/html/ -d САЙТ
sudo openssl dhparam -out /etc/ssl/certs/dhparam.pem 2048
printf "ssl_certificate /etc/letsencrypt/live/САЙТ/fullchain.pem;\nssl_certificate_key /etc/letsencrypt/live/САЙТ/privkey.pem;" | sudo tee -a /etc/nginx/snippets/ssl-САЙТ.conf
sudo nano /etc/nginx/snippets/ssl-params.conf
```
Вставляем

```
ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
ssl_prefer_server_ciphers on;
ssl_ciphers "EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH";
ssl_ecdh_curve secp384r1;
ssl_session_cache shared:SSL:10m;
ssl_session_tickets off;
ssl_stapling on;
ssl_stapling_verify on;
resolver 8.8.8.8 8.8.4.4 valid=300s;
resolver_timeout 5s;
add_header Strict-Transport-Security "max-age=63072000; includeSubdomains";
add_header X-Frame-Options DENY;
add_header X-Content-Type-Options nosniff;
ssl_dhparam /etc/ssl/certs/dhparam.pem;
```
```bash
sudo nano /etc/nginx/sites-available/default
```
```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name САЙТ;
	return 301 https://$server_name$request_uri;
}
server {
	listen 443 ssl http2 default_server;
	listen [::]:443 ssl http2 default_server;
	include snippets/ssl-САЙТ.conf;
	include snippets/ssl-params.conf;
	root /var/www/html;
	index index.html index.htm index.nginx-debian.html;
	location ~ /.well-known {
		allow all;
	}
}
```
```bash
sudo nginx -t
sudo systemctl restart nginx
```
## Ghost
<a id="ghost"></a>

### Node.js
```bash
sudo apt install -y nodejs npm
```
Установка Node.js 6+ (если требуется)

```bash
curl -sL https://deb.nodesource.com/setup_6.x | sudo -E bash -
sudo apt install -y nodejs
```
```bash
cd /tmp
sudo apt install imagemagick php7.0-curl php7.0-gd php7.0-mbstring php7.0-mysql libapache2-mod-php7.0 php7.0-mcrypt
wget https://ghost.org/zip/ghost-latest.zip
sudo mkdir -p /var/www/ghost
sudo unzip -d /var/www/ghost ghost-latest.zip
cd /var/www/ghost
sudo npm install --production
sudo mv config.example.js config.js
sudo sed -i "s/http://my-ghost-blog.com/https://САЙТ/" /var/www/ghost/config.js
sudo rm /etc/nginx/sites-enabled/default
sudo touch /etc/nginx/sites-available/ghost
sudo nano /etc/nginx/sites-available/ghost
```
```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name САЙТ;
	return 301 https://$server_name$request_uri;
}
server {
	listen 443 ssl http2 default_server;
	listen [::]:443 ssl http2 default_server;
	include snippets/ssl-САЙТ.conf;
	include snippets/ssl-params.conf;
	root /var/www/ghost;
    index index.js;
    location / {
        proxy_set_header   X-Real-IP $remote_addr;
        proxy_set_header   Host      $http_host;
        proxy_pass         http://127.0.0.1:2368;
    }
	location ~ /.well-known {
		allow all;
	}
}
```
```bash
sudo ln -s /etc/nginx/sites-available/ghost /etc/nginx/sites-enabled/ghost
sudo adduser --no-create-home --gecos 'Ghost app' ghost
sudo chown -R ghost:ghost /var/www/ghost/
su - ghost
cd /var/www/ghost
npm start –-production
```
[https://САЙТ/ghost/setup](https://САЙТ/ghost/setup)

    !Нужно добавить автозагрузку

## MySQL
<a id="mysql"></a>

```bash
sudo apt install -y mariadb-server
sudo systemctl start mysql
sudo mysql_secure_installation
sudo mysql -u root -p
```
```sql
create database db;
create user 'userdb'@'localhost' identified by 'pwddb';
grant all privileges on db.* to 'userdb'@'localhost' identified by 'pwddb';
flush privileges;
\q
```
Для удаления

```sql
drop database db;
drop user 'userdb'@'localhost';
```

## PHP-FPM
<a id="php-fpm"></a>

```bash
sudo apt install -y php-fpm php-mysql php-curl php-gd php-mbstring php-mcrypt php-xml php-xmlrpc php-zip
sudo sed -i "s/;cgi.fix_pathinfo=1/cgi.fix_pathinfo=0/" /etc/php/7.0/fpm/php.ini
sudo sed -i "s/memory_limit = .*/memory_limit = 128MB/" /etc/php/7.0/fpm/php.ini
sudo sed -i "s/upload_max_filesize = .*/upload_max_filesize = 10M/" /etc/php/7.0/fpm/php.ini
sudo sed -i "s/post_max_size = .*/post_max_size = 12M/" /etc/php/7.0/fpm/php.ini
sudo sed -i "s/allow_url_fopen = On/allow_url_fopen = Off/" /etc/php/7.0/fpm/php.ini
sudo nano /etc/nginx/sites-available/default
```
```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name САЙТ;
	return 301 https://$server_name$request_uri;
}
server {
	listen 443 ssl http2 default_server;
	listen [::]:443 ssl http2 default_server;
	include snippets/ssl-САЙТ.conf;
	include snippets/ssl-params.conf;
	root /var/www/html;
	index index.php index.html index.htm index.nginx-debian.html;
    location / {
        try_files $uri $uri/ =404;
    }
	location ~ \.php$ {
	include snippets/fastcgi-php.conf;
	fastcgi_pass unix:/run/php/php7.0-fpm.sock;
	fastcgi_cache_lock on;
	fastcgi_cache_lock_timeout 6s;
	}
    location ~ /\.ht {
        deny all;
    }
    location ~ /.well-known {
		allow all;
	}
}
```
```bash
sudo nginx -t
sudo systemctl restart php7.0-fpm nginx
```

## MODX
<a id="modx"></a>

```bash
cd /tmp
wget https://modx.s3.amazonaws.com/releases/2.5.5/modx-2.5.5-pl.zip
unzip modx*.zip
mv modx*/ modx/
sudo cp -a /tmp/modx/. /var/www/html
sudo touch /var/www/html/core/config/config.inc.php
```
	!printf "link_tag_scheme — 1\nserver_protocol — https" | sudo tee -a /var/www/html/core/config/config.inc.php
```bash
sudo chown -R www-data:www-data /var/www/html
sudo chmod -R 775 /var/www/html/core/cache
sudo chmod -R 775 /var/www/html/core/config
sudo chmod -R 775 /var/www/html/core/export
sudo chmod -R 775 /var/www/html/core/import
sudo chmod -R 775 /var/www/html/core/packages
```

## OpenCart
<a id="opencart"></a>

```bash
cd /tmp
wget --content-disposition http://opencart-russia.ru/down.php?opencart2.3.0.2
unzip opencart*.zip
mv upload-*/ opencart/
mv opencart/config-dist.php opencart/config.php
mv opencart/admin/config-dist.php opencart/admin/config.php
sudo cp -a /tmp/opencart/. /var/www/html
sudo systemctl restart nginx php7.0-fpm
```
[САЙТ](https://САЙТ)

```bash
sudo rm -rf /var/www/html/install/
```
```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name САЙТ;
	return 301 https://$server_name$request_uri;
}
server {
	listen 443 ssl http2 default_server;
	listen [::]:443 ssl http2 default_server;
	include snippets/ssl-САЙТ.conf;
	include snippets/ssl-params.conf;
	root /var/www/html;
	index index.php;
	location ~ /.well-known {
		allow all;
	}
    location ~ \.php$ {
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/run/php/php7.0-fpm.sock;
		fastcgi_cache_lock on;
		fastcgi_cache_lock_timeout 6s;
	}
	location ~ /\.ht {
		deny all;
	}
	location ~ /\. {
		deny all;
		access_log off;
		log_not_found off;
    }
	location /image/data {
        autoindex on;
    }
    location /admin {
        index index.php;
    }
    location / {
        try_files $uri @opencart;
    }
    location @opencart {
        rewrite ^/(.+)$ /index.php?_route_=$1 last;
    }
    location = /favicon.ico {
    log_not_found off; access_log off;
    }
    location = /robots.txt {
    log_not_found off; access_log off; allow all;
    }
    location ~* \.(engine|inc|info|install|make|module|profile|test|po|sh|.*sql|theme|tpl(\.php)?|xtmpl)$|^(\..*|Entries.*|Repository|Root|Tag|Template)$|\.php_ {
        deny all;
    }
    location ~*  \.(jpg|jpeg|png|gif|css|js|ico)$ {
        expires max;
        log_not_found off;
    }
}
```

## Joomla
<a id="joomla"></a>

```bash
cd /tmp
wget https://downloads.joomla.org/ru/cms/joomla3/3-6-5/joomla_3-6-5-stable-full_package-zip?format=zip
sudo unzip joomla* -d /var/www/html/
sudo systemctl restart nginx php7.0-fpm
```

## WordPress
<a id="wordpress"></a>

```bash
cd /tmp
curl -O https://wordpress.org/latest.tar.gz
tar xzvf latest.tar.gz
cp /tmp/wordpress/wp-config-sample.php /tmp/wordpress/wp-config.php
mkdir /tmp/wordpress/wp-content/upgrade
sudo cp -a /tmp/wordpress/. /var/www/html
sudo chown -R Юзер:www-data /var/www/html
sudo find /var/www/html -type d -exec chmod g+s {} \;
sudo chmod g+w /var/www/html/wp-content
sudo chmod -R g+w /var/www/html/wp-content/themes
sudo chmod -R g+w /var/www/html/wp-content/plugins
curl -o salt.txt https://api.wordpress.org/secret-key/1.1/salt/
sudo cat salt.txt >> /var/www/html/wp-config.php
rm salt.txt
sudo sed -i 's/database_name_here/db/' /var/www/html/wp-config.php
sudo sed -i 's/username_here/userdb/' /var/www/html/wp-config.php
sudo sed -i 's/password_here/pwddb/' /var/www/html/wp-config.php
sudo echo "define('FS_METHOD', 'direct');" >> /var/www/html/wp-config.php
```
```nginx
server {
	listen 80 default_server;
	listen [::]:80 default_server;
	server_name САЙТ;
	return 301 https://$server_name$request_uri;
}
server {
	listen 443 ssl http2 default_server;
	listen [::]:443 ssl http2 default_server;
	include snippets/ssl-САЙТ.conf;
	include snippets/ssl-params.conf;
	root /var/www/html;
	index index.php index.html index.htm index.nginx-debian.html;
    location / {
        #try_files $uri $uri/ =404;
        try_files $uri $uri/ /index.php$is_args$args;
    }
	location ~ \.php$ {
	include snippets/fastcgi-php.conf;
	fastcgi_pass unix:/run/php/php7.0-fpm.sock;
	fastcgi_cache_lock on;
	fastcgi_cache_lock_timeout 6s;
	}
    location = /favicon.ico { log_not_found off; access_log off; }
    location = /robots.txt { log_not_found off; access_log off; allow all; }
    location ~* \.(css|gif|ico|jpeg|jpg|js|png)$ {
        expires max;
        log_not_found off;
    }
    location ~ /\.ht {
        deny all;
    }
    location ~ /.well-known {
		allow all;
	}
}
```
	!updates
```bash
sudo chown -R www-data /var/www/html
sudo chown -R Юзер /var/www/html
```

### Pictures optimization

```bash
sudo apt install -y jpegoptim optipng
cd /var/www/html/
find -name *.jpg -exec jpegoptim --strip-all '{}' \;
find -name *.png -exec optipng -o3 '{}' \;
```
