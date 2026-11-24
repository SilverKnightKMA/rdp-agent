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

## 6. Smoke mở rộng 2026-11-24 (task #348, cùng VM 192.168.10.52, Windows 10)

Mỗi kết luận dưới đây có bằng chứng pixel-diff trước/sau + M3 mô tả ảnh, không đoán.

| Lệnh | Kết quả | Bằng chứng |
|---|---|---|
| `key super+r` → `type` → `key Return` | OK | Run mở (M3: CÓ) → Notepad mở → `Hello from pi over RDP 123` đọc lại khớp |
| `paste` tiếng Việt có dấu (clipboard) | OK, khớp từng byte | Ctrl+A/Ctrl+C trong Notepad → clipboard Windows → cliprdr → X CLIPBOARD trên `:77` đọc lại **200/200 byte, 151/151 ký tự** (`đặng`, `hiếm`, `ặ ằ Ệ Ộ ữ ỷ`, `Đ`) — nhịp clipboard chạy được cả 2 chiều |
| `move X Y` | **KHÔNG đưa con trỏ sang Windows** | Notepad đang active → `move` ra desktop → `scroll down 8` vẫn cuộn Notepad (diff 1.67%); hover nút Start không đổi; XTEST MotionNotify tuyệt đối cũng vậy. Chỉ chuyển động **khi đang giữ nút** (`drag`) mới tới Windows |
| `click` | OK, toạ độ chính xác | click (150,187) → thanh trạng thái `Ln=38, Col=8` đúng dòng đang thấy |
| `scroll up/down N` | OK cả 2 chiều, **phải click trước** | down 4 → diff 0.48%; up 4 → về đúng frame cũ (0 pixel) |
| `drag X1 Y1 X2 Y2` | OK | `super+Left` snap nửa trái [0,0,959,1039] → kéo (300,15)→(980,200) → cửa sổ [681,187,1917,794] |
| `key alt+Tab` | OK (chuyển focus) | alt+Tab rồi alt+F4 đóng Calculator; chữ gõ xen kẽ vào đúng cửa sổ |
| `key ctrl+plus` / `ctrl+0` | OK | Notepad 160% rồi về 100% (thanh trạng thái) |
| `key super+l` → `state` → `unlock` | khóa OK; **heuristic `state` sai** | Win+L → màn đăng nhập `admin` + ô mật khẩu (M3), nhưng `state` = `desktop (dark bottom 1.0)` → `unlock` thường "doing nothing". `unlock --force` → reconnect NLA → `Session unlocked and still alive.`, exit 0, desktop trở lại (M3) |
| `reconnect` | OK | xfreerdp pid mới (2088), state UP, 0 dòng ERRCONNECT/LOGON trong log |
| `lang` / `langtoggle` | không chạy | máy 1 layout, không có chỉ báo (M3 nhìn s0) |
| AUTH_MARKERS vs FreeRDP 3.15 | 8/8 chuỗi có trong binary | quét byte `xfreerdp3`, `libfreerdp3.so.3.15.0`, `libfreerdp-client3.so.3.15.0`, `libwinpr3.so.3.15.0`: `ERRCONNECT_LOGON_FAILURE`, `ERRCONNECT_ACCOUNT_LOCKED_OUT`, `ERRCONNECT_ACCOUNT_DISABLED`, `ERRCONNECT_PASSWORD_EXPIRED`, `ERRCONNECT_PASSWORD_MUST_CHANGE`, `ERRCONNECT_AUTHENTICATION_FAILED`, `SEC_E_LOGON_DENIED`, `STATUS_LOGON_FAILURE`… (chỉ kiểm tĩnh, không tạo lỗi thật) |
| Tài nguyên | nhẹ | idle: CPU 1.17%, RSS 26 MB (Xvfb 12.7, xfreerdp 6.3, x11vnc 2.3); sau smoke: RSS 133 MB, NET I/O 18.8 MB. `docker stats` báo MEM 0B (cgroup không đếm) → đo bằng `/proc/*/status` |

Drift mới tìm thấy:

- **Thiếu Pillow trong image**: `state`/`unlock` trả `unknown (dark bottom None)` vì `Dockerfile.smoke` không cài `python3-pil` → đã thêm vào Dockerfile (commit này); container đang chạy thì `apt-get install python3-pil` tay.
- **Heuristic `state` không dùng được trên máy này**: màn đăng nhập Win10 có đáy tối gần hết (dark bottom 1.000, desktop 0.997) → luôn "desktop" → cần `unlock --force` hoặc để model mắt quyết định "có phải màn đăng nhập".
- **`move` không tới Windows** (xem bảng): muốn cuộn/hover thì định vị con trỏ bằng `click` trước; tooltip/menu hover không làm được bằng `move`.
- **M3 đọc chữ Việt cỡ nhỏ không ổn định với dấu chồng** (`ậ ế ệ ộ ữ`): cùng một từ `đặng` cho 4 kết quả khác nhau (đáng / đẳng / dấu / tầng) ở ảnh gốc lẫn khi zoom `-resize`. Kiểm text đã gõ thì đọc lại bằng clipboard (Ctrl+A/Ctrl+C → đọc X CLIPBOARD), đừng tin OCR. Crop đưa M3 đọc phải ~dưới 1300px chiều dài (crop 2700x102 bị đọc tệ hơn ảnh gốc).
- M3 trả lời "cửa sổ nào đang active" bằng cảm tính (to hơn = active), sai; focus phải kiểm bằng hiệu quả hành động (alt+F4 đóng đúng cửa sổ, chữ gõ vào đúng ô).
- **Bài học suýt nguy hiểm**: Notepad mở lại ở vị trí cũ (nửa phải [681,186,1916,793]), click (600,400) rơi ra desktop → mất focus → paste/scroll vô hiệu, **Alt+F4 trên desktop = hộp thoại "Shut Down Windows"** (Enter là tắt máy). Đã huy bằng Escape, không có gì xẩy ra. Luật cho loop: trước `alt+F4` phải chắc cửa sổ đích có focus (click TRONG toạ độ cửa sổ vừa chụp, hoặc alt+Tab rồi kiểm bằng diff); khoáфе có focus KHÔNG Alt+F4.

Dọn dẹp: Notepad đóng không lưu, Calculator đóng, desktop trống như ban đầu; jiggler **tắt** (`keepalive off`); phiên `rdp-session` vẫn UP chờ user quyết.

