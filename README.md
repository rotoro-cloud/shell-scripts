# Тестовое приложение для разворачивания с помощью Shell Scripts

В рамках финального задания для новичков я предлагаю "испачкать руки" и развернуть приложение с большим количеством зависимостей на нашем тестовом стенде.

Нужно создать скрипт, который выполнит развертывание приложения на узле `app01`. Тестовый стенд на базе CentOS 7.

1. Установим MariaDB
    - Установим вспомогательные утилиты
      ```
      yum update -y
      yum install -y wget net-tools
      ```
    - Добавим репозиторий MariaDB
      ```
      wget https://downloads.mariadb.com/MariaDB/mariadb_repo_setup
      chmod a+x mariadb_repo_setup 
      ./mariadb_repo_setup --mariadb-server-version=mariadb-10.6
      ```
    - Установка самой MariaDB
      ```
      yum install -y MariaDB-server
      ```
    - Запустим MariaDB и поставим в автозагрузку
      ```
      systemctl enable --now mariadb
      ```
    - Настройка MariaDB (user "root" на "localhost")
      ```
      mysql -sfu root <<EOS
      -- set root password
      ALTER USER 'root'@'localhost' IDENTIFIED BY 'my_strong_password';
      -- delete anonymous users
      DELETE FROM mysql.user WHERE User='';
      -- delete remote root capabilities
      DELETE FROM mysql.user WHERE User='root' AND Host NOT IN ('localhost', '127.0.0.1', '::1');
      -- drop database 'test'
      DROP DATABASE IF EXISTS test;
      -- also make sure there are lingering permissions to it
      DELETE FROM mysql.db WHERE Db='test' OR Db='test\\_%';
      -- make changes immediately
      FLUSH PRIVILEGES;
      EOS

2. Установим Nginx 
    - Добавим репозиторий Nginx
      ```
      (cat <<-EOF
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
      )>/etc/yum.repos.d/nginx.repo
      ```
    - Включим репозиторий nginx-mainline
      ```
      yum --enablerepo=nginx-mainline update
      ```
    - Установим Nginx и OpenSSL
      ```
      yum -y install nginx openssl
      ```
    - Настроим Nginx
      ```
      (cat <<-EOF
      server {
          listen       80;
          server_name  *.environments.katacoda.com;

          root   /usr/share/nginx/html/public;
          index index.php index.html index.htm;

          location / {
              try_files \$uri \$uri/ /index.php;
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
      )>/etc/nginx/conf.d/default.conf
      ```
    - Запустим Nginx и поставим в автозагрузку
      ```
      systemctl enable --now nginx
      ```

3. Установим PHP-FPM 7.4
    - Установим репозитории EPEL и Remi, там свежий PHP
      ```
      yum install -y yum-utils
      yum localinstall -y https://dl.fedoraproject.org/pub/epel/epel-release-latest-7.noarch.rpm
      yum localinstall -y https://rpms.remirepo.net/enterprise/remi-release-7.rpm
      yum-config-manager --enable remi-php74
      yum makecache fast
      ```
    - Установим PHP 7.4
      ```
      yum install -y php-cli php-fpm php-common php-curl php-gd php-imap php-intl php-mbstring php-xml php-zip php-bz2 php-bcmath php-json php-opcache php-devel php-mysqlnd php-pear gcc ImageMagick ImageMagick-devel
      ```
    - Поправим php.ini под пользователя nginx
      ```
      sed -i "s/^user = apache/user = nginx/g" /etc/php-fpm.d/www.conf
      sed -i "s/^group = apache/group = nginx/g" /etc/php-fpm.d/www.conf
      sed -i "s/^listen = 127.0.0.1:9000/listen = /var/run/php-fpm/php-fpm.sock/g" /etc/php-fpm.d/www.conf
      sed -i "s/^;listen.owner = nobody/listen.owner = nginx/g" /etc/php-fpm.d/www.conf
      sed -i "s/^;listen.group = nobody/listen.group = nginx/g" /etc/php-fpm.d/www.conf
      sed -i "s/^;listen.mode = 0660/listen.mode = 0660" /etc/php-fpm.d/www.conf
      ```
    - Настроим модуль PHP
      ```
      cat "extension=imagick.so" > /etc/php.d/imagick.ini
      ```
    - Запустим PHP-FPM и поставим в автозагрузку
      ```
      systemctl enable --now php-fpm
      ```
      
4. Ставим приложение
    - git clone https://github.com/rotoro-cloud/Laravel-Real-Estate-Venue-Portal.git /opt/app
    - создать .env из .env.example
      ```
      cp .env.example .env
      ```
    - поменять в нем параметры `DB_DATABASE`, `DB_USERNAME`, `DB_PASSWORD` на заданные ранее
      ```
      sed -i "s/^DB_PASSWORD=/DB_PASSWORD=my_strong_password/g" .env
      ```
    - с помощью пакетного менеджера OC установить `composer`
      ```
      yum install -y composer
      ```
    - подтянуть зависимости проекта с помощью 
      ```
      composer install
      ```
    - выполнить создание ключа приложения 
      ```
      php artisan key:generate;
      ```
    - наполнить базу тестовой информацией
      ```
      php artisan migrate --seed;
      ```
    - создать символические ссылки для хранилища 
      ```
      rm -rf public/storage; sudo php artisan storage:link;
      ```
    - дать доступ нужным директориям
      ```
      chown -R nginx.nginx /usr/share/nginx/html/;
      chmod -R ug+rwx /usr/share/nginx/html/storage /usr/share/nginx/html/bootstrap/cache;
      chmod -R o+rwx /usr/share/nginx/html/storage/logs;)
      ```

В курсе будет демонстрация решения, если ты вдруг застрял.
