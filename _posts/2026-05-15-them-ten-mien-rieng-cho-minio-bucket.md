---
title: Thêm tên miền riêng cho từng MinIO bucket (path-style + nginx rewrite)
date: 2026-05-15
categories: [DevOps, Storage, Ubuntu]
tags: [minio, s3, nginx, cloudflare, domain, reverse-proxy, ssl]
---

Sau bài [cài MinIO trên Ubuntu](/posts/install-minio-ubuntu-notes/), mình có 1 con server đang chạy nhiều bucket dưới chung 1 domain `s3.domain.com` (path-style: `s3.domain.com/<bucket>/<key>`).

Hôm nay có nhu cầu **map 1 bucket cụ thể ra 1 domain riêng**, ví dụ:

```
https://cdn.example.com/images/sample/1.jpg
              ↑                 ↑
        domain mới        path bên trong bucket
```

Bucket tên `my-bucket`, nhưng URL **không có** `/my-bucket/` ở đầu — client cứ tưởng đang gọi thẳng `domain/key`.

Cách làm: dùng **nginx rewrite** để tự chèn `/my-bucket/` vào URL trước khi proxy về MinIO. Không cần đụng vào config MinIO, không cần restart MinIO, không ảnh hưởng các bucket khác.

---

## Tổng quan flow

```bash
Browser → Cloudflare → Nginx (443, cdn.example.com)
                         ↓ proxy_pass http://127.0.0.1:9000/my-bucket/
                        MinIO (path-style, bucket = my-bucket)
```

Client request `/images/abc.jpg` → nginx forward thành `/my-bucket/images/abc.jpg` → MinIO trả file.

---

## Bước 1: Lấy SSL cert cho domain mới

Nếu domain mới khác zone với cert cũ → cert cũ không cover → phải tạo cert mới.

Trên Cloudflare:

1. Vào zone `example.com` → **SSL/TLS** → **Origin Server** → **Create Certificate**
2. Hostnames: `*.example.com, example.com`
3. Validity: 15 năm (đỡ phải lo gia hạn)
4. Bấm **Create** → copy cert + key

Lưu xuống server:

```bash
# Cert
cat > /etc/ssl/cf-example-com.pem << 'EOF'
-----BEGIN CERTIFICATE-----
... (paste vào đây)
-----END CERTIFICATE-----
EOF
chmod 644 /etc/ssl/cf-example-com.pem

# Key
cat > /etc/ssl/cf-example-com.key << 'EOF'
-----BEGIN PRIVATE KEY-----
... (paste vào đây)
-----END PRIVATE KEY-----
EOF
chmod 600 /etc/ssl/cf-example-com.key
```

Verify cert và key match nhau (cùng public key):

```bash
diff <(openssl x509 -in /etc/ssl/cf-example-com.pem -noout -pubkey) \
     <(openssl pkey -in /etc/ssl/cf-example-com.key -pubout)
```

Không output gì = match ✓

---

## Bước 2: Nginx site với rewrite path-style

Tạo file `/etc/nginx/sites-available/minio-my-bucket`:

```nginx
server {
    listen 80;
    listen 443 ssl;
    server_name cdn.example.com;

    ssl_certificate     /etc/ssl/cf-example-com.pem;
    ssl_certificate_key /etc/ssl/cf-example-com.key;

    ignore_invalid_headers off;
    client_max_body_size 0;
    proxy_buffering off;
    proxy_request_buffering off;

    # Trick chính nằm ở đây:
    # location khớp /, proxy_pass có URI /my-bucket/
    # → nginx auto thay phần khớp ("/") bằng "/my-bucket/"
    location / {
        proxy_pass http://127.0.0.1:9000/my-bucket/;
        proxy_set_header Host              $http_host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header Authorization     $http_authorization;

        chunked_transfer_encoding off;
        proxy_connect_timeout 300;
        proxy_http_version 1.1;
    }
}
```

Enable:

```bash
ln -sf /etc/nginx/sites-available/minio-my-bucket \
       /etc/nginx/sites-enabled/minio-my-bucket
nginx -t && systemctl reload nginx
```

---

## Bước 3: Bucket policy public read (nếu muốn ai cũng tải được)

