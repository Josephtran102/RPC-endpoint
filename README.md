# I. RPC-endpoint config:

## 1. Tạo 1 subdomain trỏ về địa chỉ VPS:
Example: rpc.0gchain.josephtran.xyz
```
Type: A
Host: rpc.0gchain
Value: 80.240.29.172 (Public IP VPS)
TTL: Automatic
```
Check kiểm tra xem subdomain đã load chưa và trỏ đúng về IP VPS chưa:
```
nslookup rpc.0gchain.josephtran.xyz
```
## 2. Mở cổng cho kết nối từ bên ngoài tại file `config.toml`:
`laddr = "tcp://127.0.0.1:26657"` đổi thành `laddr = "tcp://0.0.0.0:26657"`

*!!!Restart service file: `sudo systemctl restart ogchain` để áp dụng cấu hình mới*

## 3. Cấu hình server proxy:
Để bảo mật và quản lý dễ dàng hơn, sử dụng một reverse proxy server như Nginx hoặc Apache để chuyển tiếp các yêu cầu từ subdomain tới RPC node.
Sử dụng Nginx để cấu hình một server.
### Cài đặt nginx:
```
sudo apt update
sudo apt install nginx
nginx -v
```
### Check port RPC đang listen `port: 26657`:
```
RPC="http://$(wget -qO- eth0.me)$(grep -A 3 "\[rpc\]" $HOME/.0gchain/config/config.toml | egrep -o ":[0-9]+")" && echo $RPC
```
### Tạo một server block mới:
Sau khi xác định rằng Nginx đã được cài đặt, tạo một server block mới bằng cách thêm một tệp cấu hình mới trong thư mục `/etc/nginx/sites-available`. 
Ví dụ, tạo một tệp mới có tên `rpc.0gchain.josephtran.xyz`
```
sudo nano /etc/nginx/sites-available/rpc.0gchain.josephtran.xyz
```
Dán nội dung vào như sau:
```
server {
    listen 80;
    server_name rpc.0gchain.josephtran.xyz;

    location / {
        proxy_pass http://localhost:26657;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```
Lưu và đóng tệp cấu hình sau khi hoàn thành.
Tiếp theo, cần tạo một liên kết từ tệp cấu hình trong `sites-available` đến `sites-enabled` để Nginx có thể đọc cấu hình mới:
```
sudo ln -s /etc/nginx/sites-available/rpc.0gchain.josephtran.xyz /etc/nginx/sites-enabled/
```
Kiểm tra cấu hình và load lại cấu hình:
```
sudo nginx -t
sudo systemctl reload nginx
```
Đến đây chỉ mới truy cập được `http` như: `http://rpc.0gchain.josephtran.xyz` để truy cập `https` phải đăng ký SSL.

