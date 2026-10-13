# Ss14-Ex01: Nâng cấp luồng Đăng nhập (Access Token & Refresh Token)

Dự án Identity Service triển khai xác thực người dùng bằng cơ chế kép: Access Token (JWT - ngắn hạn) và Refresh Token (UUID - dài hạn trong PostgreSQL/Database).

## 1. Yêu cầu & Cấu hình

### JJWT & Dependencies
- JJWT phiên bản: `0.11.5`
- Spring Boot: `3.4.3`
- Java: `17`

### Cấu hình `application.yml`
```yaml
server:
  port: 8080

spring:
  application:
    name: identity-service
  datasource:
    url: jdbc:postgresql://${DB_HOST:localhost}:${DB_PORT:5432}/${DB_NAME:identity_db}
    username: ${DB_USER:postgres}
    password: ${DB_PASSWORD:postgres}
    driver-class-name: org.postgresql.Driver
  jpa:
    hibernate:
      ddl-auto: update
    show-sql: true

app:
  jwt:
    secret: "vS6K5vH8N2zB4xR9mQ3pL1wT7yC5vH8N2zB4xR9mQ3pL1wT7yC5vH8N2z"
    access-token-expiration: 900000 # 15 mins
    refresh-token-expiration: 604800000 # 7 days
```

---

## 2. Các thành phần chính

1. **Entity `RefreshToken` (`com.rikkei.microservice.identity.entity.RefreshToken`)**:
   - Trường: `id`, `token` (String, UUID), `expiryDate` (Instant), và quan hệ `@OneToOne` tới `User`.
   - Bảng cơ sở dữ liệu: `refresh_tokens`.

2. **Entity `User` (`com.rikkei.microservice.identity.entity.User`)**:
   - Quản lý thông tin đăng nhập, danh sách `roles`, triển khai `UserDetails`.

3. **`JwtUtils` (`com.rikkei.microservice.identity.security.JwtUtils`)**:
   - Hàm `generateAccessToken(User user)` tạo JWT token ký bằng HS256 với secret key từ cấu hình, mang các claims: `roles`, `email`, `userId`, `sub`, thời hạn 15 phút.
   - Hàm `validateJwtToken(String token)` và `getUsernameFromJwtToken(String token)`.

4. **`RefreshTokenService` (`com.rikkei.microservice.identity.service.RefreshTokenService`)**:
   - `createRefreshToken(userId)`: Tạo UUID, tính thời gian hết hạn (`now + 7 days`), lưu vào DB và trả về `RefreshToken`.
   - `verifyExpiration(token)`: Kiểm tra hạn dùng của token, tự động xóa nếu đã hết hạn.

5. **`AuthController` (`com.rikkei.microservice.identity.controller.AuthController`)**:
   - `POST /api/auth/register`: Đăng ký tài khoản người dùng mới.
   - `POST /api/auth/login`: Xác thực thông tin đăng nhập, cấp phát đồng thời `accessToken` và `refreshToken` trả về qua `JwtResponse`.
   - `POST /api/auth/refreshtoken`: Cấp mới Access Token từ Refresh Token còn hạn.

---

## 3. Hướng dẫn kiểm thử (Postman & Database)

### Bước 1: Đăng ký tài khoản
- **Method**: `POST`
- **URL**: `http://localhost:8080/api/auth/register`
- **Body (JSON)**:
  ```json
  {
    "username": "john_doe",
    "email": "john@example.com",
    "password": "password123",
    "roles": ["USER"]
  }
  ```

### Bước 2: Đăng nhập (Login)
- **Method**: `POST`
- **URL**: `http://localhost:8080/api/auth/login`
- **Body (JSON)**:
  ```json
  {
    "username": "john_doe",
    "password": "password123"
  }
  ```
- **Response mẫu**:
  ```json
  {
    "accessToken": "eyJhbGciOiJIUzI1NiJ9...",
    "tokenType": "Bearer",
    "refreshToken": "e3851532-6a45-42a1-a4be-a6bcf3a1e9c2",
    "id": 1,
    "username": "john_doe",
    "email": "john@example.com",
    "roles": ["USER"]
  }
  ```

### Bước 3: Kiểm tra Database Console
Mở Database Console (PostgreSQL):
```sql
SELECT * FROM refresh_tokens;
```
Xác nhận bản ghi chứa đúng `token` (UUID), `expiry_date` (sau 7 ngày) và `user_id` liên kết với người dùng.

### Bước 4: Kiểm tra Access Token trên jwt.io
- Copy chuỗi `accessToken` dán vào [jwt.io](https://jwt.io/).
- Kiểm tra các Claims trong Payload:
  - `sub`: `john_doe`
  - `roles`: `["USER"]`
  - `exp`: timestamp sau 15 phút.

### Bước 5: Kiểm tra gia hạn với Refresh Token
- **Method**: `POST`
- **URL**: `http://localhost:8080/api/auth/refreshtoken`
- **Body (JSON)**:
  ```json
  {
    "refreshToken": "e3851532-6a45-42a1-a4be-a6bcf3a1e9c2"
  }
  ```
