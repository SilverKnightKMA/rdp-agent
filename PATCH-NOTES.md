# PATCH-NOTES — drift house mình so với upstream EvilFreelancer/rdp-agent @2e8ca75

Ngày smoke: 2026-11-23 (VM thật 192.168.10.52, MiniMax-M3 qua magpie). Không có file này,
mọi kiến thức dưới đây chết cùng session compact.

## 1. Patch bắt buộc: `/from-stdin:force` (FreeRDP 3.x)

- `scripts/rdpctl.py:341` — upstream viết `"/from-stdin"` (ngữ nghĩa FreeRDP 2.x: đọc password từ stdin ngay từ đầu).
- FreeRDP 3.15 (Debian trixie, package `freerdp3-x11`) đổi ngữ nghĩa: không có `:force` thì client **chờ server yêu cầu** rồi prompt `Domain:` + `Password:` riêng lẻ → stdin một dòng của rdpctl bị nuốt nhầm → `ERRCONNECT_CONNECT_CANCELLED` lúc `nla_client_setup_identity`.
- Tái tạo patch trên clone mới: `sed -i 's|"/from-stdin",|"/from-stdin:force",|' scripts/rdpctl.py`
- Chẩn đoán phân biệt: lỗi này KHÔNG phải mạng (port OPEN), KHÔNG phải sai password (không có LOGON_FAILURE / AUTH_MARKERS).

## 2. Binary name: `xfreerdp3` không `xfreerdp`

- Package trixie đặt binary `/usr/bin/xfreerdp3`; rdpctl gọi tên `xfreerdp` (:340).
- Fix trong image: `ln -s /usr/bin/xfreerdp3 /usr/local/bin/xfreerdp` (đã có trong Dockerfile.smoke).

## 3. Kiến trúc smoke (không cần sudo trên container chính)

- pi chạy trong container `code-server-omp-docker` (user coder, không sudo, KHÔNG cài được apt).
- Docker socket = daemon HOST (thấy stack homelab), nhưng `/home/coder` của daemon KHÁC của mình (thấy openchamber-workspaces) → **bind-mount không được, docker cp thì được** (client-side).
- Image `rdp-smoke:stage0` build từ `Dockerfile.smoke` (context gửi từ client — chạy được).
- Container dài hạn `rdp-session` giữ phiên RDP sống qua các turn; lệnh: `docker exec rdp-session python3 /opt/rdp-agent/scripts/rdpctl.py <cmd>`.
- Profile: `docker cp rdp-agent.env rdp-session:/root/.config/rdp-agent/default.env` + `chmod 600` (rdpctl check `0o077`). Meuuhpwh không bao giờ qua chat.
- Ảnh ra vào: `docker exec rdp-session ... shot /tmp/s.png` → `docker cp rdp-session:/tmp/s.png /tmp/`.

## 4. Protocol mắt MiniMax-M3 (của nhà mình — KHÔNG có trong upstream)

- Gateway magpie `/chat/completions`, model `minimax/MiniMax-M3`, ảnh = `image_url` data-URL base64 PNG.
- Grounding prompt: "Locate <element>. Reply ONLY JSON {\"bbox_0_1000\": [x1,y1,x2,y1...]} coordinates normalized 0-1000".
- Quy đổi pixel (màn 1920x1080): `px = round(v/1000 * 1920 hoặc 1080)`, click tâm bbox.
- Verify: frame-diff (kích thước PNG trước/sau) + hỏi M3 mô tả màn sau click.
- Đã kiểm chứng live: bbox Start button [8,972,17,991] → click (24,1060) → Start menu mở → M3 tự xác nhận.

## 5. Ghi chú vận hành (từ SKILL.md upstream — giữ lại khi dùng)

- Sau khi mở menu/app: chờ 1-3s trước khi shot (kẻo bắt intermediate frame).
- `start` chiếm session console Windows (người đang ngồi bị đá) — máy test thì vô hại.
- AUTH_MARKERS fail 1 lần → `start` từ chối tới khi `--force` (chống khóa tài khoản) — đừng vòng bypass.
- `stop` chỉ theo lệnh user (luật upstream).