## 4. Đăng ký SSL:
### Tạo folder chứa cặp key:
```
sudo mkdir -p /etc/nginx/ssl/
```
### Lệnh tạo key:
```
sudo openssl genpkey -algorithm RSA -out /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.key -pkeyopt rsa_keygen_bits:2048
```
Kết quả:
```
....+......+...+....+..+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*......+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*.+....+.....+....+.
........+......+.....+.........+......+.......+..+...+............+.......+.....+.........+
....+.................+....+............+.....+....+.....+..........+......+..+...+........
..........+....+..+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++
.........+......+.....+...+.+.....+...+.......+++++++++++++++++++++++++++++++++++++++++++++
++++++++++++++++++++*........................+..+...+.+..+.............+......+.....+.+...+
..............+.+++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++++*....+....
..+...+.......+.....+...+.+.....+....+......+..+...+.+......+........+.+........+..........
+...+..+...+.......+..+.....................+............+..........+...+..+......+.+......
+..+.+...........+.+..............+.+..++++++++++++++++++++++++++++++++++++++++++++++++++++
+++++++++++++
```
### Chạy tiếp lệnh sau để tạo CSR (Certificate Signing Request):
```
sudo openssl req -new -key /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.key -out /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.csr
```
Điền các thông tin cần thiết để đăng ký và kết quả như sau:
```
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [AU]:VN
State or Province Name (full name) [Some-State]:HoChiMinh
Locality Name (eg, city) []:HoChiMinh
Organization Name (eg, company) [Internet Widgits Pty Ltd]:JosephTran
Organizational Unit Name (eg, section) []:
Common Name (e.g. server FQDN or YOUR name) []:Joseph
Email Address []:josephtran@gmail.com

Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:JosephTran
 ⚡️ root@vultr  ~/L0  openssl x509 -req -days 365 -in /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.csr -signkey /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.key -out /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.crt

Certificate request self-signature ok
subject=C = VN, ST = HoChiMinh, L = HoChiMinh, O = JosephTran, CN = Joseph, emailAddress = josephtran0505@gmail.com
```
### Tạo chứng chỉ tự ký từ CSR:
```
sudo openssl x509 -req -days 365 -in /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.csr -signkey /etc/nginx/ssl/rpc.0gchain.josephtran.xyz -out /etc/nginx/ssl/rpc.0gchain.josephtran.xyz.crt
```
## 5. Đăng ký và cấu hình chứng chỉ SSL miễn phí từ Let's Encrypt trên máy chủ:
### Cài đặt `Certbot`:
Đầu tiên, cần cài đặt Certbot, một công cụ tự động hóa quá trình lấy chứng chỉ SSL từ Let's Encrypt. Certbot có thể được cài đặt trên hầu hết các hệ điều hành Linux thông qua gói phần mềm.
```
sudo apt update
sudo apt install certbot python3-certbot-nginx
```
### Lấy chứng chỉ SSL từ Let's Encrypt:
Sau khi cài đặt Certbot, bạn có thể sử dụng nó để lấy chứng chỉ SSL cho tên miền của bạn
```
sudo certbot --nginx -d rpc.0gchain.josephtran.xyz
```
### Tự động gia hạn chứng chỉ SSL:
Một khi đã có chứng chỉ SSL từ Let's Encrypt có thể cấu hình Certbot để tự động gia hạn chúng mỗi khi chúng hết hạn. Bạn có thể thêm một cron job để tự động chạy Certbot hàng tháng, như sau:
```
sudo crontab -e
```
*Chọn `1` để dùng nano mở file và chỉnh sửa, sau đó thêm dòng này vào cuối file:
```
0 0 1 * * /usr/bin/certbot renew --quiet
```
Sau khi đã thực hiện các bước này, trang web của bạn sẽ được cấu hình để sử dụng HTTPS với chứng chỉ SSL miễn phí từ Let's Encrypt.
### Cập nhật lại file cấu hình nginx:
```
server {
    if ($host = rpc.0gchain.josephtran.xyz) {
        return 301 https://$host$request_uri;
    } # managed by Certbot


    listen 80;
    server_name rpc.0gchain.josephtran.xyz;
    return 301 https://$host$request_uri;


}

server {
    listen 443 ssl;
    server_name rpc.0gchain.josephtran.xyz;
    ssl_certificate /etc/letsencrypt/live/rpc.0gchain.josephtran.xyz/fullchain.pem; # ma>
    ssl_certificate_key /etc/letsencrypt/live/rpc.0gchain.josephtran.xyz/privkey.pem; # >


    location / {
        proxy_pass http://localhost:26657;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

}
```
Reload lại nginx:
```
sudo nginx -t
sudo systemctl reload nginx
```
Đến đây nếu không listen được cổng 443 thì mở cổng 443 (nếu không muốn có port sau domain) hoặc tùy chọn `8443` và sửa lại cổng trong file nginx bên trên thành port `8443`
Lệnh mở port `8443`:
```
sudo iptables -A INPUT -p tcp --dport 8443 -j ACCEPT
```




# II. EVM-RPC endpoint:  

