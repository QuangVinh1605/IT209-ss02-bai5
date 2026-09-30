# Bài 5: Quản lý quyền sở hữu thư mục Web

## 1. Mục tiêu

Phân quyền cho thư mục `/var/www/ptit-web/` để đáp ứng đồng thời các yêu cầu:

- User `devops` là chủ sở hữu và có thể đọc, sửa mã nguồn mà không cần `sudo`.
- Group `www-data` có thể đọc file và đi xuyên qua các thư mục để Nginx phục vụ website tĩnh.
- Người dùng khác không có quyền truy cập.
- File và thư mục mới do `devops` tạo tiếp tục thuộc group `www-data`.

## 2. Kiểm tra trước khi thực hiện

Kiểm tra user `devops`, group `www-data` và cây thư mục web:

```bash
id devops
getent group www-data
sudo find /var/www/ptit-web -maxdepth 2 -printf '%M %u:%g %p\n'
```

Nếu hai lệnh đầu trả về thông tin tài khoản và nhóm thì có thể tiếp tục. Trong bài này giả sử website đã nằm tại `/var/www/ptit-web/html/` và Nginx đang trỏ `root` tới thư mục đó.

## 3. Thay đổi owner và group

Chuyển chủ sở hữu toàn bộ cây thư mục sang `devops` và group sang `www-data`:

```bash
sudo chown -R devops:www-data /var/www/ptit-web
```

Tùy chọn `-R` áp dụng thay đổi cho thư mục gốc và toàn bộ nội dung bên trong.

## 4. Thiết lập permission

Không đặt cùng một mode số cho cả file và thư mục. Thư mục bắt buộc phải có bit `x` thì Nginx mới có thể đi xuyên qua để mở file bên trong.

```bash
# Thư mục: devops=rwx, www-data=r-x, others=---, có setgid
sudo find /var/www/ptit-web -type d -exec chmod 2750 {} +

# File: devops=rw-, www-data=r--, others=---
sudo find /var/www/ptit-web -type f -exec chmod 0640 {} +
```

| Đối tượng | Mode | `devops` | `www-data` | Others |
| --- | ---: | --- | --- | --- |
| Thư mục | `2750` | `rwx` | `r-x` | `---` |
| File | `0640` | `rw-` | `r--` | `---` |

Bit đầu tiên `2` trong `2750` là **setgid**. Nhờ đó file và thư mục mới được tạo bên dưới sẽ kế thừa group `www-data`. Khi triển khai file mới, có thể đặt thêm `umask 027` để file mới nhận quyền mặc định `640` và thư mục mới nhận quyền `750`.

> Không dùng `chmod -R 640 /var/www/ptit-web`. Cách này xóa quyền `x` của thư mục, khiến Nginx không thể truy cập file và có thể trả về `403 Forbidden`.

## 5. Kiểm tra kết quả bằng `ls -la`

```bash
ls -la /var/www/ptit-web/
ls -la /var/www/ptit-web/html/
```

Kết quả kiểm thử thực tế trong môi trường Nginx Linux dùng cho bài thực hành (kích thước và thời gian có thể khác trên máy triển khai):

```text
$ ls -la /var/www/ptit-web/
total 24
drwxr-s---    1 devops   www-data      4096 Sep 30 16:19 .
drwxr-xr-x    1 root     root          4096 Sep 30 16:19 ..
drwxr-s---    1 devops   www-data      4096 Sep 30 16:19 html

$ ls -la /var/www/ptit-web/html/
total 20
drwxr-s---    1 devops   www-data      4096 Sep 30 16:19 .
drwxr-s---    1 devops   www-data      4096 Sep 30 16:19 ..
-rw-r-----    1 devops   www-data       131 Sep 30 16:19 index.html
```

Các dấu hiệu cần kiểm tra:

- Owner và group là `devops www-data`.
- Thư mục hiển thị `drwxr-s---`; ký tự `s` là setgid và đồng thời group vẫn có quyền đi xuyên thư mục.
- File tĩnh hiển thị `-rw-r-----`.

## 6. Kiểm tra ghi file bằng user `devops`

Đăng nhập bằng `devops`:

```bash
sudo -iu devops
```

Trong shell của `devops`, thực hiện đúng phép thử của đề bài, không dùng `sudo`:

```bash
whoami
echo "Update" >> /var/www/ptit-web/html/index.html
tail -n 1 /var/www/ptit-web/html/index.html
exit
```

Kết quả kiểm thử:

```text
devops
Update
```

Có thể kiểm tra nhanh từ tài khoản quản trị mà không mở shell tương tác:

```bash
sudo -u devops sh -c 'echo "Update from devops" >> /var/www/ptit-web/html/index.html'
```

Phải đặt cả phép chuyển hướng `>>` bên trong `sh -c`. Lệnh `sudo -u devops echo ... >> file` không phải phép thử đúng vì shell hiện tại xử lý `>>` trước khi `sudo` chạy.

## 7. Kiểm tra Nginx có quyền đọc

Kiểm tra trực tiếp dưới quyền `www-data`:

```bash
sudo -u www-data sh -c '
  test -x /var/www/ptit-web &&
  test -x /var/www/ptit-web/html &&
  test -r /var/www/ptit-web/html/index.html &&
  test ! -w /var/www/ptit-web/html/index.html
' && echo "www-data can read the website and cannot modify it"
```

Kết quả kiểm thử:

```text
www-data can read the website and cannot modify it
```

Kiểm tra quyền trên toàn bộ đường dẫn:

```bash
namei -l /var/www/ptit-web/html/index.html
```

Ngoài `/var/www/ptit-web` và `html`, các thư mục cha `/var` và `/var/www` cũng phải cho phép Nginx đi xuyên qua (có bit `x` phù hợp).

Kiểm tra cấu hình và HTTP:

```bash
sudo nginx -t
curl -I http://127.0.0.1/
curl -sS http://127.0.0.1/ | tail
```

Trong môi trường kiểm thử tối giản, `wget -S -O - http://127.0.0.1:8080/` được dùng tương đương với `curl` để xem cả header và nội dung. Các dòng đầu ra chính:

```text
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful

HTTP/1.1 200 OK
Server: nginx/1.31.6
Content-Type: text/html

<!doctype html>
<html lang="vi">
<head><meta charset="utf-8"><title>PTIT Web</title></head>
<body><h1>PTIT Web</h1></body>
</html>
Update
```

Nếu virtual host sử dụng tên miền riêng, thêm header tương ứng, ví dụ:

```bash
curl -I -H 'Host: ptit-web.local' http://127.0.0.1/
```

Không cần reload Nginx khi chỉ đổi quyền filesystem; quyền mới có hiệu lực ngay. Chỉ chạy `sudo systemctl reload nginx` nếu đã thay đổi cấu hình Nginx.

## 8. Kết luận

Sau khi thiết lập:

- `devops` có thể cập nhật website mà không cần quyền root.
- Nginx có thể đọc file thông qua group `www-data` và không gặp lỗi `403 Forbidden` do thiếu quyền filesystem.
- Người dùng khác không có quyền truy cập nội dung website.
- Setgid giúp các nội dung tạo mới tiếp tục kế thừa đúng group `www-data`.
