# JEval Proxy Deployment Guide
## คู่มือการ Deploy nginx reverse proxy อย่างปลอดภัย

### 🚀 Quick Start Deployment

**ขั้นตอนที่ 1: เตรียมระบบ**

```bash
# เช็คสถานะ SSL Certificate ที่มีอยู่
ls -la appdata/ssl/
openssl x509 -in appdata/ssl/server.crt -text -noout | grep "Not After"

# Backup configuration เดิม (ทำแล้ว)
ls -la appdata/nginx/nginx.conf.backup

# ตรวจสอบ Docker networks
docker network ls | grep -E "(jeval-network|Strapi)"
```

**ขั้นตอนที่ 2: ทดสอบ Configuration**

```bash
# ทดสอบ nginx syntax ก่อน deploy
docker-compose exec eval-frontend nginx -t

# หากยังไม่มี container ให้ start ขึ้นมาก่อน
docker-compose up -d

# ทดสอบอีกครั้ง
docker-compose exec eval-frontend nginx -t
```

**ขั้นตอนที่ 3: Apply Changes**

```bash
# Restart nginx service
docker-compose restart eval-frontend

# ตรวจสอบสถานะ
docker-compose ps
docker-compose logs -f eval-frontend
```

### 🧪 การทดสอบระบบอย่างครอบคลุม

**1. ทดสอบ Basic Connectivity**

```bash
# ทดสอบ HTTP redirect to HTTPS
curl -I http://jeval.otepc.go.th
# ควรได้ 301 redirect

# ทดสอบ HTTPS
curl -k -I https://jeval.otepc.go.th
# ควรได้ 200 OK

# ทดสอบ Frontend
curl -k https://jeval.otepc.go.th/ | head -20

# ทดสอบ API
curl -k https://jeval.otepc.go.th/api/ | head -10
```

**2. ทดสอบ Security Headers**

```bash
# ตรวจสอบ Security Headers
curl -k -I https://jeval.otepc.go.th | grep -E "(X-Frame-Options|X-Content-Type-Options|Strict-Transport-Security)"

# ทดสอบ Rate Limiting
for i in {1..35}; do curl -k -w "%{http_code}\n" -o /dev/null -s https://jeval.otepc.go.th/api/; done
# หลัง request ที่ 30 ควรเริ่มได้ 429 (Too Many Requests)
```

**3. ทดสอบ Admin Access (ขณะนี้ deny all)**

```bash
# ทดสอบ admin access (ควรได้ 403)
curl -k -I https://jeval.otepc.go.th/admin
curl -k -I https://jeval.otepc.go.th/content-manager
curl -k -I https://jeval.otepc.go.th/content-type-builder
```

**4. ทดสอบ File Upload**

```bash
# สร้างไฟล์ทดสอบ
echo "test content" > test.txt

# ทดสอบ upload (ต้องมี authentication)
curl -k -X POST -F "file=@test.txt" https://jeval.otepc.go.th/upload

# ลบไฟล์ทดสอบ
rm test.txt
```

### 🔧 การตั้งค่า IP Whitelist สำหรับ Admin

**ขั้นตอนที่ 1: หา IP Address ของ Admin Users**

```bash
# วิธีที่ 1: ใช้ curl (จากเครื่องของ admin)
curl ifconfig.me

# วิธีที่ 2: จาก nginx logs
docker-compose exec eval-frontend tail -20 /var/log/nginx/access.log

# วิธีที่ 3: ใช้ ip command (สำหรับ local network)
ip route get 8.8.8.8 | awk '{print $7}'
```

**ขั้นตอนที่ 2: แก้ไข nginx.conf**

แก้ไขไฟล์ `appdata/nginx/nginx.conf` ในส่วน admin locations:

```bash
# เปิดไฟล์
nano appdata/nginx/nginx.conf

# หาส่วน location /admin และเปลี่ยนจาก:
deny all;
# allow 192.168.1.100;  # Replace with admin IP

# เป็น:
allow 192.168.1.100;    # IP ของ Admin คนที่ 1
allow 192.168.1.101;    # IP ของ Admin คนที่ 2
allow 192.168.1.102;    # IP ของ Admin คนที่ 3
deny all;               # ปฏิเสธที่เหลือ
```

**ขั้นตอนที่ 3: Apply IP Whitelist**

```bash
# ทดสอบ configuration
docker-compose exec eval-frontend nginx -t

# Apply changes
docker-compose exec eval-frontend nginx -s reload

# ทดสอบจาก IP ที่อนุญาต
curl -k -I https://jeval.otepc.go.th/admin
# ควรได้ response ปกติ (ไม่ใช่ 403)
```

### 📊 Monitoring และ Logs

**1. Real-time Monitoring**

```bash
# ดู logs แบบ real-time
docker-compose logs -f eval-frontend

# ดู nginx access logs
docker-compose exec eval-frontend tail -f /var/log/nginx/access.log

# ดู error logs
docker-compose exec eval-frontend tail -f /var/log/nginx/error.log

# Filter เฉพาะ errors
docker-compose exec eval-frontend grep -E "(4[0-9][0-9]|5[0-9][0-9])" /var/log/nginx/access.log | tail -10
```

