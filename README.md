# Тестовое приложение для разворачивания с помощью Shell Scripts

[Репозиторий проекта](https://github.com/rotoro-cloud/Laravel-Real-Estate-Venue-Portal)

В рамках финального задания для новичков я предлагаю "испачкать руки" и развернуть приложение с большим количеством зависимостей на нашем тестовом стенде.

Нужно создать скрипт, который выполнит развертывание приложения на узле `starbase`. Тестовый стенд на базе CentOS 9.

Эта инструкция подразумевает развертывание вручную.

1. Установим MariaDB
    - Установим вспомогательные утилиты
      ```
      sudo yum install -y wget net-tools
      ```
    - Установка MariaDB
      ```
      sudo yum install -y mariadb-server
      ```
    - Запустим MariaDB и поставим в автозагрузку
      ```
      sudo systemctl enable --now mariadb
      ```
    - Настройка MariaDB (user "root" на "localhost")
      ```
      sudo mysql
      ```
      В mysql выполни команды:
      ```
      ALTER USER 'root'@'localhost' IDENTIFIED BY 'my_strong_password';
      DELETE FROM mysql.user WHERE User='';
      DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
      DROP DATABASE IF EXISTS test;
      CREATE DATABASE laravel;
      DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
      FLUSH PRIVILEGES;
      exit;
      ```

2. Установим Nginx 
    - Установим Nginx и OpenSSL
      ```
      sudo yum -y install nginx openssl
      ```
    - Настроим Nginx
      ```
      sudo vi /etc/nginx/conf.d/default.conf
      ```
      Замени содержимое на это:
      ```
      server {
          listen       80;
          server_name  *.dusha.name;

          root   /usr/share/nginx/html/public/public;
          index index.php index.html index.htm;

          location / {
              try_files $uri $uri/ /index.php$is_args$args;
          }
          error_page 404 /404.html;
          error_page 500 502 503 504 /50x.html;

          location = /50x.html {
              root /usr/share/nginx/html;
          }

          location ~ \.php$ {
              try_files $uri =404;
              fastcgi_pass unix:/var/run/php-fpm/www.sock;
              fastcgi_index index.php;
              fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
              include fastcgi_params;
          }
      }
      ```
    - Запустим Nginx и поставим в автозагрузку
      ```
      sudo systemctl enable --now nginx
      ```

3. Установим PHP-FPM 8
    - Установим PHP и дополнительные пакеты
      ```
      sudo yum install -y php-cli php-fpm php-common php-curl php-gd php-intl php-mbstring php-xml php-zip php-bz2 php-bcmath php-json php-opcache php-devel php-mysqlnd php-pear gcc ImageMagick ImageMagick-devel
      ```
    - Поправим php.ini под пользователя nginx
      ```
      sudo vi /etc/php-fpm.d/www.conf
      ```
      Здесь замени следующее:
      ```
      user = apache => user = nginx
      group = apache => group = nginx
      ;listen.owner = nobody => listen.owner = nginx
      ;listen.group = nobody => listen.group = nginx
      ;listen.mode = 0660 => listen.mode = 0660
      ```
    - Установим пакеты pear
      ```
      sudo pecl install imagick
      ```
    - Настроим модуль PHP
      ```
      sudo vi /etc/php.d/imagick.ini
      ```
      Добавь сюда:
      ```
      extension=imagick.so
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
      sudo vi .env
      ```
      Установи:
      ```
      DB_PASSWORD= => DB_PASSWORD=my_strong_password
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

В курсе будет демонстрация решения, если ты вдруг застрял. 

Обрати внимание, что демо сейчас на версии CentOS 7, а это решение уже для CentOS 9.
Отличия в том, что нам не требуется устанавливать дополнительные репозитории.
