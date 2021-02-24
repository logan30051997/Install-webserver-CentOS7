# Install LEMP stack CentOS7 Drupal


#### Mục lục
 
[Requirements](#requirements)
 
[I. Install Nginx](#Nginx)
 - [1. Tạo VitualHost chạy HTTP](#http)
 - [2. Tạo VitualHost chạy HTTPS](#https)
        
[II. Install Mariadb](#Mariadb)
 - [1. Tạo database](#createdatabase)
 - [2. Kiểm tra database](#checkdatabase)
 
[III. Install PHP-FPM](#php-fpm)

[IV. Install Drupal](#drupal)
 - [1. Download Drupal](#download)
 - [2. Phân quyền](#permission)
 - [3. Cài đặt](#install)
 - [4. Kiểm tra](#check)
 
 ===================================
 <a name="requirements"></a>
 ## Các Service và version yêu cầu 

System requirements: 

[Drupal Requirements](https://www.drupal.org/docs/system-requirements)

Version sử dụng:

- Nginx v1.16.1 [Repo epel](http://nginx.org/packages/centos/7/x86_64/)

- Mariadb-server v10.4 [Repo mariadb](https://mariadb.org/download/#mariadb-repositories)

- php-fpm v8.0 [Repo php-fpm](http://rpms.remirepo.net/enterprise/remi-release-7.rpm)

- Drupal newversion 9.1.3 [Drupal](https://www.drupal.org/download-latest/tar.gz)

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

`vim /etc/nginx/conf.d/drupal.linex.vn.conf`

- File cấu hình như sau:

```
server{
        listen 80;
        server_name drupal.linex.vn;
        root /home/www/drupal.linex.vn/drupal;
        error_log /var/log/nginx/drupal.linex.vn_error.log error;
        access_log /var/log/nginx/drupal.linex.vn_access.log main;
location /{
        index index.html;
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

`vim /etc/nginx/conf.d/drupal.linex.vn.conf`

File conf như sau:

```
server {
       #listen       80;
        listen       443 ssl;
        server_name  drupal.linex.vn;
        root         /home/www/drupal.linex.vn/drupal/;
        error_log /var/log/nginx/drupal.linex.vn_error.log error;
        access_log /var/log/nginx/drupal.linex.vn_access.log main;
        ssl_protocols TLSv1 TLSv1.1 TLSv1.2;
        ssl_prefer_server_ciphers on;
        ssl_ciphers ECDHE+RSAGCM:ECDH+AESGCM:DH+AESGCM:ECDH+AES256:DH+AES256:ECDH+AES128:DH+AES:!aNULL!eNull:!EXPORT:!DES:!3DES:!MD5:!DSS;
        ssl_certificate      /etc/pki/tls/certs/server.crt;
        ssl_certificate_key  /etc/pki/tls/certs/server.key;
location /{
        index index.html index.php;
       #try_files $uri $uri/ /index.php?$args;
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
create database drupal;

# Tạo user
create user 'drupal'@localhost identified by 'password';

# Cấp all quyền cho user vào database
grant all privileges on *.* to 'drupal'@localhost;

# Xác nhận
flush privileges;
```
<a name="checkdatabase"></a>
### 2. Kiểm tra database

```
show databases;

SELECT host, user, password FROM mysql.user;

SHOW GRANTS FOR 'drupal'@localhost;
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

`vim /etc/php-fpm.d/www.conf` 

```
[www]
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

Thêm cấu hình sau: `vim /etc/nginx/conf.d/drupal.linex.vn.conf`

```
location /{
     try_files $uri /index.php?$query_string;
}
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

<a name="drupal"></a>
## IV. Install Drupal

<a name="download"></a>
### 1. Download Drupal

Cài Drupal version mới nhất

`cd /tmp && wget https://www.drupal.org/download-latest/tar.gz`

Giải nén file vừa cài vào thư mục chứa

`mkdir -p /home/www/drupal.linex.vn`

`tar -xvzf tar.gz -C /home/www/drupal.linex.vn`

Đổi tên file folder chứa Drupal

`mv /home/www/drupal.linex.vn/drupal-9.1.3/ /home/www/drupal.linex.vn/drupal/`

<a name="permission"></a>
### 2.Phân quyền cho Drupal

- Tạo thư mục chưa Drupal và phân quyền để folder theo đường dẫn `/home/www/drupal.linex.vn/`

`chown -R nginx:nginx /home/www/`

- Phân quyền Directory 755

`find /home/www/drupal.linex.vn/drupal -type d -print0 | xargs -0 chmod 755`

- Phân quyền File 644

`find /home/www/drupal.linex.vn/drupal -type f -print0 | xargs -0 chmod 644`

<a name="install"></a>
### 3. Cài đặt

- Nhập `https://drupal.linex.vn` để cài đặt

- Chọn ngôn ngữ

<img src="https://i.imgur.com/QLYeH9v.png">

- Có 3 chế độ:

*Standard*: sử dụng các tính năng của bản Drupal gốc. Profile này bao gồm tất cả những modules chuẩn được dùng nhiều nhất và thân thiện với người dùng.

*Minimal*: có nhiều khả năng tùy biến hơn để dựng website. Profile này được thiết kế để sử dụng bởi những lập trình viên chuyên nghiệp.

*Demo*:
    

<img src="https://i.imgur.com/zke3oQj.png">

- Kiểm tra lại các service và yêu cầu liên quan đến cấu hình

<img src="https://i.imgur.com/5df2jh9.png">


- Nhập dữ liệu database đã tạo ở Mariadb

<img src="https://i.imgur.com/dKfVxXE.png">

- Install

<img src="https://i.imgur.com/6Y05a0j.png">

- Nhập thông tin

<img src="https://i.imgur.com/nWw5JrM.png">

- Hoàn tất Drupal

<img src="https://i.imgur.com/R0NQ2sa.png">

<a name="check"></a>
### 4. Kiểm tra

- Đăng bài mới có hình ảnh

<img src="https://i.imgur.com/Ip4zrUo.png">

- Quản lý các bài đăng

<img src="https://i.imgur.com/osf3XsT.png">

- Quản lý user

<img src="https://i.imgur.com/avLQKQ4.png">

- **Thông tin Web**

```
Địa chỉ IP: 10.20.20.101
Port: 443
Domain: drupal.linex.vn
Account: admin
Passwd: admin
```