Nếu bucket cần public (CDN ảnh, static asset, etc.), set policy chỉ cho `GetObject`, không cho list:

```bash
cat > /tmp/policy-my-bucket.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": {"AWS": ["*"]},
    "Action": ["s3:GetObject"],
    "Resource": ["arn:aws:s3:::my-bucket/*"]
  }]
}
EOF

mc anonymous set-json /tmp/policy-my-bucket.json local/my-bucket
```

Check policy hiện tại:

```bash
mc anonymous get-json local/my-bucket
```

---

## Bước 4: Test trước khi đổi DNS

**Đừng đổi DNS xong mới test.** Test tại chỗ bằng `curl --resolve` để bypass DNS:

```bash
curl -sI -k --resolve cdn.example.com:443:127.0.0.1 \
     https://cdn.example.com/images/sample/1.jpg
```

Phải thấy:

```
HTTP/1.1 200 OK
Content-Type: image/jpeg
Content-Length: 1028575
```

→ Pipeline OK, giờ mới đổi DNS.

---

## Bước 5: Trỏ DNS

Trên Cloudflare DNS:

* Type: `A`
* Name: `cdn` (subdomain)
* IPv4: IP server
* Proxy: **ON** (orange cloud)
* TTL: Auto

SSL mode (SSL/TLS → Overview): **Full (strict)** — vì đã có Origin Cert.

---

## Mấy lỗi mình đã dính lúc cutover 😅

### ❌ Error 1016 (Origin DNS error)

Cloudflare edge chưa thấy A record mới — vẫn cache resolver cũ trỏ về origin cũ (mình đang migrate từ provider storage khác sang).

Fix: **chờ vài giây**, hoặc purge cache Cloudflare.

---

### ❌ InternalError (XML response)

Lúc cutover, có request được Cloudflare cache stub error trong một khoảnh khắc race condition.

Fix: cũng **tự khỏi** sau khi cache layer ổn. Nếu lâu thì purge cache.

---

### ❌ HTTP 200 nhưng nội dung sai (last-modified cũ)

Cloudflare đang trả **cached response từ origin cũ**. Header `cf-cache-status: HIT` + `age: ...` lớn.

Cách kiểm tra nội dung từ origin mới:

```bash
# Thêm query string ngẫu nhiên để bypass cache
curl -sI "https://cdn.example.com/path/file.jpg?nocache=$(date +%s)"
```

Hoặc purge URL trên Cloudflare: **Caching → Configuration → Purge Custom URL**.

---

### ❌ 404 sau khi rewrite

Lúc đầu mình viết:

```nginx
location / {
    proxy_pass http://127.0.0.1:9000;  # ← KHÔNG có /my-bucket/
    rewrite ^/(.*)$ /my-bucket/$1 break;
}
```

Cách này cũng chạy được, **NHƯNG** `proxy_pass http://127.0.0.1:9000/my-bucket/;` (có URI ở cuối) sạch hơn — nginx tự thay phần khớp bởi URI, không cần `rewrite` riêng.

---

## Tại sao không dùng virtual-host style?

MinIO có support virtual-host style qua env:

```bash
MINIO_DOMAIN=example.com
```

Khi đó MinIO tự hiểu `my-bucket.example.com` → bucket `my-bucket`. Nhưng:

* Phải **restart MinIO** → downtime
* Áp dụng cho **toàn bộ** subdomain dưới `example.com` — có thể đụng các domain khác
* Bucket name phải DNS-safe (không underscore, etc.)

Path-style + nginx rewrite linh hoạt hơn: thêm bucket-domain mới chỉ cần thêm 1 file nginx, reload, xong.

---

## Tổng kết

* 1 bucket = 1 nginx site (`/etc/nginx/sites-available/minio-<bucket>`)
* `proxy_pass http://127.0.0.1:9000/<bucket>/;` — nginx tự rewrite path
* Cloudflare Origin Cert per zone, Full (strict)
* Test bằng `curl --resolve` **trước** khi đổi DNS
* Cutover dính 1016 / InternalError là bình thường, để cache settle

Nếu bạn có nhiều bucket cần map riêng (mỗi project / mỗi khách hàng 1 domain), pattern này scale tốt — chỉ cần copy file nginx, đổi `server_name` và bucket trong `proxy_pass`.
