---
menuTitle: "CI/CD"
date: 2026-01-09T16:03:43+07:00
weight: 90
chapter: false
pre: " <b> 5.6. </b> "
---

# CI/CD — Triển khai Tự động hóa

Trong một dự án AI Production, việc tự động hóa (CI/CD) không chỉ giúp tiết kiệm thời gian mà còn đảm bảo hạ tầng Cloud luôn đồng bộ với mã nguồn. Chúng ta sẽ sử dụng **AWS Amplify Console** kết nối trực tiếp với Repo GitHub.

## 1. Kết nối Nhánh (Branch Strategy)
Kết nối các nhánh GitHub tương ứng với các môi trường AWS:
- **`main`**: Triển khai môi trường **Production**.
- **`staging`**: (Tùy chọn) Triển khai cho đội ngũ QA.
- **`sandbox`**: Dành cho phát triển cá nhân (thường sử dụng lệnh `npx ampx sandbox`).

## 2. Cấu hình Build (`amplify.yml`)

Dưới đây là tệp cấu hình `amplify.yml` nâng cao, được tối ưu hóa để build cả Backend (CDK) và Frontend (Expo) trong cùng một lượt. 

> [!IMPORTANT]
> Lưu ý phần cài đặt Lambda: Chúng ta cần `cd` vào từng thư mục hàm để cài đặt dependencies riêng biệt trước khi Amplify thực hiện đóng gói.

```yaml
version: 1
backend:
  phases:
    build:
     commands:
        - cd backend
        - npm install --legacy-peer-deps --include=dev

        - cd amplify/ai-engine
        - npm install --include=dev
        - cd ../..

        - cd amplify/process-nutrition
        - npm install --include=dev
        - cd ../..

        - cd amplify/friend-request
        - npm install --include=dev
        - cd ../..

        - cd amplify/resize-image
        - rm -rf node_modules package-lock.json
        - npm install --platform=linux --arch=x64 sharp
        - npm install --legacy-peer-deps
        - cd ../..

        - npx ampx pipeline-deploy --branch $AWS_BRANCH --app-id $AWS_APP_ID --outputs-out-dir ../frontend
        - cd ..
frontend:
  phases:
    preBuild:
      commands:
        - cd frontend && npm install --legacy-peer-deps && cd ..
    build:
      commands:
        - cd frontend && npm run build
  artifacts:
    baseDirectory: frontend/dist
    files:
      - "**/*"
  cache:
    paths:
      - frontend/node_modules/**/*
      - frontend/.expo/**/*
  customRules:
    - source: </^[^.]+$|\.(?!(css|gif|ico|jpg|js|png|txt|svg|woff|woff2|ttf|map|json|webp)$)([^.]+$)/>
      target: /index.html
      status: '200'
  customHeaders:
    - pattern: "**/*.html"
      headers:
        - key: Cache-Control
          value: no-cache
    - pattern: "**/*.js"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
    - pattern: "**/*.css"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
    - pattern: "**/*.(png|jpg|jpeg|svg|webp)"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
    - pattern: "**/*.(woff|woff2|ttf)"
      headers:
        - key: Cache-Control
          value: public, max-age=31536000, immutable
```

## 3. Tối ưu hóa & Lưu ý kỹ thuật

### Giải quyết xung đột Dependency
Vì dự án sử dụng **Expo 54** và **React 19**, bạn có thể gặp lỗi `peer dependency conflict`. Hãy luôn sử dụng cờ `--legacy-peer-deps` trong các lệnh cài đặt để đảm bảo quá trình build thành công.

---

## 3. Chỉnh sửa cấu hình Build trên Console

Sau khi kết nối dự án với GitHub, bạn cần cập nhật nội dung `amplify.yml` đã chuẩn bị ở trên vào AWS Console:

1. Truy cập [AWS Amplify Console](https://console.aws.amazon.com/amplify).
2. Chọn ứng dụng của bạn.
3. Ở menu bên trái, chọn **App settings** -> **Build settings**.
4. Nhấn nút **Edit** ở góc phải phần **Build specification**.
5. Dán nội dung YAML ở mục 2 vào và nhấn **Save**.

## 4. Đồng bộ môi trường Local (`amplify_outputs.json`)

Tệp `amplify_outputs.json` là "cầu nối" giúp ứng dụng Frontend biết được các Endpoint của Backend (API, Auth, Storage). Trong quá trình CI/CD, Amplify tự động tạo file này, nhưng để lập trình ở máy cá nhân (Local), bạn cần tải nó về theo cách thủ công.

### Câu lệnh đồng bộ:
Mở terminal tại thư mục dự án và chạy:

```bash
npx ampx generate outputs --branch <tên-nhánh> --app-id <app-id-của-bạn>
```

> [!TIP]
> Bạn có thể tìm thấy **App ID** ngay tại trang Overview của dự án trên Amplify Console.

> [!CAUTION]
> Tuyệt đối không đẩy tệp `amplify_outputs.json` lên GitHub để tránh lộ thông tin endpoint và API Keys.

---

[Tiếp tục đến 5.7 Dọn dẹp](../5.7-Cleanup/)
