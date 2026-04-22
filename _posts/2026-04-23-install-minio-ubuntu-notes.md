---
title: Cài MinIO trên Ubuntu — vài note sau khi tự vật lộn
date: 2026-04-23 03:00:00 +0700
categories: [DevOps, Storage, Ubuntu]
tags: [minio, s3, ubuntu, nginx, cloudflare, storage, self-hosted]
---

Mình vừa setup MinIO trên một con server Ubuntu để làm storage (kiểu S3 clone), nên tiện viết lại vài thứ cho ai cần. Bài này không phải lý thuyết — toàn là những gì mình đã làm + lỗi mình đã dính.

---

## Tổng quan setup của mình

```bash
Browser ←→ Cloudflare ←→ Nginx ←→ MinIO
                             ├── 9000 (API)
                             └── 9001 (Console)
```

* `9000`: app dùng để upload/download
* `9001`: giao diện web quản lý

Mình dùng:

* Cloudflare để xử lý SSL ngoài
* Nginx làm reverse proxy vào MinIO

👉 Quan trọng: cần **2 domain riêng**

* `s3.domain.com` → cho API
* `console.domain.com` → để login quản lý

---

## Cài đặt nhanh bằng script

Mình không cài tay, mà nhờ AI viết luôn script cho đỡ mệt:

```bash
curl -O https://gist.githubusercontent.com/jhin1m/1f0b11f7d14a1ffe14b8a04b4e665307/raw/508b246592fbe9832feed4a8507e2393475a3a5c/install-minio.sh
sudo bash install-minio.sh
```

Script này basically làm hết:

* tải MinIO
* tạo user riêng (`minio-user`)
* setup systemd
* config firewall
* dựng Nginx
* cài SSL (Cloudflare hoặc Let's Encrypt)
* cài luôn `mc` CLI

Chạy xong là gần như có server dùng luôn.

---

## Một lỗi ngu nhưng rất dễ dính 😅

### Chọn sai ổ đĩa lưu data

Lúc script hỏi:

> "Data directory là gì?"

👉 Nhớ check trước:

```bash
df -h
```

Ví dụ:

* `/` = 2TB
* `/home` = 13TB

→ thì phải chọn `/home/minio-data`

Không là sau này đầy ổ root là toang 😭

---

## Sau khi cài xong cần làm gì?

### 1. Trỏ domain

* tạo A record trỏ về IP server
* nếu dùng Cloudflare → bật proxy (màu cam)

SSL mode:

* `Full` hoặc `Full (Strict)`

---

### 2. Check server sống chưa

```bash
systemctl status minio
mc admin info local
```

---

## Mấy lệnh `mc` mình dùng nhiều

### Tạo bucket

```bash
mc mb local/my-bucket
```

### Upload file

```bash
mc cp file.jpg local/my-bucket/
```

### Download

```bash
mc cp local/my-bucket/file.jpg ./
```

### Xoá sạch bucket

```bash
mc rm --recursive --force local/my-bucket
```

---

## Kết nối từ app (cực kỳ quan trọng)

### Laravel

```env
AWS_ENDPOINT=https://s3.domain.com
AWS_USE_PATH_STYLE_ENDPOINT=true
```

### Node.js

```ts
forcePathStyle: true
```

👉 Nếu quên cái này → lỗi DNS ngay.

Vì MinIO dùng dạng:

```bash
domain.com/bucket/file
```

chứ không phải:

```bash
bucket.domain.com/file
```

---

## Trick bảo mật hay dùng

Mặc định nếu bạn chạy:

```bash
mc anonymous set download
```

→ ai cũng:

* xem file
* **và xem luôn danh sách file**

👀 không ổn lắm

---

### Cách mình fix

Chỉ cho download file trực tiếp, không cho list:

```bash
cat > /tmp/policy.json << 'EOF'
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {"AWS": ["*"]},
      "Action": ["s3:GetObject"],
      "Resource": ["arn:aws:s3:::my-bucket/*"]
    }
  ]
}
EOF

mc anonymous set-json /tmp/policy.json local/my-bucket
```

→ dùng link vẫn tải được
→ nhưng không ai browse được bucket

---

## Những lỗi mình đã gặp (và cách fix)

### ❌ Login Console bị 401

Do mình từng set:

```bash
MINIO_SERVER_URL
MINIO_BROWSER_REDIRECT_URL
```

👉 Xoá hết là xong.

---

### ❌ SDK báo lỗi kiểu: bucket.domain.com

→ do nó tự chuyển sang subdomain

Fix:

```bash
forcePathStyle: true
```

---

### ❌ Cloudflare chặn request (403 / Just a moment...)

Cái này khá khó chịu.

Fix nhanh:

* tắt Bot Fight Mode
* tạo rule skip cho domain S3
* nếu vẫn lỗi → chuyển DNS only (tắt proxy)

---

### ❌ Mở port mà vẫn không vào được

Do UFW:

```bash
ufw status numbered
```

→ rule `deny` nằm trên `allow`

Fix: xoá rule sai đi

---

### ❌ Không connect được từ máy khác

MinIO đang bind localhost

Fix:

```bash
--address :9000
```

---

## Tổng kết kiểu "rút kinh nghiệm xương máu"

* Đừng set `MINIO_SERVER_URL` nếu dùng proxy
* Luôn bật `forcePathStyle`
* Đừng dùng `set download` nếu không muốn lộ file list
* Dùng `mc` CLI cho nhanh, Console nhiều lúc hơi tù do bản community bị cắt bớt tính năng

---

Nếu bạn đang build:

* hệ thống upload ảnh
* CDN nội bộ
* hoặc thay S3 để tiết kiệm chi phí

→ MinIO khá ổn, miễn là config đúng ngay từ đầu.
