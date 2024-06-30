- 👋 Hi, I’m @ABISHEK-SUNIL

This Installation method is customised and installed according to my idea and its working
if your getting any error and trouble please make sure your insttalled according to my script
so if your having any trouble contact me on discord .
<!---
ABISHEK-SUNIL/ABISHEK-SUNIL is a ✨ special ✨ repository because its `README.md` (this file) appears on your GitHub profile.
You can click the Preview link to take a look at your changes.
--->
<!--Install Dependencies-->

       apt -y install software-properties-common curl apt-transport-https ca-certificates gnupg

       LC_ALL=C.UTF-8 add-apt-repository -y ppa:ondrej/php add-apt-repository ppa:redislabs/redis -y

       curl -sS https://downloads.mariadb.com/MariaDB/mariadb_repo_setup | sudo bash

       apt update

       apt -y install php8.1 php8.1-{cli,gd,mysql,pdo,mbstring,tokenizer,bcmath,xml,fpm,curl,zip} mariadb-server nginx tar unzip git redis-server

       curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer
       
# Create Directory
<!-- Downloading Files-->
       mkdir -p /var/www/jexactyl
       cd /var/www/jexactyl
       curl -Lo panel.tar.gz https://github.com/jexactyl/jexactyl/releases/latest/download/panel.tar.gz
       tar -xzvf panel.tar.gz
       chmod -R 755 storage/* bootstrap/cache/

# Database Setuping

       mysql -u root -p

# Remember to change 'yourPassword' below to be a unique password
       CREATE USER 'jexactyl'@'127.0.0.1' IDENTIFIED BY 'yourPassword';
       CREATE DATABASE panel;
       GRANT ALL PRIVILEGES ON panel.* TO 'jexactyl'@'127.0.0.1' WITH GRANT OPTION;
       exit

# Environment Setup

<!--Environment Setup-->
       cp .env.example .env

       composer install --no-dev --optimize-autoloader

       php artisan key:generate --force

       php artisan p:environment:setup
       php artisan p:environment:database
       php artisan p:environment:mail # Not required to run the Panel.

       php artisan migrate --seed --force

       php artisan p:user:make

# If using NGINX or Apache (not on CentOS):

       chown -R www-data:www-data /var/www/jexactyl/*

# If using NGINX on CentOS:
       chown -R nginx:nginx /var/www/jexactyl/*

# If using Apache on CentOS:
       chown -R apache:apache /var/www/jexactyl/*

# -Queue Workers
<!--Queue Workers-->
       * * * * * php /var/www/jexactyl/artisan schedule:run >> /dev/null 2>&1

# Now setup Queue do not miss this one Important
<!--Now setup Queue do not miss this one Importent-->

       [Unit]
       Description=Jexactyl Queue Worker

       [Service]
       User=www-data
       Group=www-data
       Restart=always
       ExecStart=/usr/bin/php /var/www/jexactyl/artisan queue:work --queue=high,standard,low --sleep=3 --tries=3
       StartLimitInterval=180
       StartLimitBurst=30
       RestartSec=5s

       [Install]
       WantedBy=multi-user.target

# Enable Queue Workers
<!--Next cmd for setuping-->
       sudo systemctl enable --now panel.service
       sudo systemctl enable --now redis-server

<!--Setup SSL with Certbot-->
# If using NGINX run the following:
       apt install -y certbot python3-certbot-nginx

# If you are using NGINX:
       certbot certonly --nginx -d <domain>

# Nginx with SSL Configuration
<!--Nginx with SSL Configuration-->

rm /etc/nginx/sites-available/default; rm /etc/nginx/sites-enabled/default

<!--Setting up Make a file called panel.conf in /etc/nginx/sites-available-->
       server {
       listen 80;
       server_name <domain>;
       return 301 https://$server_name$request_uri;
       }

server {
    listen 443 ssl http2;
    server_name <domain>;

    root /var/www/jexactyl/public;
    index index.php;

    access_log /var/log/nginx/jexactyl.app-access.log;
    error_log  /var/log/nginx/jexactyl.app-error.log error;

    # allow larger file uploads and longer script runtimes
    client_max_body_size 100m;
    client_body_timeout 120s;

    sendfile off;

    # SSL Configuration
    ssl_certificate /etc/letsencrypt/live/<domain>/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/<domain>/privkey.pem;
    ssl_session_cache shared:SSL:10m;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers "ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384:ECDHE-ECDSA-CHACHA20-POLY1305:ECDHE-RSA-CHACHA20-POLY1305:DHE-RSA-AES128-GCM-SHA256:DHE-RSA-AES256-GCM-SHA384";
    ssl_prefer_server_ciphers on;

    add_header X-Content-Type-Options nosniff;
    add_header X-XSS-Protection "1; mode=block";
    add_header X-Robots-Tag none;
    add_header Content-Security-Policy "frame-ancestors 'self'";
    add_header X-Frame-Options DENY;
    add_header Referrer-Policy same-origin;

    location / {
        try_files $uri $uri/ /index.php?$query_string;
    }

    location ~ \.php$ {
        fastcgi_split_path_info ^(.+\.php)(/.+)$;
        fastcgi_pass unix:/run/php/php8.1-fpm.sock;
        fastcgi_index index.php;
        include fastcgi_params;
        fastcgi_param PHP_VALUE "upload_max_filesize = 100M \n post_max_size=100M";
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        fastcgi_param HTTP_PROXY "";
        fastcgi_intercept_errors off;
        fastcgi_buffer_size 16k;
        fastcgi_buffers 4 16k;
        fastcgi_connect_timeout 300;
        fastcgi_send_timeout 300;
        fastcgi_read_timeout 300;
        include /etc/nginx/fastcgi_params;
    }

    location ~ /\.ht {
        deny all;
    }
}
////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////

ln -s /etc/nginx/sites-available/panel.conf /etc/nginx/sites-enabled/panel.conf

nginx -t

systemctl restart nginx