**2. Security Monitoring**

```bash
# ดู blocked requests
docker-compose exec eval-frontend grep " 444 " /var/log/nginx/access.log

# ดู rate limited requests
docker-compose exec eval-frontend grep "limiting requests" /var/log/nginx/error.log

# ดู failed admin access attempts
docker-compose exec eval-frontend grep "/admin" /var/log/nginx/access.log | grep " 403 "
```

**3. Performance Monitoring**

```bash
# ดู response times
docker-compose exec eval-frontend awk '{print $NF}' /var/log/nginx/access.log | grep -E "rt=[0-9]" | tail -10

# ดู top requested URLs
docker-compose exec eval-frontend awk '{print $7}' /var/log/nginx/access.log | sort | uniq -c | sort -nr | head -10
```

### 🚨 Troubleshooting Common Issues

**Problem 1: 502 Bad Gateway**

```bash
# ตรวจสอบ backend services
docker-compose ps

# ตรวจสอบ backend logs
docker-compose logs jeval-app
docker-compose logs jeval-strapi-app

# ทดสอบ connectivity จาก proxy
docker-compose exec eval-frontend curl http://jeval-app:3000
docker-compose exec eval-frontend curl http://jeval-strapi-app:1337

# แก้ไข: Restart backend services
docker-compose restart jeval-app
docker-compose restart jeval-strapi-app
```

**Problem 2: SSL Errors**

```bash
# ตรวจสอบ certificate
openssl x509 -in appdata/ssl/server.crt -text -noout | grep -E "(Subject|Not After)"

# ตรวจสอบ permissions
ls -la appdata/ssl/
# ควรเป็น:
# -rw-r--r-- server.crt
# -rw------- server.key

# แก้ไข permissions หากจำเป็น
chmod 644 appdata/ssl/server.crt
chmod 600 appdata/ssl/server.key
```

**Problem 3: Rate Limiting Too Strict**

```bash
# ดู rate limit errors
docker-compose exec eval-frontend grep "limiting requests" /var/log/nginx/error.log

# ปรับแต่งใน nginx.conf (เพิ่ม burst size)
# จาก: limit_req zone=api burst=10 nodelay;
# เป็น: limit_req zone=api burst=20 nodelay;

# หรือเพิ่ม rate
# จาก: rate=30r/m
# เป็น: rate=50r/m
```

**Problem 4: Admin Access Denied**

```bash
# ตรวจสอบ IP ปัจจุบัน
curl ifconfig.me

# ตรวจสอบ configuration
docker-compose exec eval-frontend nginx -T | grep -A5 -B5 "location /admin"

# ดู access logs สำหรับ admin
docker-compose exec eval-frontend grep "/admin" /var/log/nginx/access.log | tail -5

# Temporary fix: อนุญาต IP ชั่วคราว
# เพิ่มใน nginx.conf: allow YOUR_IP_HERE;
```

### ⚡ Performance Optimization

**1. สำหรับระบบ 3 users**

```nginx
# แนะนำค่าใน nginx.conf:
worker_processes 2;
worker_connections 512;
keepalive_timeout 65;
client_max_body_size 50M;
```

**2. การ Cache Static Files**

Configuration ปัจจุบันมี static file caching แล้ว:

```bash
# ทดสอบ caching headers
curl -k -I https://jeval.otepc.go.th/favicon.ico
# ควรเห็น: Cache-Control: public, immutable
```

**3. Gzip Compression**

```bash
# ทดสอบ gzip
curl -k -H "Accept-Encoding: gzip" https://jeval.otepc.go.th/ -v 2>&1 | grep "Content-Encoding"
# ควรเห็น: Content-Encoding: gzip
```

### 📋 Deployment Checklist

**Pre-Deployment:**
- [ ] Backup current nginx.conf
- [ ] Test new configuration syntax
- [ ] Verify SSL certificates exist and valid
- [ ] Check backend services are running
- [ ] Document IP addresses for whitelist

**Deployment:**
- [ ] Apply new configuration
- [ ] Test nginx syntax
- [ ] Restart/reload nginx
- [ ] Verify all services start successfully

**Post-Deployment Testing:**
- [ ] Test HTTP to HTTPS redirect
- [ ] Test frontend access
- [ ] Test API endpoints
- [ ] Test security headers
- [ ] Test rate limiting
- [ ] Test admin access (should be blocked initially)
- [ ] Configure IP whitelist
- [ ] Test admin access from allowed IPs
- [ ] Monitor logs for errors

**Final Verification:**
- [ ] All endpoints responding correctly
- [ ] SSL working properly
- [ ] Security headers present
- [ ] Rate limiting functional
- [ ] Admin access restricted to allowed IPs
- [ ] Log monitoring active
- [ ] Performance acceptable

### 🔄 Rollback Procedure

หากเกิดปัญหา สามารถ rollback ได้:

```bash
# Restore backup configuration
cp appdata/nginx/nginx.conf.backup appdata/nginx/nginx.conf

# Test configuration
docker-compose exec eval-frontend nginx -t

# Apply
docker-compose restart eval-frontend

# Verify
curl -k -I https://jeval.otepc.go.th
```