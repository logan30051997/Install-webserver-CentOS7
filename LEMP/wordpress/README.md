# Install LEMP stack CentOS7 Wordpress
 
#### Mục lục
 
[Requirements](#requirements)
 
[I. Install Nginx](#Nginx)
 - [1. Tạo VitualHost chạy HTTP](#http)
 - [2. Tạo VitualHost chạy HTTPS](#https)
        
[II. Install Mariadb](#Mariadb)
 - [1. Tạo database](#createdatabase)
 - [2. Kiểm tra database](#checkdatabase)
 
[III. Install PHP-FPM](#php-fpm)

[IV. Install Wordpress](#wordpress)
 - [1. Download wordpress](#caidat)
 - [2. Phân quyền](#phanquyen)
 
 ===================================
 <a name="requirements"></a>
 ## Các Service và version yêu cầu 

System requirements: 

[Wordpress Requirements](https://wordpress.org/support/article/requirements/)

Version sử dụng:

- Nginx v1.16.1 [Repo epel](http://nginx.org/packages/centos/7/x86_64/)

- Mariadb-server v10.4 [Repo mariadb](https://mariadb.org/download/#mariadb-repositories)

- php-fpm v8.0 [Repo php-fpm](http://rpms.remirepo.net/enterprise/remi-release-7.rpm)

- Wordpress newversion 5.6.2 [Wordpress](https://wordpress.org/latest.zip)


=============================================
<a name="Nginx"></a>
## I. Install Nginx

- Cài Nginx v1.16.1 với repo có sẵn epel-release

```sudo yum install nginx```

```systemctl start nginx```

```nginx -v ```  ## kiểm tra version vừa cài

- Cho phép cổng firewall


```sudo firewall-cmd --permanent --add-service=http```

```sudo firewall-cmd --permanent --add-service=https```

```sudo firewall-cmd --reload```

- Truy cập IP để kiểm tra 

`http://'địa chỉ ip'`


<a name="http"></a>
### 1. Tạo VitualHost chạy HTTP

- Tạo file .conf trong `/etc/nginx/conf.d/`

`vim /etc/nginx/conf.d/wordpress.linex.vn.conf`

- File cấu hình như sau:

```
server{
        listen 80;
        server_name wordpress.linex.vn;
        root /home/www/wordpress.linex.vn/wordpress/;
        error_log /var/log/nginx/wordpress.linex.vn_error.log error;
        access_log /var/log/nginx/wordpress.linex.vn_access.log main;
    location /{
        index index.html index.php;
     }
}
```
<a name="https"></a>
### 2. Tạo VitualHost chạy HTTPS (Local)

- Cài mod_ssl và openssl

`yum install mod_ssl openssl -y`


- Tạo key và tự ký trên local

Di chuyển vào: `cd /etc/pki/tls/certs`

Tạo khóa: `make server.key` (Tạo passphrase)

Bỏ passphrase tron key: `openssl rsa -in server.key -out server.key`

Tự ký: `make server.csr`

Xác nhận và cấp hiệu lực key: `openssl x509 -in server.csr -out server.crt -req -signkey server.key -days 3650`

- Conf file cấu hình VitualHost chạy 443 và đường dẫn xác thực SSL

`vim /etc/nginx/conf.d/wordpress.linex.vn.conf`

File conf như sau:

```
server {
       #listen       80;
        listen       443 ssl;
        server_name  wordpress.linex.vn;
        root         /home/www/wordpress.linex.vn/wordpress/;
        error_log /var/log/nginx/wordpress.linex.vn_error.log error;
        access_log /var/log/nginx/wordpress.linex.vn_access.log main;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE+RSAGCM:ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:!aNULL!eNull:!EXPORT:!DES:!3DES:!MD5:!DSS;
        ssl_certificate      /etc/pki/tls/certs/server.crt;
        ssl_certificate_key  /etc/pki/tls/certs/server.key;
location /{
        index index.html index.php;
        try_files $uri /index.php?$query_string;

     }
}
```
<a name="Mariadb"></a>
## II. Install Mariadb

- Cài Mariadb v10.4 với repo [link](https://mariadb.org/download/#mariadb-repositories) có hướng dẫn

```sudo yum install MariaDB-server MariaDB-client```

```systemctl start mariadb```

```mariadb -v ``` ## kiểm tra version vừa cài

- Install secure database

```mysql_secure_installation  #set passwd root cho database```

<img src="https://i.imgur.com/uIxk9uF.png">

<a name="createdatabase"></a>
### 1. Tạo DataBase

- Truy cập vào database với quyền root, với password vừa tạo ở bước trên 

`mariadb -u root -p`

- Tạo database và user:

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
<a name="checkdatabase"></a>
### 2. Kiểm tra database

```
show databases;

SELECT host, user, password FROM mysql.user;

SHOW GRANTS FOR 'wordpress'@localhost;
```

<a name="php-fpm"></a>
## III. Install PHP-FPM

- Cài php-fpm với repo có các version đầy đủ

```sudo yum install http://rpms.remirepo.net/enterprise/remi-release-7.rpm```

*Enabled version mà mình muốn cài (Xem repo version tại ```cd /etc/yum.repos.d/```)*

```yum-config-manager --enable remi-php80 ```

```sudo yum install php php-fpm php-mysqlnd php-dom php-gd php-xml php-SimpleXML```

`systemctl start php-fpm`

```php-fpm -v ```   ## kiểm tra version vừa cài

### Cấu hình

- Backup file `www.conf`

`mv /etc/php-fpm.d/www.conf /etc/php-fpm.d/www.conf.bak`

- Sửa file cấu hình php-fpm:

`vim /etc/php-fpm.d/wordpress.conf` 

```
[wordpress]
listen = 127.0.0.1:9000
listen.allowed_clients = 127.0.0.1
listen.owner = nginx
listen.group = nginx
listen.mode = 0660
user = nginx
group = nginx
pm = dynamic
pm.max_children = 5
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 4
pm.max_requests = 200
listen.backlog = -1
pm.status_path = /status
request_terminate_timeout = 120s
rlimit_files = 131072
rlimit_core = unlimited
slowlog = /var/log/php-fpm/www-slow.log
php_admin_value[error_log] = /var/log/php-fpm/www-error.log
php_admin_flag[log_errors] = on
php_value[session.save_handler] = files
php_value[session.save_path] = /var/lib/php/session
security.limit_extensions = .php .php3 .php4 .php5 .php7
```

Sửa file cấu hình VitualHost chạy được file .php qua php-fpm

Thêm cấu hình sau: `vim /etc/nginx/conf.d/wordpress.linex.vn.conf`

```
location ~ \.php {
#   fastcgi_pass unix:/var/run/php_fpm.sock;
    fastcgi_pass 127.0.0.1:9000;
    include        /etc/nginx/fastcgi_params;
    fastcgi_param   SCRIPT_FILENAME $document_root/$fastcgi_script_name;
}
```
- reload php-fpm, nginx

`systemctl reload php-fpm`

`systemctl reload nginx`

<a name="wordpress"></a>
## IV. Install Wordpress

<a name="caidat"></a>

### 1. Cài Wordpress

Cài Wordpress version mới nhất

`cd /tmp && wget http://wordpress.org/latest.tar.gz`

Giải nén file vừa cài vào thư mục chứa

`tar -xvzf latest.tar.gz -C /home/www/test.com`

Phân quyền cho wordpress

`chown -R nginx /home/www/test.com/wordpress`

Copy file conf mẫu

`cp /home/www/test.com/wordpress/wp-config-sample.php /home/www/test.com/wordpress/wp-config.php`

Sửa file conf vừa copy thêm Database

`vim /home/www/test.com/wordpress/wp-config.php`

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

Kết quả khi nhập domain VitualHost


<img src=https://kinsta.com/wp-content/uploads/2018/12/wordpress-install-language.png>

<a name="phanquyen"></a>

### 2. Phân quyền cho folder và file 

*Theo quy chuẩn Wordpress để đảm bảo an toàn, các folder và file trong wordpress sẽ được phân quyền hợp lí: [link tham khảo](https://www.thuysys.com/domain-hosting/wordpress-co-ban/tim-hieu-chmod-chown-cach-sua-loi-phan-quyen-wordpress-tren-linux.html)*



- Phân quyền Folder 755

`find /home/www/test.com/wordpress -type d -print0 | xargs -0 chmod 755`

- Phân quyền File 644

`find /home/www/test.com/wordpress -type f -print0 | xargs -0 chmod 644`