## Mở port: 8545 ở file `app.toml`:
```
nano $HOME/.0gchain/config/app.toml
```
Nội dung chỉnh sửa: đổi `address = "127.0.0.1:8545` thành `address = "0.0.0.0:8545` để có thể truy cập từ bên ngoài.
**WebSocket không cần mở vì khá tốn tài nguyên*
```
###############################################################################
###                           JSON RPC Configuration                        ###
###############################################################################

[json-rpc]

# Enable defines if the gRPC server should be enabled.
enable = true

# Address defines the EVM RPC HTTP server address to bind to.
#address = "127.0.0.1:8545"
address = "0.0.0.0:8545"**

# Address defines the EVM WebSocket server address to bind to.
ws-address = "127.0.0.1:8546"

# API defines a list of JSON-RPC namespaces that should be enabled
# Example: "eth,txpool,personal,net,debug,web3"
api = "eth,txpool,personal,net,debug,web3"

```
## Restart lại 0g service:
```
sudo systemctl restart ogd && sudo journalctl -u ogd -f -o cat
```

### Tương tự như các bước trỏ subdomain cho RPC Cosmos:
Tạo server block mới (tên `evm.rpc.0gchain.josephtran.xyz`), đăng ký SSL, cấu hình file nginx,...

File cấu hình nginx:

```
server {
    listen 443 ssl; # Lắng nghe cổng 443
    server_name evm.rpc.0gchain.josephtran.xyz;

    ssl_certificate /etc/letsencrypt/live/evm.rpc.0gchain.josephtran.xyz/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/evm.rpc.0gchain.josephtran.xyz/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

    location / {
        proxy_pass http://localhost:8545;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}

server {
    listen 80; # Lắng nghe cổng 80
    server_name evm.rpc.0gchain.josephtran.xyz;
    return 301 https://$host$request_uri; # Chuyển hướng yêu cầu HTTP sang HTTPS
}
```
Kiểm tra cấu hình và load lại cấu hình:
```
sudo nginx -t
sudo systemctl reload nginx
```
Nếu chưa được check xem cổng 443 đã mở chưa hoặc quy tắc tường lửa?:
```
sudo ufw status
```
Kết quả:
```
Status: active

To                         Action      From
--                         ------      ----
22/tcp                     ALLOW       Anywhere                  
26657                      ALLOW       Anywhere                  
80/tcp                     ALLOW       Anywhere                  
22656/tcp                  ALLOW       Anywhere                  
22657/tcp                  ALLOW       Anywhere                  
51656/tcp                  ALLOW       Anywhere                  
51657/tcp                  ALLOW       Anywhere                  
51658/tcp                  ALLOW       Anywhere                  
51660/tcp                  ALLOW       Anywhere                  
26656/tcp                  ALLOW       Anywhere                  
8545/tcp                   ALLOW       Anywhere                  
445/tcp                    ALLOW       Anywhere                  
443/tcp                    ALLOW       Anywhere                  
22/tcp (v6)                ALLOW       Anywhere (v6)             
26657 (v6)                 ALLOW       Anywhere (v6)             
80/tcp (v6)                ALLOW       Anywhere (v6)             
22656/tcp (v6)             ALLOW       Anywhere (v6)             
22657/tcp (v6)             ALLOW       Anywhere (v6)             
51656/tcp (v6)             ALLOW       Anywhere (v6)             
51657/tcp (v6)             ALLOW       Anywhere (v6)             
51658/tcp (v6)             ALLOW       Anywhere (v6)             
51660/tcp (v6)             ALLOW       Anywhere (v6)             
26656/tcp (v6)             ALLOW       Anywhere (v6)             
8545/tcp (v6)              ALLOW       Anywhere (v6)             
445/tcp (v6)               ALLOW       Anywhere (v6)             
443/tcp (v6)               ALLOW       Anywhere (v6)  
```
Nếu chưa mở thì có thể mở bằng:
```
sudo ufw allow 443/tcp
```
```
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT
```








