
## Hướng dẫn cài đặt và cấu hình Authelia Docker
### Bước 1: Chuẩn bị
Tạo thư mục Authelia và file Docker Compose
```
mkdir authelia
cd authelia
touch docker-compose.yml
```
## Bước 2: Cấu hình Docker Compose
Sao chép nội dung dưới đây vào file docker-compose.yml:
```
version: '3'
services:
  auth:
    container_name: auth
    image: authelia/authelia:latest
    ports:
      - "9091:9091"
    volumes:
      - ./config:/config
    labels:
      traefik.enable: true
      traefik.http.routers.authelia.entryPoints: https
    environment:
      - TZ=Asia/Ho_Chi_Minh
    networks:
      - proxy
    restart: unless-stopped
    depends_on:
      - redis
      - mariadb

  redis:
    container_name: redis
    image: bitnami/redis:latest
    ports:
      - "6379:6379"
    volumes:
      - ./redis:/bitnami/
    environment:
      - REDIS_PASSWORD="pass@redit"
      - TZ=Asia/Ho_Chi_Minh
    networks:
      - proxy
    restart: unless-stopped

  mariadb:
    container_name: mariadb
    image: linuxserver/mariadb:latest
    ports:
      - "3306:3306"
    volumes:
      - ./mysql:/config
    environment:
      - MYSQL_ROOT_PASSWORD="pass@123"
      - MYSQL_ROOT_USER=root
      - MYSQL_DATABASE=authelia
      - MYSQL_USER=authelia
      - MYSQL_PASSWORD="pass@123"
      - TZ=Asia/Ho_Chi_Minh
    networks:
      - proxy
    restart: unless-stopped

networks:
  proxy:
    driver: bridge
    external: true
```
## Bước 3: Cấu hình Authelia
Tạo file cấu hình configuration.yml
```
mkdir config
touch config/configuration.yml
```
Sao chép nội dung dưới đây vào file config/configuration.yml:
```
###############################################################################
#                           Authelia Configuration                            #
###############################################################################

theme: dark
jwt_secret: "Tạo 1 mã bảo mật ngẫu nhiên"
  #default_redirection_url: https://quantri.test.vn/PasswordVault/v10/logon/cyberark

server:
  host: 0.0.0.0
  port: 9091
  path: ""
  read_buffer_size: 4096
  write_buffer_size: 4096
  enable_pprof: false
  enable_expvars: false
  disable_healthcheck: false
  tls:
    key: ""
    certificate: ""
  endpoints:
    authz:
      forward-auth:
        implementation: 'ForwardAuth'

ntp:
  address: "172.16.106.23:123"
  version: 4
  max_desync: 3s
  disable_startup_check: false
  disable_failure: true


log:
  level: debug

totp:
  issuer: test.vn
  period: 30
  skew: 1

authentication_backend:
  ldap:
    implementation: activedirectory
    url: ldap://192.168.188.125:389
    timeout: 5s
    start_tls: false
    tls:
      skip_verify: true
      minimum_version: TLS1.2
    base_dn: DC=blt,DC=vn
    username_attribute: sAMAccountName
    users_filter: (&(|({username_attribute}={input})({mail_attribute}={input}))(objectCategory=person)(objectClass=user)(!userAccountControl:1.2.840.113556.1.4.803:=2)(!pwdLastSet=0))
    groups_filter: (&(member:1.2.840.113556.1.4.1941:={dn})(objectClass=group)(objectCategory=group))
    group_name_attribute: cn
    mail_attribute: mail
    display_name_attribute: displayName
    user: CN=Authelia,OU=ServiceUsers,OU=BLTUsers,DC=AD,DC=vn
    password: authelia@123456

#Sử dụng user không xác thực ldap
#  password_reset:
#    disable: false
#  refresh_interval: 5m
#  file:
#    path: /config/users_database.yml
#    password:
#      algorithm: argon2id
#      iterations: 1
#      key_length: 32
#      salt_length: 16
#      memory: 1024
#      parallelism: 8

access_control:
  default_policy: deny
  networks:
  - name: 'internal'
    networks:
    - '172.16.0.0/12'
    - '192.168.0.0/18'
  rules:
    - domain:
        - "quantri.test.vn"
      resources:
        - '^/([/?].*)?$'
        - '^/PasswordVault([/?].*)?$'
        - '^/passwordvault/favicon.ico([/?].*)?$'
        - '^/favicon.ico([/?].*)?$'
        - '^/iisstart.png([/?].*)?$'
        - '^/Test([/?].*)?$'
      subject: 
        - "group:HT Group"
      methods:
        - 'GET'
        - 'POST'
        - 'HEAD'
      policy: two_factor

    - domain:
        - "webtest.test.vn"
      policy: bypass

session:
  name: authelia_session
  same_site: lax
  secret: "Tạo 1 mã bảo mật ngẫu nhiên"
  expiration: 1h
  inactivity: 5m
  remember_me_duration: 2M
  cookies:
    - domain: 'test.vn'
      authelia_url: 'https://admin-stb.test.vn'
        #default_redirection_url: 'https://quantri.test.vn/PasswordVault/v10/logon/cyberark'
        #default_redirection_url: 'https://quantri.test.vn/PasswordVault'
      #default_redirection_url: 'https://quantri.test.vn'
      name: 'authelia_session'
      same_site: 'lax'
      inactivity: '5m'
      expiration: '1h'
      remember_me: '1d'

  redis:
    host: redis
    port: 6379
    password: "Test@redit"
    database_index: 0
    maximum_active_connections: 50
    minimum_idle_connections: 0

regulation:
  max_retries: 5
  find_time: 10m
  ban_time: 12h

storage:
  encryption_key: "Tạo 1 mã bảo mật ngẫu nhiên"
  mysql:
    host: mariadb
    port: 3306
    database: authelia
    username: authelia
    password: "Testdb@123"

notifier:
  filesystem:
    filename: /config/notification.txt
# Cấu hình sau:	
#  smtp: 
#    username: ''
#    password: ''
#    host: ''
#    port: 25
#    sender: "Authelia"
#    subject: "[Authelia] {title}"
#    disable_require_tls: true
#    disable_starttls: false
#   disable_html_emails: false
#    startup_check_address: ''
#    tls:
#      server_name: ''
#      skip_verify: true
#      minimum_version: 'TLS1.2'
#      maximum_version: 'TLS1.3'
```

