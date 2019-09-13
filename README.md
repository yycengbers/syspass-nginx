> Just a note here: This configuration may need to be tweaked to allow older browsers and devices to access the instance.
> The devices accessing our instances are up-to-date and running the same OS and browser, so I locked the server down as much as possible.
> The links in my sources doc should help find the additional settings to open security up a bit.
> Just keep in mind, less is more.  The more you open up, the more vulnerable your server can become.
> Also note, I'm not a \*nix pro or an nginx pro...I just did a whole lot of reading and applied what I thought was best for us.  YMMV.

# syspass-nginx
Installation instructions for [sysPass](https://www.syspass.org/) on NGINX

[https://www.syspass.org/](https://www.syspass.org/)

Gain console or ssh access (keep ssh limited to vpn or source IP)
This document assumes the firewall is not open to public.  If it is, configure and enable fail2ban immediately after installing.

Install aptitude
```
sudo apt-get update
sudo apt-get install aptitude
sudo aptitude update
sudo aptitude upgrade
```


Install fail2ban and enable firewall (ufw already installed on 18.04)
```
sudo aptitude install fail2ban ufw
```

Set timezone to UTC
```
sudo ln -sf /usr/share/zoneinfo/UTC /etc/localtime
```

Install memcached
```
sudo aptitude install memcached
sudo sed -i 's/-l 127.0.0.1/-l 0.0.0.0/' /etc/memcached.conf
sudo systemctl restart memcached
```

Install nginx and php-fpm
```
sudo aptitude install nginx php-fpm
```

Tweak nginx and php-fpm settings
```
sudo sed -i "s/error_reporting = .*/error_reporting = E_ALL \& ~E_NOTICE \& ~E_STRICT \& ~E_DEPRECATED/" /etc/php/7.2/fpm/php.ini
sudo sed -i "s/display_errors = .*/display_errors = Off/" /etc/php/7.2/fpm/php.ini
sudo sed -i "s/memory_limit = .*/memory_limit = 512M/" /etc/php/7.2/fpm/php.ini
sudo sed -i "s/upload_max_filesize = .*/upload_max_filesize = 256M/" /etc/php/7.2/fpm/php.ini
sudo sed -i "s/post_max_size = .*/post_max_size = 256M/" /etc/php/7.2/fpm/php.ini
sudo sed -i "s/;date.timezone.*/date.timezone = UTC/" /etc/php/7.2/fpm/php.ini

sudo sed -i "s/;listen\.mode.*/listen.mode = 0666/" /etc/php/7.2/fpm/pool.d/www.conf
sudo sed -i "s/;request_terminate_timeout.*/request_terminate_timeout = 60/" /etc/php/7.2/fpm/pool.d/www.conf
sudo sed -i "s/pm\.max_children.*/pm.max_children = 70/" /etc/php/7.2/fpm/pool.d/www.conf
sudo sed -i "s/pm\.start_servers.*/pm.start_servers = 20/" /etc/php/7.2/fpm/pool.d/www.conf
sudo sed -i "s/pm\.min_spare_servers.*/pm.min_spare_servers = 20/" /etc/php/7.2/fpm/pool.d/www.conf
sudo sed -i "s/pm\.max_spare_servers.*/pm.max_spare_servers = 35/" /etc/php/7.2/fpm/pool.d/www.conf
sudo sed -i "s/;pm\.max_requests.*/pm.max_requests = 500/" /etc/php/7.2/fpm/pool.d/www.conf

sudo sed -i "s/worker_processes.*/worker_processes auto;/" /etc/nginx/nginx.conf
sudo sed -i "s/# multi_accept.*/multi_accept on;/" /etc/nginx/nginx.conf
sudo sed -i "s/# server_names_hash_bucket_size.*/server_names_hash_bucket_size 128;/" /etc/nginx/nginx.conf
sudo sed -i "s/# server_tokens off/server_tokens off/" /etc/nginx/nginx.conf
```

Increase nginx ssl security (this will take a long time)
```
sudo openssl dhparam -out /etc/nginx/dhparams.pem 4096
sudo ufw allow 'Nginx Full'
```

Install php modules
```
sudo aptitude install php-pear php-cgi php-gd php-json php-mysql php-readline php-curl php-intl php-xml php-mbstring php-xdebug
```

Install zip
```
sudo aptitude install zip
```

Install and secure MariaDB
```
sudo aptitude install mariadb-server
sudo mysql_secure_installation
```

Setup user for sysPass DB (replace 'admin and 'password' with unique values)
```
CREATE USER 'admin'@'localhost' IDENTIFIED BY 'password';
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'localhost' WITH GRANT OPTION;
FLUSH PRIVILEGES;
exit
```

Install letsencrypt certbot (need domain, http, https)
```
sudo aptitude install certbot python-certbot-nginx
```

Setup hostname for ssl cert and enable ssl
```
sudo sed -i "s/server_name _;/server_name syspass.example.com;/" /etc/nginx/sites-available/default
sudo certbot --nginx -d syspass.example.com  #choose "No redirect"
sudo certbot renew --dry-run  #test cert auto-renewal
```

Additional params
```
sudo nano /etc/nginx/snippets/fastcgi-php.conf    #Add to end of file
  fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
  fstcgi_buffers 256 4k;
  fastcgi_intercept_errors on;
  fastcgi_read_timeout 14400;
  include fastcgi_params;

###### Don't do this in production #####
#sudo nano /etc/php/7.2/mods-available/xdebug.ini    #Edit file to look like this
#	;zend_extension=xdebug.so
#	zend_extension=/usr/lib/php/20170718/xdebug.so
###### Don't do this in production #####

sudo nano /etc/php/7.2/fpm/php.ini    #Uncommment and edit these lines
	cgi.fix_pathinfo=0
	session.cookie_secure = 1
```

Sample nginx.conf
```
	user www-data;
	worker_processes auto;
	pid /run/nginx.pid;
	include /etc/nginx/modules-enabled/*.conf;

	events {
	        worker_connections 768;
	        use epoll;
	        multi_accept on;
	}

	http {

	        ##
	        # Basic Settings
	        ##

	        index index.php;

	        sendfile on;
	        tcp_nopush on;
	        tcp_nodelay on;
	        keepalive_timeout 65;
	        types_hash_max_size 2048;
	        server_tokens off;
	        disable_symlinks off;
	        aio threads;

	        server_names_hash_bucket_size 128;
	        # server_name_in_redirect off;

	        include /etc/nginx/mime.types;
	        default_type application/octet-stream;

	        add_header X-Frame-Options "SAMEORIGIN";
	        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

	        ##
	        # Logging Settings
	        ##

	        access_log /var/log/nginx/access.log;
	        error_log /var/log/nginx/error.log;

	        ##
	        # Virtual Host Configs
	        ##

	        include /etc/nginx/conf.d/*.conf;
	        include /etc/nginx/sites-enabled/*;
	}
```

Sample server block (/etc/nginx/sites-available/default)
```
server {

        listen 443 ssl http2 default_server;
        listen [::]:443 ssl http2 ipv6only=on default_server;

        ssl on;

        root /var/www/html;
        client_max_body_size 256M;

        index index.php;
        server_name syspass.example.com;

        include snippets/fastcgi-php.conf;

        add_header X-XSS-Protection "1; mode=block";
        proxy_cookie_path / "/; HTTPOnly; Secure";

        location = /robots.txt {
            return 200 "User-agent: *\nDisallow: /\n";
        }

        location ~* \.(jpg|jpeg|gif|css|png|js|map|woff|woff2|ttf|svg|eot)$ {
            expires 30d;
            access_log off;
        }

        location = /favicon.ico {
            try_files /favicon.ico =204;
        }

        location ~* ^/(?:COPYING|README|LICENSE[^.]*|LEGALNOTICE)(?:\.txt)*$ {
            return 404;
        }

        location ~ ^/(lib|schemas|vendor|\.rnd)/ {
            deny all;
        }

        location ~ /\. {
            deny all;
        }

        if ($request_method !~ ^(GET|HEAD|POST)$ ) {
            return 405;
        }

        location / {
            try_files $uri $uri/ =404;
        }

        location ~ \.php$ {
            fastcgi_pass unix:/var/run/php/php7.2-fpm.sock;
        }

    ssl_certificate /etc/letsencrypt/live/syspass.example.com/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/syspass.example.com/privkey.pem; # managed by Certbot

        ssl_protocols TLSv1.3 TLSv1.2;
        ssl_dhparam /etc/nginx/dhparams.pem;
        ssl_prefer_server_ciphers   on;
        ssl_ciphers 'ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:ECDH$
}
```

Check everything over and restart services
```
sudo nginx -t
sudo systemctl restart nginx
sudo systemctl restart php7.2-fpm
sudo systemctl restart mariadb
```

Clone sysPass git repository
```
cd /var/www/html
sudo rm index.nginx-debian.html
sudo git clone https://github.com/nuxsmin/sysPass.git .
```

Set owner on RW dirs (note - nginx doesn't require user/group ownership like apache)
```
sudo chown -R www-data ./app/backup/ ./app/cache/ ./app/config/ ./app/resources/ ./app/temp/
sudo chmod 750 ./backup/ ./cache/ ./config/ ./resources/ ./temp/
```

Install sysPass
```
cd /var/www/html/syspass
sudo php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
### Go to https://getcomposer.org/download/ to find the latest version of this command - the hash will update by version
sudo php -r "if (hash_file('sha384', 'composer-setup.php') === '93b54496392c062774670ac18b134c3b3a95e5a5e5c8f1a9f115f203b75bf9a129d5daa8ba6a13e2cc8a1da0806388a8') { echo 'Installer verified'; } else { echo 'Installer corrupt'; unlink('composer-setup.php'); } echo PHP_EOL;"
sudo php composer-setup.php
sudo php -r "unlink('composer-setup.php');"
php composer.phar install --no-dev
```

Setup sysPass
	https://syspass.example.com
	For mysql creds use the user/pass you configured earlier in the mariadb config

Don't forget to back everything up.
	https://tecadmin.net/bash-script-mysql-database-backup/
