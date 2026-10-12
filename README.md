# Bài 2: Tự Động Hóa Đóng Gói Và Phát Hành Docker Image Khi Gắn Git Tag (v*.*.*)

## 🎯 1. Mục Tiêu
- Xây dựng quy trình Continuous Delivery (CD) tự động kích hoạt đóng gói và phát hành Docker Image khi lập trình viên gán nhãn phiên bản (**Git Tag** chuẩn Semantic Versioning: `v*.*.*`).
- Sử dụng action `docker/metadata-action@v5` để tự động trích xuất số phiên bản từ Git Tag và gắn thẻ Docker Tag tương ứng (`v1.0.0` và `latest`).
- Xuất bản an toàn Docker Image lên **GitHub Container Registry (GHCR - `ghcr.io`)** thông qua `docker/build-push-action@v5`.

---

## 📁 2. Cấu Trúc Thư Mục
```text
bai2/
├── .github/
│   └── workflows/
│       └── release.yml          # Kịch bản phát hành tự động khi push Git Tag
├── .dockerignore
├── Dockerfile                   # Khai báo đóng gói ứng dụng
├── index.html                   # Giao diện web ứng dụng
└── README.md                    # Tài liệu hướng dẫn & quy trình kiểm tra
```

---

## ⚙️ 3. Kịch Bản Workflow Phát Hành (`.github/workflows/release.yml`)

```yaml
name: Release Docker Image to GHCR

on:
  push:
    tags:
      - 'v*.*.*'

jobs:
  release-image:
    name: Build & Release Docker Image
    runs-on: ubuntu-latest

    permissions:
      contents: read
      packages: write

    steps:
      - name: Checkout mã nguồn
        uses: actions/checkout@v5

      - name: Đăng nhập vào GitHub Container Registry
        uses: docker/login-action@v3
        with:
          registry: ghcr.io
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - name: Trích xuất metadata Docker Tag từ Git
        id: meta
        uses: docker/metadata-action@v5
        with:
          images: ghcr.io/${{ github.repository }}
          tags: |
            type=semver,pattern={{version}}
            type=raw,value=latest

      - name: Đóng gói và phát hành Docker Image lên GHCR
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          labels: ${{ steps.meta.outputs.labels }}
```

### 💡 Phân Tích Cơ Chế CD:
1. **Trigger `push.tags: ['v*.*.*']`**:
   - Workflow chỉ được kích hoạt khi có Git Tag dạng `v1.0.0`, `v1.0.1`, `v2.0.0` được đẩy lên repository. Các commit thông thường vào nhánh `main` hoặc `develop` sẽ không kích hoạt luồng phát hành này, giúp bảo vệ môi trường sản phẩm.
2. **Quyền `packages: write`**:
   - Cho phép runner sử dụng token nội bộ `${{ secrets.GITHUB_TOKEN }}` để đẩy trực tiếp package lên GitHub Container Registry mà không cần tạo Personal Access Token thủ công.
3. **Action `docker/metadata-action@v5`**:
   - Đọc tên tag Git (ví dụ `v1.0.0`) và tự động sinh ra các tag Docker chuẩn OCI:
     - `ghcr.io/<owner>/<repo>:v1.0.0`
     - `ghcr.io/<owner>/<repo>:latest`
4. **Action `docker/build-push-action@v5`**:
   - Tự động gọi Docker Buildx, đóng gói image và đẩy thẳng lên `ghcr.io`.

---

## 🚀 4. Quy Trình Kiểm Thử & Kích Hoạt Release

### Bước 1: Commit mã nguồn và đẩy lên nhánh main
```bash
git add .
git commit -m "feat: complete session 12 exercise 2 with release.yml workflow and Dockerfile"
git push origin main
```

### Bước 2: Tạo Git Tag và đẩy Tag lên GitHub
```bash
git tag v1.0.0
git push origin v1.0.0
```

---

## 🔍 5. Kiểm Tra Kết Quả Trên GitHub

1. **Tab Actions**:
   - Mở mục **Actions** của repository, quan sát workflow **"Release Docker Image to GHCR"** được tự động kích hoạt bởi tag `v1.0.0`.
   - Các bước Đăng nhập GHCR, Extract Metadata, Build and Push đều hiển thị tích xanh (Success).
2. **Mục GitHub Packages**:
   - Truy cập trang chủ của repository hoặc trang cá nhân GitHub -> Chọn tab **Packages**.
   - Xác nhận sự xuất hiện của Container Image `ghcr.io/...:v1.0.0`.
   - Có thể tải thử image về máy cá nhân bằng lệnh:
     ```bash
     docker pull ghcr.io/<username>/re-deloys-session12-lesson2:v1.0.0
     ```
