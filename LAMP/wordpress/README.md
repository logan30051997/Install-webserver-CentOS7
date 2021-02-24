# Install-LAMP-stack-CentOS7-Wordpress
Cài LAMP, Wordpress trên CentOS7

#### Các Service và version yêu cầu 


- Apache (httpd) v2.46 (Có sẵn trong repo epel)

- Mariadb-server v10.1 ++ [repo mariadb sv](https://mariadb.org/download/#mariadb-repositories)

- php-fpm v7.3 ++ [repo php-fpm](http://rpms.remirepo.net/enterprise/remi-release-7.rpm)

- wordpress newversion [down wordpress](http://wordpress.org/latest.tar.gz)

# I. Install httpd (Apache)

- Cài  với repo có sẵn epel-release

```sudo yum install httpd```

```httpd -v ```  ## kiểm tra version vừa cài

- Bật service và cho phép cổng firewall

```systemctl start httpd```

```sudo firewall-cmd --permanent --add-service=http```

```sudo firewall-cmd --permanent --add-service=https```

```sudo firewall-cmd --reload```

### 1. Tạo VitualHost chạy HTTP

- Tạo các folder liên quan

Tạo sites folder để dễ dàng quản lí các web

`mkdir /etc/httpd/sites-available /etc/httpd/sites-enabled`

Tạo folder đường dẫn để file .html, .php và phân quyền

`mkdir -p /var/www/html/test.com`

`chown -R apache:apache /var/www/html/test.com`

`chmod -R 755 /var/www`

Tạo folder chưa file log của trang

`mkdir -p /var/www/html/test.com/log`

Tạo file .html để test

`vim /var/www/html/test.com/test.html`

- Tạo VitualHost trong conf

Vào file httpd.conf trong `vim /etc/httpd/conf/httpd.conf` để thêm dòng

`IncludeOptional sites-enabled/*.conf` 

Tạo VitualHost trong sites-available 

`vim /etc/httpd/sites-available/test.com.conf`

Thêm các cấu hình sau:

```
<VirtualHost *:80>
    ServerName test.com
    ServerAlias www.test.com
    DocumentRoot /var/www/html/test.com
    ErrorLog /var/www/html/test.com/log/error.log
    CustomLog /var/www/html/test.com/log/requests.log combined
</VirtualHost>
```

Symlink từ sites-available sang sites-enabled

`ln -s /etc/httpd/sites-available/test.com.conf /etc/httpd/sites-enabled/test.com.conf`

Reload httpd 

`systemctl reload httpd`


### 2. Tạo VitualHost chạy HTTPS (local)

- Cài mod_ssl và openssl

`yum install mod_ssl openssl -y `

- Tạo key và tự ký trên local

Di chuyển vào: `cd /etc/pki/tls/certs`

Tạo khóa: `make server.key` (Tạo passphrase)

Bỏ passphrase tron key: `openssl rsa -in server.key -out server.key`

Tự ký: `make server.csr`

Xác nhận và cấp hiệu lực key: `openssl x509 -in server.csr -out server.crt -req -signkey server.key -days 3650`


- Conf file cấu hình VitualHost chạy 443 và đường dẫn xác thực SSL

`vim /etc/httpd/sites-available/test.com.conf`

Sửa file thêm cấu hình như sau:
```
<VirtualHost *:443>
        ServerName test.com
        ServerAlias test.com
        DocumentRoot /var/www/html/test.com/wordpress
        ErrorLog /var/www/html/test.com/log/error.log
        CustomLog /var/www/html/test.com/log/requests.log combined
        SSLEngine on
        SSLCertificateFile /etc/pki/tls/certs/server.crt
        SSLCertificateKeyFile /etc/pki/tls/certs/server.key
</VirtualHost>
```

- Symlink lại sang sites-enabled và reload để chạy

`ln -sf /etc/httpd/sites-available/test.com.conf /etc/httpd/sites-enabled/test.com.conf`

`systemctl reload httpd`

## II. Install Mariadb

- Cài Mariadb v10.5 với repo [link](https://mariadb.org/download/#mariadb-repositories) có hướng dẫn

```sudo yum install MariaDB-server MariaDB-client```


```mariadb -v ``` ## kiểm tra version vừa cài

- Bật service và install bảo mật cho database

```systemctl start mariadb```

```mysql_secure_installation  #set passwd root cho database```

### 1. Tạo DataBase

Truy cập vào database với quyền root, với password vừa tạo ở bước trên 

`mariadb -u root -p`

Tạo database và user:

```
# Tạo database
create database wordpress;

# Tạo user
create user 'wordpress'@localhost identified by 'password';

# Cấp quyền cho user vào database
grant all privileges on *.* to 'wordpress'@localhost;

# Xác nhận
flush privileges;
```

Kiểm tra database đã tạo:

```
SELECT host, user, password FROM mysql.user;

SHOW GRANTS FOR 'user'@localhost;
```

## III. Install PHP

- Cài php-fpm v7.4 với repo có các version đầy đủ

```sudo yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm```

*Enabled version mà mình muốn cài (Xem repo version tại ```cd /etc/yum.repos.d/```)*

```yum-config-manager --enable remi-php74 ```

```sudo yum install php php-mysqlnd php-fpm```

`systemctl start php-fpm`

```php-fpm -v ```   ## kiểm tra version vừa cài

### 1. Cấu hình

Sửa file cấu hình php-fpm `vim /etc/php-fpm.d/www.conf` 

```
[www]
listen = 127.0.0.1:9000
listen.allowed_clients = 127.0.0.1
listen.owner = apache
listen.group = apache
listen.mode = 0660
user = apache
group = apache
pm = dynamic
pm.max_children = 50
pm.start_servers = 5
pm.min_spare_servers = 5
pm.max_spare_servers = 35
slowlog = /var/log/php-fpm/www-slow.log
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
php_value[session.save_handler] = files
php_value[session.save_path] = /var/lib/php/session
security.limit_extensions = .php .php3 .php4 .php5 .php7
```

Sửa file cấu hình VitualHost chạy được file /php qua php-fpm

Thêm cấu hình `proxy:fcgi..` sau vào: `vim /etc/httpd/conf.d/php.conf`

```
...
    <FilesMatch \.(php|phar)$>
       # SetHandler application/x-httpd-php
        SetHandler "proxy:fcgi://127.0.0.1:9000"
    </FilesMatch>

```

## IV. Install Wordpress

### 1. Cài Wordpress

Cài Wordpress version mới nhất

`cd /tmp && wget http://wordpress.org/latest.tar.gz`

Giải nén file vừa cài vào thư mục chứa

`tar -xvzf latest.tar.gz -C /var/www/html/test.com`

Phân quyền cho wordpress

`chown -R apache:apache /var/www/html/test.com/wordpress`

Copy file conf mẫu

`cp /var/www/html/test.com/wordpress/wp-config-sample.php /var/www/html/test.com/wordpress/wp-config.php`

Sửa file conf vừa copy thêm Database

`vim /var/www/html/test.com/wordpress/wp-config.php`

Thêm các dòng:

```
/** The name of the database for WordPress */
define( 'DB_NAME', 'wordpress' );
/** MySQL database username */
define( 'DB_USER', 'wordpress' );
/** MySQL database password */
define( 'DB_PASSWORD', 'password' );
/** MySQL hostname */
define( 'DB_HOST', 'localhost' );
```

### 2. Phân quyền cho folder và file 

*Theo quy chuẩn Wordpress để đảm bảo an toàn, các folder và file trong wordpress sẽ được phân quyền hợp lí: [link tham khảo](https://www.thuysys.com/domain-hosting/wordpress-co-ban/tim-hieu-chmod-chown-cach-sua-loi-phan-quyen-wordpress-tren-linux.html)*



- Phân quyền Folder 755

`find /var/www/html/test.com/wordpress -type d -print0 | xargs -0 chmod 755`

- Phân quyền File 644

`find /var/www/html/test.com/wordpress -type f -print0 | xargs -0 chmod 644`
