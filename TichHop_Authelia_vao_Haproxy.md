## Hướng dẫn cài đặt và tích hợp Authelia với HAProxy trên Ubuntu

### Chuẩn bị

- **Một máy chủ Ubuntu 22.04.4 LTS (hoặc mới hơn)**.
- **Máy chủ đã cài đặt Docker và Docker Compose**.

### Bước 1: Cài đặt các gói cần thiết

```bash
apt-get update --fix-missing
apt-get upgrade
apt install wget screen lsof rsync nmap net-tools unzip sudo sysstat epel-release perl-core vim open-vm-tools iotop git -y
apt install htop -y
```

**Tải các file lua cần thiết:**  
**haproxy-auth-request**: <a href="https://github.com/haproxytech/haproxy-lua-http" target="_blank">link-down</a>

**lua-json**: <a href="https://github.com/rxi/json.lua/blob/master/json.lua" target="_blank">link-down</a>

**haproxy-auth-request**: <a href="https://github.com/TimWolla/haproxy-auth-request/blob/main/auth-request.lua" target="_blank">link-down</a>


### Bước 2: Cài đặt HAProxy
```bash
add-apt-repository ppa:vbernat/haproxy-2.8 -y
curl https://haproxy.debian.net/bernat.debian.org.gpg | gpg --dearmor > /usr/share/keyrings/haproxy.debian.net.gpg
echo deb "[signed-by=/usr/share/keyrings/haproxy.debian.net.gpg]" https://haproxy.debian.net bookworm-backports-2.8 main > /etc/apt/sources.list.d/haproxy.list
apt update
apt-get install haproxy

```
Sau khi cài đặt xong, kiểm tra phiên bản HAProxy:
Kết quả sẽ hiển thị phiên bản HAProxy, ví dụ:
```bash
haproxy -v
HAProxy version 2.8.9-1ppa1~jammy 2024/04/06
```
### Bước 3: Cài đặt Lua cho Ubuntu
```bash
apt install lua5.3 lua-json -y
```
Kiểm tra phiên bản Lua:
```bash
lua -v
```
Nếu kết quả hiển thị phiên bản Lua, việc cài đặt đã thành công.

### Bước 4: Kiểm tra Lua
Tạo một file Lua với tên my-lua.lua và nội dung sau:
```
local json = require "json"
print("hello world")
```
Sau đó, kiểm tra Lua bằng cách chạy file vừa tạo:
Nếu kết quả in ra "hello world", Lua đã hoạt động chính xác.
```bash
lua my-lua.lua
hello world
```
### Bước 5: Cấu hình HAProxy cho Authelia
Thêm các dòng sau vào cấu hình **global** của file **haproxy.conf**:
```bash
lua-load /usr/share/lua/5.3/json.lua
lua-prepend-path /etc/haproxy/lua/http.lua
lua-load /etc/haproxy/lua/auth-request.lua
```
*Chú ý: Đảm bảo đường dẫn là chính xác.*

### Bước 6: Thêm cấu hình xác thực Authelia vào HAProxy
Thêm cấu hình xác thực Authelia vào haproxy:
Cấu hình được thêm vào frontend: (Xem thêm hướng dẫn: <a href="https://www.authelia.com/integration/proxies/haproxy/" target="_blank">link authelia</a>)
VD: Cấu hình xác thực cho 2 domain (quantri|webtest).test.vn
```bash
### Authelia-Start-Config ###

# Trusted Proxies
http-request del-header X-Forwarded-For
acl hdr-xff_exists req.hdr(X-Forwarded-For) -m found
http-request set-header X-Forwarded-For %[src] if !hdr-xff_exists
option forwardfor

# Host ACLs Authelia
acl protected-frontends hdr(host) -m reg -i ^(?i)(quantri|webtest)\.test\.vn
acl host-authelia hdr(host) -i admin-stb.test.vn
acl host-test hdr(host) -i quantri.test.vn
acl host-webtest hdr(host) -i webtest.test.vn
acl is_root path /

http-request set-var(req.scheme) str(https) if { ssl_fc }
http-request set-var(req.scheme) str(http) if !{ ssl_fc }
http-request set-var(req.questionmark) str(?) if { query -m found }

# Required Headers
http-request set-header X-Forwarded-Method %[method]
http-request set-header X-Forwarded-Proto %[var(req.scheme)]
http-request set-header X-Forwarded-Host %[req.hdr(Host)]
http-request set-header X-Forwarded-URI %[path]%[var(req.questionmark)]%[query]

# Chỉ chuyển hướng người dùng chưa xác thực tới trang xác thực Authelia
http-request lua.auth-intercept authelia_backend /api/authz/forward-auth HEAD * remote-user,remote-groups,remote-name,remote-email - if protected-frontends
http-request deny if protected-frontends !{ var(txn.auth_response_successful) -m bool } { var(txn.auth_response_code) -m int 403 }
http-request redirect location %[var(txn.auth_response_location)] if protected-frontends !{ var(txn.auth_response_successful) -m bool }

# Chuyển hướng domain theo patch
http-request redirect location https://quantri.test.vn code 301 if host-test is_root

### Authelia-End-Config ###

### Authelia-Backend-start ###

backend authelia_backend
    mode http
    balance roundrobin
    server authelia 172.16.207.125:9091 check

backend cyberark_backend
    mode http
    balance roundrobin
    redirect scheme https if !{ ssl_fc }
    acl set_cookie_exist var(req.auth_response_header.set_cookie) -m found
    http-response set-header Set-Cookie %[var(req.auth_response_header.set_cookie)] if set_cookie_exist
    server cyberark 172.16.207.124:443 ssl verify none

backend webtest_backend
    mode http
    balance roundrobin
    acl set_cookie_exist var(req.auth_response_header.set_cookie) -m found
    http-response set-header Set-Cookie %[var(req.auth_response_header.set_cookie)] if set_cookie_exist
    server webtest 172.16.207.125:8000 check

### Authelia-Backend-end ###
```

### Bước 7: Khởi động lại HAProxy
```bash
systemctl restart haproxy
```

