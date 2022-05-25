# Тестовое приложение для разворачивания с помощью Shell Scripts 
[Репозиторий проекта](https://github.com/rotoro-cloud/Laravel-Real-Estate-Venue-Portal)

Эти команды для автоматического развертывания приложения на тестовом стенде `starbase` на базе CentOS 7.
Таким образом, можно их вставить в свой скрипт для работы лабораторной.

1. Установим MariaDB
    - Установим вспомогательные утилиты
      ```
      sudo yum install -y wget net-tools
      ```
    - Добавим репозиторий MariaDB
      ```
      wget https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
      chmod a+x mariadb_repo_setup 
      sudo ./mariadb_repo_setup --mariadb-server-version=mariadb-10.6
      ```
    - Установка самой MariaDB
      ```
      sudo yum install -y MariaDB-server
      ```
    - Запустим MariaDB и поставим в автозагрузку
      ```
      sudo systemctl enable --now mariadb
      ```
    - Настройка MariaDB (user "root" на "localhost")
      ```
      sudo mysql -sfu root <<EOS
      -- set root password
      ALTER USER 'root'@'localhost' IDENTIFIED BY 'my_strong_password';
      -- delete anonymous users
      DELETE FROM mysql.user WHERE User='';
      -- delete remote root capabilities
      DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
      -- drop database 'test'
      DROP DATABASE IF EXISTS test;
      -- create database 'laravel'
      CREATE DATABASE laravel;
      -- also make sure there are lingering permissions to it
      DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
      -- make changes immediately
      FLUSH PRIVILEGES;
      EOS

2. Установим Nginx 
    - Добавим репозиторий Nginx
      ```
      sudo tee -a /etc/yum.repos.d/nginx.repo >/dev/null <<'EOF'
      [nginx-stable]
      name=nginx stable repo
      baseurl=http://nginx.org/packages/centos/7/x86_64/
      gpgcheck=1
      enabled=0
      gpgkey=https://nginx.org/keys/nginx_signing.key
 
      [nginx-mainline]
      name=nginx mainline repo
      baseurl=http://nginx.org/packages/mainline/centos/7/x86_64/
      gpgcheck=1
      enabled=1
      gpgkey=https://nginx.org/keys/nginx_signing.key
      EOF
      ```
    - Включим репозиторий nginx-mainline
      ```
      sudo yum --enablerepo=nginx-mainline update
      ```
    - Установим Nginx и OpenSSL
      ```
      sudo yum -y install nginx openssl
      ```
    - Настроим Nginx
      ```
      sudo tee -a /etc/nginx/conf.d/default.conf >/dev/null <<'EOF'
      server {
          listen       80;
          server_name  *.environments.katacoda.com;

          root   /usr/share/nginx/html/public/public;
          index index.php index.html index.htm;

          location / {
              try_files \$uri \$uri/ /index.php\$is_args\$args;
          }
          error_page 404 /404.html;
          error_page 500 502 503 504 /50x.html;

          location = /50x.html {
              root /usr/share/nginx/html;
          }

          location ~ \.php$ {
              try_files \$uri =404;
              fastcgi_pass unix:/var/run/php-fpm/php-fpm.sock;
              fastcgi_index index.php;
              fastcgi_param SCRIPT_FILENAME \$document_root\$fastcgi_script_name;
              include fastcgi_params;
          }
      }
      EOF
      ```
    - Запустим Nginx и поставим в автозагрузку
      ```
      sudo systemctl enable --now nginx
      ```

3. Установим PHP-FPM 7.4
    - Установим репозитории EPEL и Remi, там свежий PHP
      ```
      sudo yum install -y yum-utils
      sudo yum localinstall -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
      sudo yum localinstall -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm
      sudo yum-config-manager --enable remi-php74
      sudo yum makecache fast
      ```
    - Установим PHP 7.4
      ```
      sudo yum install -y php-cli php-fpm php-common php-curl php-gd php-imap php-intl php-mbstring php-xml php-zip php-bz2 php-bcmath php-json php-opcache php-devel php-mysqlnd php-pear gcc ImageMagick ImageMagick-devel
      ```
    - Поправим php.ini под пользователя nginx
      ```
      sudo sed -i "s\^user = apache\user = nginx\g" /etc/php-fpm.d/www.conf
      sudo sed -i "s\^group = apache\group = nginx\g" /etc/php-fpm.d/www.conf
      sudo sed -i "s\^listen = 127.0.0.1:9000\listen = /var/run/php-fpm/php-fpm.sock\g" /etc/php-fpm.d/www.conf
      sudo sed -i "s\^;listen.owner = nobody\listen.owner = nginx\g" /etc/php-fpm.d/www.conf
      sudo sed -i "s\^;listen.group = nobody\listen.group = nginx\g" /etc/php-fpm.d/www.conf
      sudo sed -i "s\^;listen.mode = 0660\listen.mode = 0660\g" /etc/php-fpm.d/www.conf
      ```
    - Установим пакеты pear
      ```
      yes '' | sudo pecl install imagick
      ```
    - Настроим модуль PHP
      ```
      sudo bash -c 'echo "extension=imagick.so" > /etc/php.d/imagick.ini'
      ```
    - Запустим PHP-FPM и поставим в автозагрузку
      ```
      sudo systemctl enable --now php-fpm
      ```
     
4. Ставим приложение
    - Установим git
      ```
      sudo yum -y install git
      ```
    - Склонируем репо
      ```
      sudo git clone https://github.com/rotoro-cloud/Laravel-Real-Estate-Venue-Portal.git /usr/share/nginx/html/public
      ```
    - Перейдем в папку
      ```
      cd /usr/share/nginx/html/public
      ```
    - Создадим .env из .env.example
      ```
      sudo cp .env.example .env
      ```
    - Поменяем в нем параметр `DB_PASSWORD` на заданный ранее
      ```
      sudo sed -i "s/^DB_PASSWORD=/DB_PASSWORD=my_strong_password/g" .env
      ```
    - Установим `composer 2`
      ```
      sudo php -r "copy('https://getcomposer.org/installer', 'composer-setup.php');"
      sudo php composer-setup.php --install-dir=/usr/local/bin --filename=composer
      sudo php -r "unlink('composer-setup.php');"
      ```
    - Подтянем зависимости проекта 
      ```
      sudo /usr/local/bin/composer update
      ```
    - Выполним создание ключа приложения 
      ```
      sudo php artisan key:generate;
      ```
    - Наполним базу тестовой информацией
      ```
      sudo php artisan migrate --seed;
      ```
    - Создадим символические ссылки для хранилища 
      ```
      sudo rm -rf public/storage; 
      sudo php artisan storage:link;
      sed -i "s\'url' => '/storage',\'url' => env('APP_URL').'/storage',\g" /usr/share/nginx/html/public/config/filesystems.php
      ```
    - Дадим доступ нужным директориям
      ```
      sudo chown -R nginx.nginx /usr/share/nginx/html/;
      sudo mkdir -p /usr/share/nginx/html/storage/logs
      sudo mkdir -p /usr/share/nginx/html/bootstrap/cache;
      sudo chmod -R ug+rwx /usr/share/nginx/html/storage /usr/share/nginx/html/bootstrap/cache;
      sudo chmod -R o+rwx /usr/share/nginx/html/storage/logs;
      ```
      
5. Ставим файрволл
    - Установим firewalld
      ```
      sudo yum -y install firewalld
      ```
    - Запустим firewalld и поставим в автозагрузку
      ```
      sudo systemctl enable --now firewalld
      ```
    - Настроим firewalld
      ```
      sudo firewall-cmd --permanent --zone=public --add-port=80/tcp
      sudo firewall-cmd --permanent --zone=public --add-port=3306/tcp
      sudo firewall-cmd --reload
      ```
