# Tips và Tricks cấu hình Nginx

## Nginx Tip 1. Tổ chức các Nginx config files

Thông thường các file config được đặt trong /etc/nginx

Có thể tổ chức các config files giống như cài đặt Apache trên Ubuntu/Debian:

**Main configuration file**
`etc/nginx/nginx.conf`

**Virtualhost configuration files on**

`
/etc/nginx/sites-available/
/etc/nginx/sites-enabled/
`

## Other config files on (if needed) ##

`/etc/nginx/conf.d/`

Virtualhost files có 2 thư mục, vì thư mục sites-available có thể chứa bất kỳ các file config nào, vd như các configs sử dụng để test, configs được clone, configs được tạo mới, config cũ, .... Và sites-enabled chỉ chứa những configs thực sự được kích hoạt, thực ra chỉ là các symbolic links tới thư mục sites-available.

Bạn cần thêm đoạn include các files config được thêm trong nginx.conf file:

**Load virtual host conf files.**

`include /etc/nginx/sites-enabled/*`

**Load another configs from conf.d/**

`include /etc/nginx/conf.d/*;`

## Nginx Tip 2. Xác định Nginx worker_processes và worker_connections

Thiết lập mặc định là ổn đối với worker_ processes và worker_connections, nhưng các giá trị này có thể tối ưu thêm một chút:

```
max_clients = worker_processes * worker_connections
```

Chỉ cần thiết lập Nginx cơ bản có thể xử lý hàng trăm kết nối đồng thời:

```
worker_processes  1;
worker_connections  1024;
```

Thông thường 1000 concurrent connection / trên 1 server là tốt, nhưng thỉnh thoảng các thành phần khác như các ổ đĩa trên server có thể chậm, và nó sẽ làm Nginx bị locked trong các hoạt động I/O. Để tránh locking sử dụng config như này: 1 worker_precess / trên 1 processor core, giống như:

**Worker Processes** 

`worker_processes [number of processor cores];`

Để kiểm tra xem bạn có bao nhiêu processor cores, hãy chạy command sau:

```
cat /proc/cpuinfo |grep processor
processor	: 0
processor	: 1
processor	: 2
processor	: 3
```
Ở đây có 4 cores và worker_processes có thể được cấu hình như sau:

`worker_processes 4;`

**Worker Connections**

Cá nhân tôi gắn bó với 1024 worker connection, bởi vì tôi không có lý do gì để nâng giá trị này. Nhưng nếu ví dụ 4096 kết nối mỗi giây là không đủ thì có thể cố gắng nhân đôi số này và đặt 2048 connections cho mỗi process.

Cấu hình cuối cùng của worker_processes có thể như sau:

`worker_connections 1024;`


## Nginx Tip 3. – Ẩn Nginx Server Tokens / Ẩn Nginx version

Với lý do bảo mật, ẩn server tokens / ẩn Nginx version, đặc biệt nếu bạn chạy phiến bản cũ của Nginx. Việc này vô cùng đơn giản, bạn chỉ cần set `server_tokens off` trong http/server/location section, như sau:

`server_tokens off;`

## Nginx Tip 4. – Nginx Request / Upload Max Body Size (client_max_body_size)

Nếu bạn muốn cho phép người dùng tải lên một cái gì đó qua HTTP thì bạn có thể tăng post size. Nó được thực hiện bằng việc thay đổi giá trị `client_max_body_size` nằm trong http/server/location section. Theo mặc định là 1 Mb, nhưng nó có thể được đặt thành 20 Mb và cũng tăng buffer size với cấu hình sau:

```
client_max_body_size 20m;
client_body_buffer_size 128k;
```

Nếu bạn gặp phải lỗi sau đây, bạn nên biết rằng client_max_body_size quá thấp:

``“Request Entity Too Large” (413)``

## Nginx Tip 5. – Nginx Cache Control cho Static Files (Browser Cache Control Directives)

Browser caching là cần thiết nếu bạn muốn tiết kiệm resources và bandwith. Nó là đơn giản để thiết lập với Nginx, tắt log (access log và not found log) và set expires headers thành 360 ngày.

```
location ~* \.(jpg|jpeg|gif|png|css|js|ico|xml)$ {
    access_log        off;
    log_not_found     off;
    expires           360d;
}
```

Nếu bạn muốn thêm các header đặc biệt cho từng loại file hoặc thời gian expires khác nhau, bạn có thể cấu hình cho từng loại file.

## Nginx Tip 6. – Nginx Pass PHP requests tới PHP-FPM

Tại đây bạn có thể sử dụng tpc/ip stack mặc định hoặc sử dụng Unix socket connection trực tiếp. Bạn cũng phải thiết lập PHP-FPM listen nghe chính xác cùng một ip:port hoặc unix socket (với Unix socket cũng phải có socket permission). Thiết lập mặc định là sử dụng ip:port (127.0.0.1:9000) tất nhiên bạn có thể thay đổi ips và ports mà PHP-FPM listens. Đây là cấu hình rất cơ bản với ví dụ về Unix socket:

**Pass PHP scripts to PHP-FPM**

```
location ~* \.php$ {
    fastcgi_index   index.php;
    fastcgi_pass    127.0.0.1:9000;
    #fastcgi_pass   unix:/var/run/php-fpm/php-fpm.sock;
    include         fastcgi_params;
    fastcgi_param   SCRIPT_FILENAME    $document_root$fastcgi_script_name;
    fastcgi_param   SCRIPT_NAME        $fastcgi_script_name;
}
```

Điều đó có nghĩa là bạn có thể chạy PHP-FPM và Nginx trên 2 server khác nhau.

## Nginx Tip 7. – Ngăn truy cập tới các File ẩn với Nginx

Nó rất phổ biến rằng server root hoặc các public directory khác có các file ẩn, bắt đầu bằng dấu chấm (.) và thông thường chúng không dành cho người dùng trang web. Public directory có thể chứa các version control files và directories, như .svn, IDE properties files và .htaccess files. Sau đây là đoạn config từ chối truy cập và tắt log cho tất cả các file ẩn.

```
location ~ /\. {
    access_log off;
    log_not_found off; 
    deny all;
}
```
## Nginx Tip 8. - Đổi định dạng đuôi file

Theo yêu cầu giấu đuôi .php, .html có thể đổi sang đuôi .* để chạy file 

- Chỉnh sửa trong file `vim /etc/nginx/mime.types`

Dòng 2 `text/html  html htm shtml in` thêm đuôi .* mà mình muốn (Trong TH này thêm đuôi in, kiểm tra đuôi in đã có trong file conf này chưa )

- Chỉnh sửa trong file `vim /etc/php-fpm.d/www.conf`

Thêm dòng: `security.limit_extensions = .php .in` và đuôi .in 

- Reload lại Nginx và php-fpm



# Tips và tricks cấu hình PHP-FPM 

## Tip 1. – PHP-FPM Configuration files

Thông thường các PHP-FPM configuration files được đặt trong /etc/php-fpm.conf và đường dẫn /etc/php-fpm.d. Đây thường là khởi đầu tuyệt vời và tất cả các pool configs sẽ chuyển đến thư mục /etc/php-fpm.d. Bạn cần thêm dòng include sau vào php-fpm.conf file:

`include=/etc/php-fpm.d/*.conf`

## PHP-FPM Tip 2. – Tinh chỉnh PHP-FPM Global Configuration

Thiết lập emergency_restart_threshold, emergency_restart_interval và process_control_timeout. Giá trị mặc định các option này đều làoff, nhưng tôi nghĩ nó tốt hơn nên sử dụng các ví dụ về các option này như sau:

```
emergency_restart_threshold 10
emergency_restart_interval 1m
process_control_timeout 10s
```

Những cái này nghĩa là gì? Nếu 10 PHP-FPM child processes exit với SIGSEGV hoặc SIGBUS trong vòng 1 phút thì PHP-FPM sẽ tự động restart. Cấu hình này cũng đặt giới hạn thời gian 10 giây cho các child processes để chờ phản ứng trên các tín hiệu từ master.

## PHP-FPM Tip 3. – PHP-FPM Pools Configuration

Với PHP-FPM, có thể sử dụng các pools khác nhau cho các trang web khác nhau và phân bổ resources rất chính xác và thậm chí sử dụng các groups và users khác nhau cho mỗi pool. Dưới đây là ví dụ cấu trúc file cho các PHP-FPM pools cho ba trang web khác nhau (hoặc thực tế là ba phần khác nhau của cùng một trang web):

```
/etc/php-fpm.d/site.conf
/etc/php-fpm.d/blog.conf
/etc/php-fpm.d/forums.conf
```

Ví dụ cấu hình cho các pools như sau:

``/etc/php-fpm.d/site.conf``

```
[site]
listen = 127.0.0.1:9000
user = site
group = site
request_slowlog_timeout = 5s
slowlog = /var/log/php-fpm/slowlog-site.log
listen.allowed_clients = 127.0.0.1
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
catch_workers_output = yes
env[HOSTNAME] = $HOSTNAME
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
/etc/php-fpm.d/blog.conf

[blog]
listen = 127.0.0.1:9001
user = blog
group = blog
request_slowlog_timeout = 5s
slowlog = /var/log/php-fpm/slowlog-blog.log
listen.allowed_clients = 127.0.0.1
pm = dynamic
pm.max_children = 4
pm.start_servers = 2
pm.min_spare_servers = 1
pm.max_spare_servers = 3
pm.max_requests = 200
listen.backlog = -1
pm.status_path = /status
request_terminate_timeout = 120s
rlimit_files = 131072
rlimit_core = unlimited
catch_workers_output = yes
env[HOSTNAME] = $HOSTNAME
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
/etc/php-fpm.d/forums.conf

[forums]
listen = 127.0.0.1:9002
user = forums
group = forums
request_slowlog_timeout = 5s
slowlog = /var/log/php-fpm/slowlog-forums.log
listen.allowed_clients = 127.0.0.1
pm = dynamic
pm.max_children = 10
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 4
pm.max_requests = 400
listen.backlog = -1
pm.status_path = /status
request_terminate_timeout = 120s
rlimit_files = 131072
rlimit_core = unlimited
catch_workers_output = yes
env[HOSTNAME] = $HOSTNAME
env[TMP] = /tmp
env[TMPDIR] = /tmp
env[TEMP] = /tmp
```

Đây là chỉ là ví dụ cho config nhiều site pools khác nhau.

## PHP-FPM Tip 4. – PHP-FPM Pool Process Manager (pm) Configuration

Cách tốt nhất để sử dụng PHP-FPM process manager là sử dụng dynamic process management, vì vậy các PHP-FPM processes chỉ được bắt đầu khi cần. Đây là thiết lập kiểu gần giống với thiết lập Nginx `worker_processes` và` worker_connections`. 
Vì vậy, giá trị rất cao không có nghĩa là tất cả mọi thứ tốt. Mọi process đều ăn bộ nhớ và tất nhiên nếu trang web có lưu lượng truy cập rất cao và rất nhiều bộ nhớ thì các giá trị cao hơn sẽ được chọn, nhưng các server, như bộ nhớ VPS (Virtual Private Servers) thường bị giới hạn ở 256 Mb, 512 Mb, 1024 Mb. RAM thấp này đủ để xử lý lưu lượng truy cập rất cao (thậm chí hàng chục yêu cầu mỗi giây), nếu nó được sử dụng một cách khôn ngoan.

Thật tốt khi kiểm tra có bao nhiêu PHP-FPM processes một máy chủ có thể xử lý dễ dàng, trước tiên hãy khởi động Nginx và PHP-FPM và tải một số trang PHP, tốt nhất là tất cả các trang nặng nhất. Sau đó kiểm tra mức sử dụng bộ nhớ cho mỗi ví dụ về PHP-FPM processes bằng command top hoặc htop trên Linux. 

Giả sử rằng server có bộ nhớ 512 Mb và 220 Mb có thể được sử dụng cho PHP-FPM, mỗi process sử dụng 24 Mb RAM(một số hệ thống quản lý nội dung khổng lồ với các plugin có thể dễ dàng sử dụng 20-40 Mb / mỗi PHP page request hoặc thậm chí nhiều hơn) . 
Sau đó, chỉ cần tính giá trị 

`max_children của server: 220/24 = 9,17`

Vì vậy, giá trị pm.max_children tốt là 9. Điều này chỉ dựa trên mức trung bình nhanh và sau này có thể là một cái gì đó khác khi bạn thấy thời gian sử dụng bộ nhớ / trên mỗi process dài hơn. Sau khi test xong, nó dễ dàng hơn nhiều để thiết lập giá trị `pm.start_servers`, giá trị `pm.min_spare_servers` và giá trị `pm.max_spare_servers`.

Cấu hình ví dụ có thể như sau:

```
pm.max_children = 9
pm.start_servers = 3
pm.min_spare_servers = 2
pm.max_spare_servers = 4
pm.max_requests = 200
