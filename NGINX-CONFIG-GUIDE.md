# คู่มือการตั้งค่า nginx.conf สำหรับระบบ JEval
## อธิบายการทำงานและการกำหนดค่าอย่างละเอียด

### 📋 สารบัญ
1. [ภาพรวมของการทำงาน](#ภาพรวมของการทำงาน)
2. [โครงสร้างไฟล์ nginx.conf](#โครงสร้างไฟล์-nginxconf)
3. [การตั้งค่าเบื้องต้น](#การตั้งค่าเบื้องต้น)
4. [การตั้งค่าความปลอดภัย](#การตั้งค่าความปลอดภัย)
5. [การตั้งค่า SSL/TLS](#การตั้งค่า-ssltls)
6. [การกำหนด Routes และ Locations](#การกำหนด-routes-และ-locations)
7. [การ Monitoring และ Logging](#การ-monitoring-และ-logging)
8. [ตัวอย่างการใช้งาน](#ตัวอย่างการใช้งาน)
9. [การแก้ไขปัญหา](#การแก้ไขปัญหา)

---

## ภาพรวมของการทำงาน

ระบบ nginx reverse proxy ของ JEval ทำหน้าที่เป็นจุดเข้าใช้งานหลัก (Entry Point) สำหรับ:

```
Internet → nginx (Port 80/443) → Backend Services
                    ↓
            ┌─────────────────┐
            │   nginx Proxy   │
            │   (eval-proxy)  │
            └─────────┬───────┘
                      │
         ┌────────────┼────────────┐
         ▼            ▼            ▼
    jeval-app:3000   jeval-strapi-app:1337
    (Frontend)       (Backend API)
```

**หน้าที่หลัก:**
- **SSL Termination**: จัดการ HTTPS และใบรับรอง SSL
- **Load Balancing**: กระจายงานไปยัง backend services
- **Security**: ป้องกันการโจมตีและจำกัดการเข้าถึง
- **Caching**: จัดเก็บไฟล์ static เพื่อเพิ่มประสิทธิภาพ
- **Rate Limiting**: จำกัดจำนวน requests ต่อเวลา

---

## โครงสร้างไฟล์ nginx.conf

### 🏗️ โครงสร้างหลัก

```nginx
# ส่วนที่ 1: การตั้งค่าเบื้องต้น
worker_processes auto;
worker_rlimit_nofile 8192;
pid /var/run/nginx.pid;

# ส่วนที่ 2: Events Block
events {
    worker_connections 1024;
    use epoll;
    multi_accept on;
}

# ส่วนที่ 3: HTTP Block
http {
    # การตั้งค่าพื้นฐาน
    # การตั้งค่าความปลอดภัย
    # การตั้งค่า SSL
    # การกำหนด Servers
}
```

---

## การตั้งค่าเบื้องต้น

### 🔧 Worker Configuration

```nginx
# กำหนดจำนวน worker processes ตามจำนวน CPU cores
worker_processes auto;

# จำกัดจำนวนไฟล์ที่ worker สามารถเปิดได้
worker_rlimit_nofile 8192;

# ตำแหน่งไฟล์ PID
pid /var/run/nginx.pid;
```

**การใช้งาน:**
- `auto`: nginx จะตั้งค่าให้เท่ากับจำนวน CPU cores
- สำหรับระบบ 3 users: 2-4 worker processes เพียงพอ
- `worker_rlimit_nofile`: ป้องกันปัญหาการเปิดไฟล์เกินขีดจำกัด

### ⚡ Events Configuration

```nginx
events {
    # จำนวน connections ต่อ worker
    worker_connections 1024;

    # ใช้ epoll สำหรับ Linux (ประสิทธิภาพสูง)
    use epoll;

    # รับ connections หลายๆ ตัวพร้อมกัน
    multi_accept on;
}
```

**คำอธิบาย:**
- **worker_connections**: จำนวน connections สูงสุดต่อ worker
- **epoll**: Event notification mechanism ที่เร็วที่สุดใน Linux
- **multi_accept**: ช่วยเพิ่มประสิทธิภาพการรับ connections

### 📁 Basic HTTP Settings

```nginx
http {
    # โหลด MIME types
    include /etc/nginx/mime.types;
    default_type application/octet-stream;
    charset utf-8;

    # การตั้งค่าประสิทธิภาพ
    sendfile on;                # ส่งไฟล์แบบ zero-copy
    tcp_nopush on;             # รวม packets ก่อนส่ง
    tcp_nodelay on;            # ส่งข้อมูลทันที
    keepalive_timeout 65;      # เวลา keep connections alive
    keepalive_requests 100;    # จำนวน requests ต่อ connection
}
```

---

## การตั้งค่าความปลอดภัย

### 🛡️ Security Headers

```nginx
# Security Headers ระดับ Global
add_header X-Frame-Options "SAMEORIGIN" always;
add_header X-Content-Type-Options "nosniff" always;
add_header X-XSS-Protection "1; mode=block" always;
add_header Referrer-Policy "strict-origin-when-cross-origin" always;
add_header Permissions-Policy "geolocation=(), microphone=(), camera=()" always;

# ซ่อนเวอร์ชันของ nginx
server_tokens off;
```

**ความหมายของแต่ละ Header:**

| Header | ความหมาย | ประโยชน์ |
|--------|-----------|----------|
| `X-Frame-Options` | ป้องกันการใส่เว็บใน iframe | ป้องกัน Clickjacking |
| `X-Content-Type-Options` | บังคับใช้ MIME type ตาม header | ป้องกัน MIME sniffing attacks |
| `X-XSS-Protection` | เปิดใช้ XSS filter ของ browser | ป้องกัน Cross-Site Scripting |
| `Referrer-Policy` | ควบคุมการส่ง referrer information | ป้องกันการรั่วไหลของข้อมูล |
| `Permissions-Policy` | จำกัดการใช้ browser features | ป้องกันการใช้งาน camera/microphone |

### 🚦 Rate Limiting

```nginx
# กำหนด Rate Limiting Zones
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;
limit_req_zone $binary_remote_addr zone=api:10m rate=30r/m;
limit_req_zone $binary_remote_addr zone=admin:10m rate=10r/m;
limit_req_zone $binary_remote_addr zone=upload:10m rate=5r/m;

# จำกัดจำนวน connections พร้อมกัน
limit_conn_zone $binary_remote_addr zone=conn_limit_per_ip:10m;
```

**การทำงาน:**
- **zone=api:10m**: สร้าง memory zone ขนาด 10MB สำหรับเก็บ IP addresses
- **rate=30r/m**: อนุญาต 30 requests ต่อนาที
- **$binary_remote_addr**: ใช้ IP address ในรูป binary (ประหยัด memory)

### 🚫 Attack Pattern Blocking

```nginx
# บล็อกรูปแบบการโจมตีทั่วไป
map $request_uri $is_blocked {
    default 0;
    "~*\.(php|asp|aspx|jsp)$" 1;                    # บล็อกไฟล์ script
    "~*/(wp-admin|wp-login|phpmyadmin|admin|cpanel)" 1;  # บล็อก admin paths
    "~*/\.(git|svn|htaccess|htpasswd)" 1;           # บล็อกไฟล์ระบบ
    "~*/(config|logs|backup|temp|tmp)" 1;           # บล็อก sensitive folders
}

# ใช้งานใน server block
if ($is_blocked) {
    return 444;  # ปิด connection ทันที
}
```

---

## การตั้งค่า SSL/TLS

### 🔐 SSL Configuration

```nginx
# โปรโตคอล SSL/TLS ที่รองรับ
ssl_protocols TLSv1.2 TLSv1.3;

# Cipher Suites ที่ปลอดภัย
ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;

# ให้ client เลือก cipher
ssl_prefer_server_ciphers off;

# การ cache SSL sessions
ssl_session_cache shared:SSL:10m;
ssl_session_timeout 10m;
ssl_session_tickets off;
```

**คำอธิบาย SSL Settings:**

| Setting | ความหมาย | ประโยชน์ |
|---------|-----------|----------|
| `TLSv1.2/1.3` | เวอร์ชัน TLS ที่รองรับ | ความปลอดภัยสูง |
| `ssl_ciphers` | อัลกอริทึมการเข้ารหัส | ป้องกัน weak ciphers |
| `ssl_session_cache` | เก็บ SSL sessions ใน memory | เพิ่มประสิทธิภาพ |
| `ssl_session_tickets off` | ปิด session tickets | เพิ่มความปลอดภัย |

### 📜 SSL Certificate Setup

```nginx
server {
    listen 443 ssl http2;
    server_name jeval.otepc.go.th;

    # SSL Certificate Files
    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    # HSTS Header
    add_header Strict-Transport-Security "max-age=31536000; includeSubDomains; preload" always;
}
```

---

## การกำหนด Routes และ Locations

### 🌐 Frontend Routes

```nginx
# Frontend Application - Next.js
location / {
    proxy_pass http://jeval-app:3000;
    proxy_http_version 1.1;

    # WebSocket Support
    proxy_set_header Upgrade $http_upgrade;
    proxy_set_header Connection 'upgrade';

    # Standard Headers
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;
    proxy_set_header X-Forwarded-Host $host;
    proxy_set_header X-Forwarded-Port $server_port;

    # Performance Settings
    proxy_cache_bypass $http_upgrade;
    proxy_redirect off;
    proxy_buffering off;

    # Timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
}
```

**Headers อธิบาย:**
- **X-Real-IP**: IP address ที่แท้จริงของ client
- **X-Forwarded-For**: รายการ IP addresses ที่ผ่าน proxies
- **X-Forwarded-Proto**: โปรโตคอลที่ใช้ (http/https)
- **X-Forwarded-Host**: hostname ที่ client ขอ

### 🔌 API Routes

```nginx
# API Routes - Strapi Backend
location /api {
    # Rate Limiting
    limit_req zone=api burst=10 nodelay;

    proxy_pass http://jeval-strapi-app:1337/api;
    proxy_http_version 1.1;

    # Headers (เหมือน frontend)
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # API-specific timeouts (เร็วกว่า frontend)
    proxy_connect_timeout 30s;
    proxy_send_timeout 30s;
    proxy_read_timeout 30s;
}
```

**Rate Limiting Parameters:**
- **zone=api**: ใช้ zone ที่กำหนดไว้
- **burst=10**: อนุญาตให้เกินขีดจำกัด 10 requests
- **nodelay**: ไม่หน่วงเวลาสำหรับ burst requests

### 🔒 Admin Routes (Protected)

```nginx
# Strapi Admin Panel (Restricted Access)
location /admin {
    limit_req zone=admin burst=5 nodelay;

    # IP Whitelist (กำหนดเฉพาะ IP ที่อนุญาต)
    deny all;
    # allow 192.168.1.100;  # IP ของ Admin คนที่ 1
    # allow 192.168.1.101;  # IP ของ Admin คนที่ 2
    # allow 192.168.1.102;  # IP ของ Admin คนที่ 3

    proxy_pass http://jeval-strapi-app:1337/admin;

    # เหมือน API routes แต่ timeout นานกว่า
    proxy_connect_timeout 60s;
    proxy_send_timeout 60s;
    proxy_read_timeout 60s;
}
```

### 📁 File Upload Routes

```nginx
# File Upload Endpoint
location /upload {
    limit_req zone=upload burst=3 nodelay;

    # เพิ่มขีดจำกัดสำหรับไฟล์ใหญ่
    client_max_body_size 100M;
    client_body_timeout 300;

    proxy_pass http://jeval-strapi-app:1337/upload;

    # Upload-specific timeouts
    proxy_connect_timeout 60s;
    proxy_send_timeout 300s;  # 5 นาที สำหรับไฟล์ใหญ่
    proxy_read_timeout 300s;  # 5 นาที สำหรับไฟล์ใหญ่
}
```

### 🏥 Health Check Routes

```nginx
# Health check endpoints
location /health {
    access_log off;  # ไม่บันทึก log สำหรับ health checks
    proxy_pass http://jeval-app:3000/health;
    proxy_set_header Host $host;
}

location /api/health {
    access_log off;
    proxy_pass http://jeval-strapi-app:1337/api/health;
    proxy_set_header Host $host;
}
```

### 📦 Static Assets Caching

```nginx
# Static assets caching
location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
    expires 1y;  # cache 1 ปี
    add_header Cache-Control "public, immutable";
    add_header X-Content-Type-Options "nosniff";

    # ลองหาจาก frontend ก่อน ถ้าไม่เจอให้ไปหา backend
    try_files $uri @backend_static;
}

location @backend_static {
    proxy_pass http://jeval-strapi-app:1337;
    proxy_set_header Host $host;
    expires 1y;
    add_header Cache-Control "public, immutable";
}
```

---

## การ Monitoring และ Logging

### 📊 Log Configuration

```nginx
# กำหนดรูปแบบ log
log_format main '$remote_addr - $remote_user [$time_local] "$request" '
               '$status $body_bytes_sent "$http_referer" '
               '"$http_user_agent" "$http_x_forwarded_for" '
               'rt=$request_time uct="$upstream_connect_time" '
               'uht="$upstream_header_time" urt="$upstream_response_time"';

# ตำแหน่งไฟล์ log
access_log /var/log/nginx/access.log main;
error_log /var/log/nginx/error.log warn;
```

**ข้อมูลใน Log:**
- **$remote_addr**: IP address ของ client
- **$request_time**: เวลาที่ใช้ประมวลผล request
- **$upstream_connect_time**: เวลาเชื่อมต่อกับ backend
- **$upstream_response_time**: เวลาตอบกลับจาก backend

### 📈 Performance Monitoring

```bash
# ดู access logs
docker compose exec eval-frontend tail -f /var/log/nginx/access.log

# ดู error logs
docker compose exec eval-frontend tail -f /var/log/nginx/error.log

# วิเคราะห์ response times
docker compose exec eval-frontend awk '{print $NF}' /var/log/nginx/access.log | grep "rt=" | tail -10

# ดู top requested URLs
docker compose exec eval-frontend awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10
```

---

## ตัวอย่างการใช้งาน

### 🔧 การเพิ่ม API Route ใหม่

หากต้องการเพิ่ม route `/api/v2` สำหรับ API เวอร์ชันใหม่:

```nginx
# เพิ่มใน server block
location /api/v2 {
    limit_req zone=api burst=15 nodelay;  # อนุญาต burst สูงกว่า

    proxy_pass http://jeval-strapi-app:1337/api/v2;
    proxy_http_version 1.1;
    proxy_set_header Host $host;
    proxy_set_header X-Real-IP $remote_addr;
    proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
    proxy_set_header X-Forwarded-Proto $scheme;

    # Timeout สั้นกว่าเพราะเป็น API ใหม่ที่เร็ว
    proxy_connect_timeout 15s;
    proxy_send_timeout 15s;
    proxy_read_timeout 15s;
}
```

### 🛡️ การเปิดใช้ IP Whitelist สำหรับ Admin

```nginx
# แก้ไขใน location /admin
location /admin {
    limit_req zone=admin burst=5 nodelay;

    # เปิด comment และใส่ IP ที่ต้องการ
    allow 192.168.1.100;    # IP ของ Admin คนที่ 1
    allow 192.168.1.101;    # IP ของ Admin คนที่ 2
    allow 172.16.0.50;      # IP ของ Admin คนที่ 3
    allow 10.0.0.25;        # IP backup (VPN หรือ office)
    deny all;               # ปฏิเสธที่เหลือ

    proxy_pass http://jeval-strapi-app:1337/admin;
    # ... (ส่วนที่เหลือเหมือนเดิม)
}
```

### ⚡ การปรับแต่ง Rate Limiting

สำหรับระบบที่มีผู้ใช้มากขึ้น:

```nginx
# ปรับแต่งใน http block
limit_req_zone $binary_remote_addr zone=api:10m rate=60r/m;    # เพิ่มจาก 30 เป็น 60
limit_req_zone $binary_remote_addr zone=admin:10m rate=20r/m;  # เพิ่มจาก 10 เป็น 20
limit_req_zone $binary_remote_addr zone=upload:10m rate=10r/m; # เพิ่มจาก 5 เป็น 10

# ปรับแต่งใน location blocks
location /api {
    limit_req zone=api burst=20 nodelay;  # เพิ่ม burst จาก 10 เป็น 20
    # ... (ส่วนที่เหลือ)
}
```

---

## การแก้ไขปัญหา

### ❌ ปัญหาที่พบบ่อยและวิธีแก้ไข

#### 1. 502 Bad Gateway

**สาเหตุ:**
- Backend services ไม่ทำงาน
- Container ไม่อยู่ใน network เดียวกัน
- Port หรือ service name ผิด

**วิธีแก้ไข:**
```bash
# ตรวจสอบ backend services
docker compose ps

# ตรวจสอบ network connectivity
docker compose exec eval-frontend curl http://jeval-app:3000
docker compose exec eval-frontend curl http://jeval-strapi-app:1337

# ตรวจสอบ logs
docker compose logs jeval-app
docker compose logs jeval-strapi-app
```

#### 2. 444 Response (Admin Access Blocked)

**สาเหตุ:**
- IP ไม่อยู่ใน whitelist
- Configuration ยังเป็น `deny all;`

**วิธีแก้ไข:**
```bash
# หา IP address ปัจจุบัน
curl ifconfig.me

# แก้ไข nginx.conf
# เปลี่ยน: deny all;
# เป็น: allow YOUR_IP; deny all;

# Apply changes
docker compose exec eval-frontend nginx -s reload
```

#### 3. SSL Certificate Errors

**สาเหตุ:**
- Certificate หมดอายุ
- Permission ของไฟล์ไม่ถูกต้อง
- ไฟล์ certificate เสียหาย

**วิธีแก้ไข:**
```bash
# ตรวจสอบ certificate
openssl x509 -in appdata/ssl/server.crt -text -noout | grep "Not After"

# ตรวจสอบ permissions
ls -la appdata/ssl/
# ควรเป็น: -rw-r--r-- server.crt, -rw------- server.key

# แก้ไข permissions
chmod 644 appdata/ssl/server.crt
chmod 600 appdata/ssl/server.key
```

#### 4. Rate Limiting Too Strict

**สาเหตุ:**
- Rate limit ต่ำเกินไป
- Burst size น้อย
- ผู้ใช้งานมากกว่าที่คาดไว้

**วิธีแก้ไข:**
```bash
# ดู rate limit logs
docker compose exec eval-frontend grep "limiting requests" /var/log/nginx/error.log

# ปรับแต่งใน nginx.conf
# เพิ่ม rate หรือ burst size

# Apply changes
docker compose exec eval-frontend nginx -s reload
```

#### 5. Slow Response Times

**สาเหตุ:**
- Backend services ช้า
- Network latency
- Database performance

**วิธีแก้ไข:**
```bash
# ตรวจสอบ response times จาก logs
docker compose exec eval-frontend awk '{print $NF}' /var/log/nginx/access.log | grep "rt=" | tail -20

# ตรวจสอบ upstream times
docker compose exec eval-frontend grep "urt=" /var/log/nginx/access.log | tail -10

# ปรับแต่ง timeouts ใน nginx.conf หากจำเป็น
proxy_connect_timeout 30s;
proxy_send_timeout 30s;
proxy_read_timeout 30s;
```

### 🔧 คำสั่งการบำรุงรักษา

```bash
# ทดสอบ configuration
docker compose exec eval-frontend nginx -t

# Reload configuration (ไม่หยุดบริการ)
docker compose exec eval-frontend nginx -s reload

# Restart service (หยุดบริการชั่วคราว)
docker compose restart eval-frontend

# ดู full configuration
docker compose exec eval-frontend nginx -T

# ตรวจสอบการใช้งาน memory
docker stats eval-proxy

# ตรวจสอบ disk usage ของ logs
docker compose exec eval-frontend du -sh /var/log/nginx/
```

### 📝 Checklist การตรวจสอบ

**รายวัน:**
- [ ] ตรวจสอบ error logs
- [ ] ตรวจสอบ response times
- [ ] ตรวจสอบ rate limiting violations

**รายสัปดาห์:**
- [ ] ตรวจสอบ disk space สำหรับ logs
- [ ] ตรวจสอบ SSL certificate expiry
- [ ] Review access patterns

**รายเดือน:**
- [ ] ปรับแต่ง rate limiting ตามการใช้งานจริง
- [ ] อัพเดท security configurations
- [ ] Review และ rotate log files

---

## สรุป

การตั้งค่า nginx.conf สำหรับระบบ JEval ครอบคลุมทั้งด้านความปลอดภัย, ประสิทธิภาพ และการ monitoring ที่เหมาะสมสำหรับทีมงาน 3 คน การกำหนดค่าแต่ละส่วนมีจุดประสงค์เฉพาะและสามารถปรับแต่งได้ตามความต้องการของระบบ

**จุดสำคัญที่ต้องจำ:**
1. **Security First**: ใช้ HTTPS, rate limiting และ IP whitelist
2. **Performance**: ใช้ caching, gzip และ proper timeouts
3. **Monitoring**: เก็บ logs และตรวจสอบสม่ำเสมอ
4. **Scalability**: สามารถปรับแต่งได้เมื่อผู้ใช้เพิ่มขึ้น

สำหรับการสนับสนุนเพิ่มเติม โปรดดูไฟล์ `SECURITY-SETUP.md` และ `DEPLOYMENT-GUIDE.md`