*lưu ý: Tạo các phần key secret:*

jwt_secret

secret

encryption_key 

**Tạo bằng lệnh sau:**
```
tr -cd '[:alnum:]' < /dev/urandom | fold -w "64" | head -n 1
Kết quả:
i3HW9Sjwbra4cZnnGroCgvmpz9gDm2YMDAlBWNGYJZ2TpNRNAXqHLxSvDR140uec
```
Thực hiện copy secret key vào các trường tương ứng, để bảo mật hơn có thể tạo 3 key secret khác nhau.

### Bước 4: Tạo file notification.txt trong config
```
touch notification.txt
```
(File này dùng để nhận các thông báo từ authelia gửi ra khi chưa cấu hình gửi mail smtp)

### Bước 5: Bật dịch vụ
```
cd /authelia/
docker compose up -d or docker-compose up -d.
```

### Bước 6: Kiểm tra các container đã start thành công chưa
```
docker ps -a
f027c6ba66f4   authelia/authelia:latest     "/app/entrypoint.sh"     2 days ago    Up 45 hours (healthy)   0.0.0.0:9091->9091/tcp, :::9091->9091/tcp   auth
250cb3e1d9c8   linuxserver/mariadb:latest   "/init"                  2 days ago    Up 2 days               0.0.0.0:3306->3306/tcp, :::3306->3306/tcp   mariadb
d37e0863dcd3   bitnami/redis:latest         "/opt/bitnami/script…"   2 days ago    Up 2 days               0.0.0.0:6379->6379/tcp, :::6379->6379/tcp   redis
```  
  
**Xem thêm tài liệu cấu hình để tích hợp với Haproxy**