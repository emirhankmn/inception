# 📘 Inception Project – Detaylı README (Türkçe Açıklamalı)

Bu proje, Docker ve Docker Compose kullanılarak MariaDB, WordPress (PHP-FPM) ve SSL destekli NGINX içeren tam kapsamlı bir web sistemi kurar. Her servis, ayrı konteynerde izole bir şekilde çalışır ve otomatik olarak yapılandırılır.

---

## 🔧 Yapılandırma Dosyalarının Açıklamaları

### 📁 `/requirements/mariadb/conf/50-server.cnf`

```ini
[mysqld]                           # MySQL sunucusu için genel ayarlar
user = mysql                      # MySQL sunucusu "mysql" kullanıcısıyla çalışır
socket = /run/mysqld/mysqld.sock  # Yerel bağlantılar için kullanılan socket dosyası
port = 3306                       # MySQL varsayılan port numarası
basedir = /usr                   # MariaDB programlarının bulunduğu dizin
datadir = /var/lib/mysql         # Veritabanı verilerinin bulunduğu dizin
log_error = /var/log/mysql/error.log  # Hata log dosyasının yolu
expire_logs_days = 10            # 10 günden eski log dosyaları silinir
character-set-server = utf8mb4   # Veritabanının karakter kodlaması (tam Unicode desteği için)
bind-address = 0.0.0.0           # Her IP'den bağlantı kabul eder (güvenlik duvarı ile sınırlandırılmalı)
```

### 🖥️ `/requirements/mariadb/tools/dbmaria.sh`

```bash
#!/bin/bash

service mariadb start                             # MariaDB servisini başlatır
echo "MariaDB service has started."
echo "Waiting for MariaDB to be ready..."
while [ ! -S /var/run/mysqld/mysqld.sock ]; do    # Socket hazır olana kadar bekler
    sleep 0.5
done

mariadb -e "CREATE DATABASE IF NOT EXISTS $MYSQL_DATABASE_NAME;"      # Veritabanı oluşturulur
mariadb -e "CREATE USER IF NOT EXISTS '$MYSQL_USER'@'%' IDENTIFIED BY '$MYSQL_PASSWORD';"  # Kullanıcı oluşturulur
mariadb -e "GRANT ALL PRIVILEGES ON $MYSQL_DATABASE_NAME.* TO '$MYSQL_USER'@'%';"  # Yetkiler verilir
mariadb -e "FLUSH PRIVILEGES;"                     # Yetkiler geçerli hale getirilir
mariadb -e "SHUTDOWN;"                             # MariaDB kapanır (CMD satırından tekrar başlar)

exec "$@"                                          # Gelen komutu çalıştırır (CMD)
```

### 🐳 `/requirements/mariadb/Dockerfile`

```Dockerfile
FROM debian:bullseye                    # Debian tabanlı imaj kullanılır
RUN apt-get update && \
    apt-get install -y mariadb-server  # MariaDB yüklenir
EXPOSE 3306                             # 3306 portu dışa açılır
COPY ./conf/50-server.cnf /etc/mysql/mariadb.conf.d/   # Konfigürasyon dosyası kopyalanır
COPY ./tools/dbmaria.sh /              # Script dosyası kopyalanır
RUN chmod +x /dbmaria.sh               # Script çalıştırılabilir hale getirilir
ENTRYPOINT [ "/dbmaria.sh" ]           # Başlangıç noktası olarak script belirtilir
CMD [ "mariadbd" ]                      # Script sonunda çalıştırılacak ana işlem
```

---

### 📁 `/requirements/nginx/conf/default`

```nginx
server {
	listen 443 ssl;                       # SSL ile 443 portunu dinle
	listen [::]:443 ssl;                  # IPv6 için SSL bağlantısı
	server_name deryacar.42.fr;          # Sunucu adı (domain)
	ssl_certificate /etc/ssl/certs/nginx.crt;          # SSL sertifikası
	ssl_certificate_key /etc/ssl/private/nginx.key;    # SSL anahtarı
	ssl_protocols TLSv1.3;               # Kullanılacak SSL protokolü
	root /var/www/html;                  # Web sunucu kök dizini
	index index.php;                     # Varsayılan dosya index.php

	location / {
		try_files $uri $uri/ =404;         # Dosya yoksa 404 döndür
	}

	location ~ \.php$ {
		include snippets/fastcgi-php.conf;    # PHP için FastCGI ayarları
		fastcgi_pass wordpress:9000;         # PHP isteklerini WordPress konteynerine yönlendir
		proxy_connect_timeout 300s;          # Zaman aşımı ayarları
		proxy_send_timeout 300s;
		proxy_read_timeout 300s;
		fastcgi_send_timeout 300s;
		fastcgi_read_timeout 300s;
	}
}
```

### 🖥️ `/requirements/nginx/tools/nxstart.sh`

