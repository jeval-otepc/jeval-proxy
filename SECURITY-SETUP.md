# JEval Security Setup Guide
## การตั้งค่าความปลอดภัยสำหรับระบบ 3 ผู้ใช้งาน

### 🔒 การตั้งค่า IP Whitelist สำหรับ Admin Access

**ขั้นตอนที่ 1: เปิดใช้งาน IP Restrictions**

แก้ไขไฟล์ `appdata/nginx/nginx.conf` และเปิด comment ในส่วนต่อไปนี้:

```nginx
# ในส่วน location /admin, /content-manager, /content-type-builder
# เปลี่ยนจาก:
deny all;
# allow 192.168.1.100;  # Replace with admin IP

# เป็น:
# deny all;  # comment บรรทัดนี้
allow 192.168.1.100;    # IP ของ Admin คนที่ 1
allow 192.168.1.101;    # IP ของ Admin คนที่ 2
allow 192.168.1.102;    # IP ของ Admin คนที่ 3
deny all;               # ปฏิเสธที่เหลือ
```

**ขั้นตอนที่ 2: หา IP Address ของผู้ใช้งาน**

```bash
# ในเครื่องผู้ใช้งาน:
curl ifconfig.me
# หรือ
ip addr show

# ใน Docker container:
docker-compose exec eval-frontend cat /var/log/nginx/access.log | tail -10
```

### 🛡️ การตั้งค่า SSL Certificate

**Option 1: Self-Signed Certificate (สำหรับ Internal Use)**

```bash
# สร้าง SSL Certificate ใหม่
cd appdata/ssl

# สร้าง Private Key
openssl genrsa -out server.key 2048

# สร้าง Certificate Request
openssl req -new -key server.key -out server.csr -config <(
cat <<EOF
[req]
distinguished_name = req_distinguished_name
req_extensions = v3_req
prompt = no

[req_distinguished_name]
C=TH
ST=Bangkok
L=Bangkok
O=OTEPC
OU=IT Department
CN=jeval.otepc.go.th

[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = jeval.otepc.go.th
DNS.2 = localhost
IP.1 = 127.0.0.1
EOF
)

# สร้าง Self-Signed Certificate
openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt -extensions v3_req -extfile <(
cat <<EOF
[v3_req]
keyUsage = keyEncipherment, dataEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = jeval.otepc.go.th
DNS.2 = localhost
IP.1 = 127.0.0.1
EOF
)

# ตั้งสิทธิ์ไฟล์
chmod 600 server.key
chmod 644 server.crt
```

**Option 2: Let's Encrypt (สำหรับ Production)**

```bash
# ติดตั้ง Certbot
sudo apt update
sudo apt install certbot

# สร้าง Certificate
sudo certbot certonly --standalone -d jeval.otepc.go.th

# Copy Certificate ไปยัง nginx directory
sudo cp /etc/letsencrypt/live/jeval.otepc.go.th/fullchain.pem appdata/ssl/server.crt
sudo cp /etc/letsencrypt/live/jeval.otepc.go.th/privkey.pem appdata/ssl/server.key

# ตั้งสิทธิ์
sudo chown $(whoami):$(whoami) appdata/ssl/server.*
chmod 600 appdata/ssl/server.key
chmod 644 appdata/ssl/server.crt
```

### 🔐 การตั้งค่า Rate Limiting

การตั้งค่าปัจจุบันเหมาะสำหรับ 3 ผู้ใช้:

```nginx
# API Requests: 30 requests/minute
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;

# Admin Access: 10 requests/minute
limit_req_zone $binary_remote_addr zone=admin:10m rate=10r/m;

# File Upload: 5 requests/minute
limit_req_zone $binary_remote_addr zone=upload:10m rate=5r/m;

# Connection Limit: 10 concurrent connections per IP
limit_conn conn_limit_per_ip 10;
```

**การปรับแต่งสำหรับผู้ใช้งานมากขึ้น:**

```nginx
# เพิ่มเป็น 50 requests/minute สำหรับ API
limit_req_zone $binary_remote_addr zone=api:10m rate=50r/m;

# เพิ่มเป็น 15 requests/minute สำหรับ Admin
limit_req_zone $binary_remote_addr zone=admin:10m rate=15r/m;
```

### 🚨 Security Monitoring

**ขั้นตอนที่ 1: ดู Log Files**

```bash
# ดู Access Logs
docker-compose exec eval-frontend tail -f /var/log/nginx/access.log

# ดู Error Logs
docker-compose exec eval-frontend tail -f /var/log/nginx/error.log

# Filter สำหรับ Failed Requests
docker-compose exec eval-frontend grep " 4[0-9][0-9] " /var/log/nginx/access.log

# Filter สำหรับ Blocked Requests
docker-compose exec eval-frontend grep " 444 " /var/log/nginx/access.log
```

**ขั้นตอนที่ 2: Setup Log Rotation**

