# Keycloak Docker Setup

## Mục tiêu
Standalone Keycloak OIDC provider, chạy độc lập bằng Docker Compose.
Phục vụ nhiều services (Outline, và các services khác trong tương lai).

## Versions
| Service    | Image                 | Tag   |
|------------|-----------------------|-------|
| Keycloak   | `keycloak/keycloak`   | 26.5  |
| PostgreSQL | `postgres`            | 16    |

## URLs
- Keycloak Admin Console: http://localhost:8080/admin

## Kiến trúc
- 2 services: `postgres` + `keycloak`
- Keycloak expose port `127.0.0.1:8080:8080` để truy cập admin console
- PostgreSQL không expose ra ngoài host
- Keycloak chạy dev mode (`start-dev`), HTTP, không cần TLS
- Kết nối với services khác qua external Docker network `services-net`

## Cấu trúc files
```
keycloak-docker/
├── docker-compose.yml
├── .env                 # secrets thật (không commit)
├── .env.example         # template
└── postgres/
    └── init.sh          # tạo DB keycloak tự động lần đầu
```

## Thứ tự setup
1. Đảm bảo external network tồn tại: `docker network create services-net`
2. Tạo `.env` từ `.env.example`, điền passwords
3. `docker compose up -d`
4. Đợi Keycloak sẵn sàng (~60s), kiểm tra log: `docker compose logs keycloak --follow`
5. Vào http://localhost:8080/admin để cấu hình realm và OIDC clients

## Keycloak OIDC Client Config (cho Outline)
- Realm: `outline`
- Client ID: `outline`
- Client authentication: ON (confidential)
- Standard Flow: enabled
- Valid Redirect URIs: `http://localhost:3000/auth/oidc.callback`
- Valid Post Logout Redirect URIs: `http://localhost:3000`
- Web Origins: `http://localhost:3000`

## Các biến env quan trọng
- `KC_HOSTNAME=localhost`: pin issuer trong token về `localhost`, tránh mismatch giữa browser (`localhost:8080`) và Docker internal (`keycloak:8080`)
- `KC_HOSTNAME_PORT=8080`: kết hợp với KC_HOSTNAME để URL trong token luôn đúng
- `KC_HOSTNAME_STRICT=false`: cho phép Keycloak nhận request nội bộ qua `keycloak:8080` (server-to-server từ Outline)
- `KC_HTTP_ENABLED=true`: bắt buộc khi chạy dev mode
- `KEYCLOAK_DB_PASSWORD`: password của DB user `keycloak` trong PostgreSQL

## Lưu ý kỹ thuật

### `services-net` phải tồn tại trước khi start
```bash
docker network create services-net
```
Nếu network chưa tồn tại, stack sẽ fail với lỗi `network not found`.

### `postgres/init.sh` chỉ chạy một lần
Script chỉ chạy khi volume `postgres_data` được tạo lần đầu.
Để reset: `docker compose down -v` rồi start lại.

### `init.sh` cần executable bit
Git không tự preserve `chmod +x`. Sau khi clone fresh, phải chạy:
```bash
chmod +x postgres/init.sh
```

## Rules cho Claude
- Không tự ý implement code khi chưa được yêu cầu rõ ràng
- Nếu phát hiện unknowns, risks hoặc các điểm cần xác nhận → hỏi user trước
- Không tự ý bắt đầu tasks khác ngoài task được yêu cầu
