# Keycloak — Docker Compose Setup

Standalone [Keycloak](https://www.keycloak.org) OIDC provider, chạy độc lập bằng Docker Compose.
Phục vụ nhiều services qua shared external Docker network `services-net`.

---

## Versions

| Service    | Image               | Tag  |
|------------|---------------------|------|
| Keycloak   | `keycloak/keycloak` | 26.5 |
| PostgreSQL | `postgres`          | 16   |

---

## Kiến trúc

```
localhost:8080 (admin)
      │
  keycloak:8080  ◄──── services-net ────► [các services khác]
  postgres:5432
```

- **2 services:** `postgres` + `keycloak`
- **Keycloak** expose port `127.0.0.1:8080:8080` để truy cập admin console
- **PostgreSQL** không expose ra ngoài host
- **Keycloak dev mode** (`start-dev`) — HTTP, không cần TLS
- Các services khác kết nối qua external Docker network `services-net`

---

## Yêu cầu

- [Docker Desktop](https://www.docker.com/products/docker-desktop/) (hoặc Docker Engine + Docker Compose v2)

---

## Cài đặt

### Bước 1 — Chuẩn bị `services-net`

Network dùng chung với các services khác. Tạo một lần duy nhất:

```bash
docker network create services-net
```

Nếu đã tồn tại thì bỏ qua.

---

### Bước 2 — Chuẩn bị `.env`

```bash
cp .env.example .env
```

Mở `.env` và điền passwords:

```bash
POSTGRES_ROOT_PASSWORD=<strong_password>
KEYCLOAK_DB_PASSWORD=<strong_password>
KC_ADMIN_USERNAME=admin
KC_ADMIN_PASSWORD=<strong_password>
```

---

### Bước 3 — Khởi động

```bash
docker compose up -d
```

Đợi Keycloak sẵn sàng (~60 giây):

```bash
docker compose logs keycloak --follow
```

Keycloak sẵn sàng khi log hiển thị:
```
Running the server in development mode.
Listening on: http://0.0.0.0:8080
```

---

### Bước 4 — Cấu hình Keycloak

Vào **http://localhost:8080/admin** — đăng nhập với `KC_ADMIN_USERNAME` / `KC_ADMIN_PASSWORD`.

#### 4.1 Tạo Realm

1. Click dropdown **"Keycloak"** → **"Create realm"**
2. **Realm name:** `outline` → **Create**

#### 4.2 Tạo OIDC Client

1. Menu trái → **Clients** → **"Create client"**
2. **Client type:** `OpenID Connect` | **Client ID:** `outline` → **Next**
3. **Client authentication:** ON | **Standard flow:** ticked → **Next**
4. **Valid redirect URIs:** `http://localhost:3000/auth/oidc.callback`
5. **Valid post logout redirect URIs:** `http://localhost:3000`
6. **Web origins:** `http://localhost:3000` → **Save**
7. Tab **"Credentials"** → copy **Client secret**

> Client secret này cần điền vào `.env` của `outline-docker`.

#### 4.3 Tạo User

1. Menu trái → **Users** → **"Create new user"**
2. Điền **Username**, **Email** (bắt buộc), bật **Email verified** → **Create**
3. Tab **"Credentials"** → **"Set password"** → tắt **Temporary** → **Save password**

---

## Cấu trúc files

```
.
├── docker-compose.yml      # 2 services: postgres, keycloak
├── .env                    # Secrets thật (không commit)
├── .env.example            # Template
├── postgres/
│   └── init.sh             # Tạo database keycloak tự động
├── CLAUDE.md               # Tài liệu kỹ thuật nội bộ
└── README.md               # File này
```

---

## Quản lý

### Khởi động lại

```bash
docker compose down && docker compose up -d
```

### Xem logs

```bash
docker compose logs keycloak --follow
docker compose logs postgres --follow
```

### Reset hoàn toàn (xóa toàn bộ dữ liệu)

```bash
docker compose down -v
```

> ⚠️ Lệnh này xóa toàn bộ volumes — mất dữ liệu Keycloak (realm, clients, users).
> Phải cấu hình lại từ Bước 4.

---

## Lưu ý kỹ thuật

### `services-net` phải tồn tại trước khi start

Nếu network chưa tồn tại, stack sẽ fail với lỗi:
```
network "services-net" declared as external, but could not be found
```

### `init.sh` chỉ chạy một lần

`postgres/init.sh` chỉ chạy khi volume `postgres_data` được tạo lần đầu.
Để reset database: `docker compose down -v`

### `init.sh` cần executable bit sau khi clone

Git không tự preserve `chmod +x`:

```bash
chmod +x postgres/init.sh
```

---

## Troubleshooting

### "HTTPS required" khi truy cập admin console

Trình duyệt có thể đã cache HSTS redirect. Thử mở tab **incognito** và truy cập
`http://localhost:8080/admin` (đảm bảo dùng `http://`, không phải `https://`).

### Keycloak không start — lỗi database connection

Kiểm tra DB `keycloak` đã được tạo:

```bash
docker compose exec postgres psql -U postgres -c "\l"
```

Nếu không thấy DB `keycloak`, volume có thể đã tồn tại từ trước và `init.sh` bị bỏ qua.
Reset: `docker compose down -v && docker compose up -d`