สร้างไฟล์ `appdata/logrotate/nginx`:

```bash
/var/log/nginx/*.log {
    daily
    missingok
    rotate 7
    compress
    delaycompress
    notifempty
    create 644 nginx nginx
    postrotate
        if [ -f /var/run/nginx.pid ]; then
            kill -USR1 `cat /var/run/nginx.pid`
        fi
    endscript
}
```

### 🔧 การทดสอบ Configuration

**ขั้นตอนที่ 1: ทดสอบ Syntax**

```bash
# ทดสอบ nginx configuration
docker-compose exec eval-frontend nginx -t

# ดู configuration ทั้งหมด
docker-compose exec eval-frontend nginx -T
```

**ขั้นตอนที่ 2: ทดสอบ SSL**

```bash
# ทดสอบ SSL Certificate
openssl s_client -connect jeval.otepc.go.th:443 -servername jeval.otepc.go.th

# ทดสอบ SSL Rating
curl -I https://jeval.otepc.go.th
```

**ขั้นตอนที่ 3: ทดสอบ Security Headers**

```bash
# ทดสอบ Security Headers
curl -I https://jeval.otepc.go.th

# ควรเห็น Headers เหล่านี้:
# Strict-Transport-Security: max-age=31536000; includeSubDomains; preload
# X-Frame-Options: SAMEORIGIN
# X-Content-Type-Options: nosniff
# X-XSS-Protection: 1; mode=block
```

### 🚀 การ Deploy และ Restart

**ขั้นตอนที่ 1: Apply Configuration Changes**

```bash
# Backup configuration ก่อน apply
cp appdata/nginx/nginx.conf appdata/nginx/nginx.conf.backup.$(date +%Y%m%d_%H%M%S)

# ทดสอบ configuration
docker-compose exec eval-frontend nginx -t

# Apply changes
docker-compose restart eval-frontend

# หรือ reload อย่างเดียว (ไม่ restart)
docker-compose exec eval-frontend nginx -s reload
```

**ขั้นตอนที่ 2: ตรวจสอบสถานะ**

```bash
# ตรวจสอบ container status
docker-compose ps

# ตรวจสอบ logs
docker-compose logs -f eval-frontend

# ทดสอบ endpoints
curl -k https://jeval.otepc.go.th/health
curl -k https://jeval.otepc.go.th/api/health
```

### ⚡ Performance Tuning สำหรับ 3 Users

**การปรับแต่งที่แนะนำ:**

```nginx
# Worker processes (ปรับตาม CPU cores)
worker_processes 2;

# Worker connections (เพียงพอสำหรับ 3 users)
worker_connections 512;

# File upload limit (ปรับตามความต้องการ)
client_max_body_size 50M;  # สำหรับไฟล์ทั่วไป
# ในส่วน /upload ตั้งเป็น 100M สำหรับไฟล์ใหญ่

# Connection timeout
keepalive_timeout 65;
```

### 🔍 Troubleshooting Common Issues

**Problem: 502 Bad Gateway**
```bash
# ตรวจสอบ backend services
docker-compose ps
docker-compose logs jeval-app
docker-compose logs jeval-strapi-app

# ทดสอบการเชื่อมต่อ
docker-compose exec eval-frontend curl http://jeval-app:3000
docker-compose exec eval-frontend curl http://jeval-strapi-app:1337
```

**Problem: SSL Certificate Errors**
```bash
# ตรวจสอบ certificate files
ls -la appdata/ssl/
openssl x509 -in appdata/ssl/server.crt -text -noout

# ตรวจสอบ permissions
chmod 600 appdata/ssl/server.key
chmod 644 appdata/ssl/server.crt
```

**Problem: Rate Limiting Too Strict**
```bash
# ดู rate limit logs
docker-compose exec eval-frontend grep "limiting requests" /var/log/nginx/error.log

# ปรับแต่งใน nginx.conf
# เพิ่ม burst size หรือ rate
limit_req zone=api burst=20 nodelay;  # เพิ่มจาก 10 เป็น 20
```

### 📋 Security Checklist

- [ ] IP Whitelist configured สำหรับ /admin paths
- [ ] SSL Certificate installed และ valid
- [ ] Rate limiting configured เหมาะสมกับจำนวนผู้ใช้
- [ ] Security headers enabled
- [ ] Sensitive file access blocked
- [ ] Log monitoring setup
- [ ] Regular security updates planned
- [ ] Backup procedures documented

### 🔄 Maintenance Schedule

**รายสัปดาห์:**
- ตรวจสอบ access logs สำหรับ unusual activity
- Verify SSL certificate expiry date

**รายเดือน:**
- Update nginx configuration if needed
- Review and rotate log files
- Check system resource usage

**ทุก 3 เดือน:**
- Update SSL certificates (if using Let's Encrypt)
- Review and update IP whitelist
- Security configuration audit