```bash
#!/bin/bash

if [ ! -f /etc/ssl/certs/nginx.crt ]; then                   # Sertifika daha önce oluşturulmadıysa
openssl req -x509 -nodes -days 365 -newkey rsa:4096 \      # Self-signed SSL sertifikası oluştur
-keyout /etc/ssl/private/nginx.key \                      
-out /etc/ssl/certs/nginx.crt \                           
-subj "/C=$C/ST=$ST/L=$L/O=$O/CN=$DOMAIN_NAME";
echo "SSL certificate and key successfully created."
fi
exec "$@"                                                 # Gelen komutu çalıştır
```

### 🐳 `/requirements/nginx/Dockerfile`

```Dockerfile
FROM debian:bullseye
RUN apt-get update && apt-get install -y nginx openssl       # NGINX ve OpenSSL kurulumu
EXPOSE 443                                                    # HTTPS portu dışa açılır
COPY ./conf/default /etc/nginx/sites-enabled/                # NGINX yapılandırması kopyalanır
COPY ./tools/nxstart.sh /                                     # Başlangıç scripti
RUN chmod +x /nxstart.sh                                     # Script çalıştırılabilir yapılır
ENTRYPOINT [ "/nxstart.sh" ]                                # Container başladığında bu script çalışır
CMD [ "nginx", "-g", "daemon off;" ]                         # Arka planda değil, direkt çalıştırılır
```

---

### 📁 `/requirements/wordpress/conf/www.conf`

```ini
[www]                                  # PHP-FPM için yapılandırma bloğu
user = www-data                        # PHP işlemleri www-data kullanıcısıyla çalıştırılır
group = www-data                       # PHP işlemleri www-data grubuyla çalıştırılır
listen = 9000                          # PHP-FPM bu port üzerinden dinler
listen.owner = www-data               # Dinleyici sahibi www-data
listen.group = www-data               # Dinleyici grubu www-data
pm = dynamic                           # Dinamik process yönetimi
pm.max_children = 5                   # Maksimum çocuk süreç sayısı
pm.start_servers = 2                  # Başlangıçta başlatılacak süreç sayısı
pm.min_spare_servers = 1             # Minimum boşta bekleyen süreç sayısı
pm.max_spare_servers = 3             # Maksimum boşta bekleyen süreç sayısı
```

### 🖥️ `/requirements/wordpress/tools/wp.sh`

```bash
#!/bin/bash

chown -R www-data: /var/www/*;                     # Dosya sahipliğini ayarla
chmod -R 755 /var/www/*;                           # Dosya izinlerini ayarla
mkdir -p /run/php/;                                # PHP çalışma dizinini oluştur
touch /run/php/php7.4-fpm.pid;                     # PID dosyası oluştur

if [ ! -f /var/www/html/wp-config.php ]; then      # Eğer WordPress kurulmamışsa
    mkdir -p /var/www/html;
    cd /var/www/html;

    wp-cli core download --allow-root;             # WordPress'i indir

    wp-cli config create --allow-root \            # wp-config.php dosyasını oluştur
        --dbname=$MYSQL_DATABASE_NAME \
        --dbuser=$MYSQL_USER \
        --dbpass=$MYSQL_PASSWORD \
        --dbhost=mariadb;

    echo "Installing WordPress core..."

    wp-cli core install --allow-root \             # WordPress çekirdeğini kur
        --url=$DOMAIN_NAME \
        --title=$TITLE \
        --admin_user=$WORDPRESS_ADMIN_NAME \
        --admin_password=$WORDPRESS_ADMIN_PASSWORD \
        --admin_email=$WORDPRESS_ADMIN_EMAIL;

    wp-cli user create --allow-root \              # İkinci bir kullanıcı oluştur
        $MYSQL_USER $MYSQL_EMAIL \
        --user_pass=$MYSQL_PASSWORD;

    echo "WordPress installation complete."
fi

echo "You're able to visit $DOMAIN_NAME in your browser."

exec "$@"                                           # Son komutu çalıştır (php-fpm)
```

### 🐳 `/requirements/wordpress/Dockerfile`

```Dockerfile
FROM debian:bullseye

RUN apt-get update && apt-get install -y php-fpm php-mysql sendmail wget   # PHP-FPM ve gerekli araçlar yüklenir

RUN wget https://raw.githubusercontent.com/wp-cli/builds/gh-pages/phar/wp-cli.phar \  # WP-CLI indirilir
    && chmod +x wp-cli.phar \
    && mv wp-cli.phar /usr/local/bin/wp-cli

EXPOSE 9000                                     # PHP-FPM portu dışa açılır

COPY ./conf/www.conf /etc/php/7.4/fpm/pool.d/   # PHP-FPM yapılandırması kopyalanır
COPY ./tools/wp.sh /                            # WordPress kurulum scripti
RUN chmod +x /wp.sh                             # Script çalıştırılabilir hale getirilir

ENTRYPOINT [ "/wp.sh" ]                         # Container başlatıldığında bu script çalışır
CMD [ "/usr/sbin/php-fpm7.4", "--nodaemonize" ] # PHP-FPM servisi foreground olarak çalışır
```
