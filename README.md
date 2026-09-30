# OpenRGB — RGB Lighting Config (Core64)

Cấu hình RGB lighting cho máy **Core64** dành cho hai hệ: **OpenRGB** (điều khiển RAM / GPU / mainboard / AIO pump) và **Lian Li daemon** (fan hub). Hệ thống này thay thế việc phải mở OpenRGB GUI và load profile `.orp` — đổi màu chỉ bằng cách sửa file text `.rgb` rồi chạy một lệnh.

---

## 🖥️ Các thiết bị RGB đang dùng

### Qua OpenRGB

| # | Thiết bị | Loại | Tên OpenRGB |
|---|----------|------|-------------|
| 0 | **Kingston Fury DDR5** | RAM | `Kingston Fury DDR5 DRAM` |
| 1 | **Sapphire RX 7800 XT Nitro+** | GPU | `Sapphire Radeon RX 7800 XT Nitro+` |
| 2 | **ASUS ROG STRIX Z690-A** | Mainboard | `ASUS ROG STRIX Z690-A GAMING WIFI` |

### Thiết bị khác

| Thiết bị | Điều khiển | Ghi chú |
|----------|------------|---------|
| **Deepcool LT720 AIO** (pump) | Nối **ARGB Header 1** mainboard → qua zone ASUS (zone 1, cần resize `SIZE=22`) | Pump có RGB; **fan FK120 KHÔNG có RGB** |
| **Lian Li Uni Hub SL v1 + 5× fan SL120 v1** | `lianli-daemon` (USB `0cf2:a100` = ENE LianLi-UNI FAN-SL-v1.8, hidraw) | Điều khiển fan hub + LED fan SL120, KHÔNG qua OpenRGB |

---

## 📦 Cấu trúc repo

```
.
├── README.md
├── systemd/
│   ├── openrgb.service                          # OpenRGB boot service
│   ├── lianli-daemon.service                    # override unit: độc lập session (bỏ graphical-session.target)
│   └── lianli-daemon.service.d/
│       └── retry-open-acl.conf                  # Lian Li drop-in → wrapper
├── udev/
│   └── 60-aura-led.rules                        # AURA (0b05:19af) → GROUP=i2c sớm
├── local/
│   ├── bin/
│   │   └── apply-rgb                            # script đổi màu
│   └── lib/
│       ├── openrgb-wrapper.sh                   # đợi I801+hidraw Aura → launch server → apply
│       └── lianli-wrapper.sh                    # đợi hidraw (GROUP=lianli, ~1s) → launch daemon
├── lianli/
│   ├── config.json                              # config Lian Li (machine-specific)
│   └── rgb_presets.json                         # preset màu Lian Li
└── schemes/
    ├── white.rgb (default)   ├── rainbow.rgb
    ├── breath.rgb            ├── red.rgb
    ├── daquang2.rgb          ├── xanhtim.rgb
    └── vangxanh.rgb
```

### Ánh xạ file → vị trí hệ thống

| Repo path | Vị trí cài đặt |
|-----------|----------------|
| `systemd/openrgb.service` | `~/.config/systemd/user/openrgb.service` |
| `systemd/lianli-daemon.service` | `~/.config/systemd/user/lianli-daemon.service` |
| `systemd/lianli-daemon.service.d/retry-open-acl.conf` | `~/.config/systemd/user/lianli-daemon.service.d/retry-open-acl.conf` |
| `local/lib/openrgb-wrapper.sh` | `~/.local/lib/openrgb-wrapper.sh` |
| `udev/60-aura-led.rules` | `/etc/udev/rules.d/60-aura-led.rules` (cần root) |
| `local/lib/lianli-wrapper.sh` | `~/.local/lib/lianli-wrapper.sh` |
| `local/bin/apply-rgb` | `~/.local/bin/apply-rgb` |
| `lianli/config.json` | `~/.config/lianli/config.json` |
| `lianli/rgb_presets.json` | `~/.config/lianli/rgb_presets.json` |
| `schemes/*.rgb` | `~/.config/openrgb/schemes/` |

---

## 🛠️ Cài đặt

> ✅ **Đã sẵn sàng** trên Core64 (máy tham chiếu). Các bước dưới đây dành cho máy mới / cài lại.

### Bước 1 — Chuẩn bị hệ thống: công cụ I2C

OpenRGB truy cập RAM/GPU/mainboard qua bus I2C/SMBus. Cài `i2c-tools` và nạp module:

```bash
sudo pacman -S i2c-tools

# nạp module ngay lần này
sudo modprobe i2c-dev
sudo modprobe i2c-i801

# tự nạp mỗi lần boot
sudo tee /etc/modules-load.d/i2c.conf <<'EOF'
i2c-dev
i2c-i801
EOF
```

> 📚 Tham khảo: [OpenRGB SMBusAccess Documentation](https://github.com/CalcProgrammer1/OpenRGB/blob/master/Documentation/SMBusAccess.md)

---

### Bước 2 — Xử lý module `spd5118` (RAM DDR5)

Với RAM DDR5, module kernel `spd5118` chiếm quyền truy cập SPD/SMBus → OpenRGB không nhận diện được RAM. **Chọn 1 trong 2 cách:**

#### Cách 1 (khuyến nghị): Blacklist `spd5118`

```bash
echo "blacklist spd5118" | sudo tee -a /etc/modprobe.d/blacklist-spd5118.conf

# cập nhật initramfs rồi reboot
sudo mkinitcpio -P
```

> ✅ **Máy tham chiếu Core64 đang dùng cách này** — `/etc/modprobe.d/blacklist-spd5118.conf` đã có, `spd5118` không load, OpenRGB detect RAM bình thường.

#### Cách 2 (thay thế Cách 1): gỡ module bằng systemd service

Nếu không muốn blacklist, dùng service `rmmod` lúc boot:

```ini
# /etc/systemd/system/rmmod-spd5118.service
[Unit]
Description=Unload spd5118 module on startup
After=network.target

[Service]
Type=oneshot
ExecStart=/sbin/rmmod spd5118
RemainAfterExit=yes

[Install]
WantedBy=multi-user.target
```

```bash
sudo nano /etc/systemd/system/rmmod-spd5118.service   # dán nội dung trên
sudo chmod 644 /etc/systemd/system/rmmod-spd5118.service
sudo systemctl daemon-reload
sudo systemctl enable --now rmmod-spd5118.service
```

> ⚠️ Chỉ dùng **1 trong 2 cách**. Cách 2 hoạt động nhưng `rmmod` chỉ có hiệu lực tới khi module được nạp lại; blacklist (Cách 1) chặn từ gốc nên bền hơn.

---

### Bước 3 — Cài đặt OpenRGB và Lian Li

```bash
# OpenRGB (bản git — khớp repo này, 0.9+)
sudo pacman -S openrgb-git            # hoặc: paru -S openrgb-git

# Lian Li RGB (quạt + AIO) — bản Linux thay thế L-Connect 3
yay -S lianli-linux-git
```

> 📚 Lian Li Linux: [sgtaziz/lian-li-linux](https://github.com/sgtaziz/lian-li-linux)
> ⚠️ Sau khi cài, `lianli-daemon` phải có trong `PATH` (`which lianli-daemon`).

---

### Bước 4 — Sao chép file vào đúng vị trí

```bash
git clone https://github.com/slimulv1/rgb-config && cd rgb-config

# tạo thư mục đích
mkdir -p ~/.config/systemd/user/lianli-daemon.service.d \
         ~/.local/lib ~/.local/bin ~/.config/openrgb/schemes

# OpenRGB
cp systemd/openrgb.service        ~/.config/systemd/user/openrgb.service
cp local/lib/openrgb-wrapper.sh   ~/.local/lib/openrgb-wrapper.sh
cp local/bin/apply-rgb            ~/.local/bin/apply-rgb
cp schemes/*.rgb                  ~/.config/openrgb/schemes/

# Lian Li — override unit + script + drop-in
cp systemd/lianli-daemon.service                    ~/.config/systemd/user/lianli-daemon.service
cp local/lib/lianli-wrapper.sh                       ~/.local/lib/lianli-wrapper.sh
cp systemd/lianli-daemon.service.d/retry-open-acl.conf \
   ~/.config/systemd/user/lianli-daemon.service.d/retry-open-acl.conf

# Lian Li — config + presets (máy tham chiếu)
mkdir -p ~/.config/lianli
cp lianli/config.json         ~/.config/lianli/config.json
cp lianli/rgb_presets.json    ~/.config/lianli/rgb_presets.json
```

### Bước 5 — Phân quyền

```bash
chmod +x ~/.local/bin/apply-rgb
chmod +x ~/.local/lib/openrgb-wrapper.sh
chmod +x ~/.local/lib/lianli-wrapper.sh
```

> `~/.local/bin` cần nằm trong `PATH`: `echo $PATH | grep .local/bin`.
> Nếu chưa có, thêm vào `~/.bashrc` / `~/.zshrc`:
> ```bash
> export PATH="$HOME/.local/bin:$PATH"
> ```

### Bước 6 — Cấp quyền truy cập device

OpenRGB cần đọc `/dev/i2c-*` **và** mở `/dev/hidraw*` của AURA LED Controller
(mainboard Z690-A được OpenRGB detect qua USB `0b05:19af`, KHÔNG qua i2c).
Lian Li cần đọc/ghi `/dev/hidraw*` của hub riêng (`0cf2:a100`).

Tạo nhóm `lianli` (cần cho udev rule của Lian Li gán quyền) rồi cho user vào **cả 2 nhóm** `i2c` và `lianli`:

```bash
# nhóm lianli (chỉ máy mới — lianli-linux-git không tự tạo)
sudo groupadd -r lianli

# cho user vào cả 2 nhóm
sudo usermod -aG i2c,lianli $USER

# đăng xuất / đăng nhập lại để áp dụng quyền nhóm mới
```

**Rule udev cho AURA** (trong repo tại `udev/60-aura-led.rules` — copy vào `/etc/udev/rules.d/` với quyền root):

```bash
sudo cp udev/60-aura-led.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=hidraw
```

Rule này gán `GROUP="i2c", MODE="0660"` cho hidraw của AURA (`0b05:19af`) —
**giữ quyền dự phòng qua nhóm i2c**, y như `45-i2c-tools.rules` làm cho `/dev/i2c-*`.
Nếu thiếu rule này, OpenRGB chỉ mở được AURA sau khi user đăng nhập GUI
(uaccess ACL), làm mainboard/RAM đổi màu trễ ~1-2 phút sau boot.

> ℹ️ Cơ chế cấp quyền:
> - **uaccess** (login GUI) → `user:USER:rw` — phát sinh trễ sau boot
> - **nhóm** `i2c` → quyền có ngay khi node tạo, kể cả trước login
> Wrapper (`openrgb-wrapper.sh`) nhận cả 2: có ACL user **hoặc** quyền đọc/ghi thực tế là OK.

**Kiểm tra OpenRGB nhận thiết bị:**

```bash
openrgb -l
# kỳ vọng:
#   0: Kingston Fury DDR5 DRAM
#   1: Sapphire Radeon RX 7800 XT Nitro+
#   2: ASUS ROG STRIX Z690-A GAMING WIFI
```

**Đổi màu nhanh (không cần boot service):**

```bash
apply-rgb white       # mặc định
apply-rgb --list      # liệt kê scheme
```

### Bước 7 — Cấu hình Lian Li (device + fan curve)

Cấu hình hiện tại dành cho **hub UNI Hub SL v1 + fan SL120 v1** (USB `0cf2:a100` — ENE LianLi-UNI FAN-SL-v1.8). Đã copy ở Bước 4 vào `~/.config/lianli/`:
- **`config.json`** — fan curve, tốc độ fan, backend `hidraw`, FPS…
- **`rgb_presets.json`** — các preset màu cho fan hub

> ⚠️ **Machine-specific**: cả 2 file chứa device ID (`hid:...`) và temp source (`acpitz_0`) của máy tham chiếu. **Không copy nguyên từ máy khác** — chỉ dùng làm tham chiếu. Với máy mới, nên để `lianli-daemon` tự sinh config rồi sửa theo phần cứng của bạn.

### Bước 8 — Bật service tự chạy lúc boot

> 💡 Cho phép user service chạy **dù chưa đăng nhập** (boot sớm):
> ```bash
> sudo loginctl enable-linger $USER
> ```
> Bỏ qua nếu bạn chỉ muốn RGB bật sau khi đăng nhập.

```bash
systemctl --user daemon-reload

# OpenRGB
systemctl --user enable --now openrgb.service

# Lian Li
systemctl --user enable --now lianli-daemon.service

# kiểm tra
systemctl --user status openrgb.service
systemctl --user status lianli-daemon.service

# log xác nhận OpenRGB đã apply scheme
journalctl -b | grep openrgb-wrapper
# kỳ vọng: OK: 3 controllers; applied scheme 'white'

# log xác nhận Lian Li ready (không cần đợi login)
journalctl -b | grep lianli-wrapper
# kỳ vọng: /dev/hidraw10 ready (after Ns)  — N bé (0–5s)
```

> ℹ️ `systemd/lianli-daemon.service` trong repo là **full override unit** (sao chép nguyên file vào `~/.config/systemd/user/`). Nó bỏ `After=`/`PartOf=graphical-session.target` của package unit — drop-in KHÔNG xóa được các dependency này (systemd merge semantics), nên cần override unit. Lợi ích: daemon **không bị stop khi logout** và không phụ thuộc vào việc session đã mở chưa. Khi nâng cấp package `lianli-linux-git`, override unit vẫn win (user unit ưu tiên hơn package unit).

> ℹ️ Các dòng sau trong log là **bình thường, đừng coi là lỗi**:
> `Dialog Warning: One or more I2C/SMBus interfaces failed to initialize` (một số bus SMBus không tồn tại) và
> `NetworkServer recv_select failed ... closing listener` (client ngắt kết nối) — bỏ qua.

---

## 🎨 Cách dùng

### Đổi màu ngay

```bash
apply-rgb              # scheme mặc định (white)
apply-rgb white        # toàn trắng: RAM 40%, GPU rainbow, pump/mainboard full
apply-rgb rainbow      # rainbow tất cả (GPU "rainbow wave")
apply-rgb breath       # breathing xanh mint
apply-rgb red          # đỏ
apply-rgb xanhtim      # xanh dương
apply-rgb vangxanh     # cam
apply-rgb --list       # liệt kê scheme
```

### Tạo/đổi scheme mới

Sửa file `.rgb` trong `schemes/` — format INI:

```ini
# header: global (áp dụng mọi thiết bị)
MODE=static
COLORS=FFFFFF
BRIGHTNESS=80

[kingston]              # RAM
BRIGHTNESS=40

[sapphire]              # GPU — rainbow tự do
MODE=rainbow wave

[asus|0]                # mainboard onboard (zone 0)
MODE=static
COLORS=FFFFFF
BRIGHTNESS=100

[asus|1]                # ARGB Header 1 = Deepcool LT720 pump (zone 1)
SIZE=22                 # resize zone về 22 LED trước khi set
MODE=static
COLORS=FFFFFF
BRIGHTNESS=100
```

> ⚠️ **Key global phải đặt TRƯỚC** section đầu tiên. Tên section = substring của tên thiết bị (`[kingston]`, `[sapphire]`, `[asus]`); `[tên|zone-index]` để chỉ zone cụ thể.

---

## 🔧 Các mode hoạt động trên từng thiết bị

Không phải device nào cũng hỗ trợ mọi mode — lý do cần section per-device:

| Mode | DRAM | GPU | Mainboard |
|------|:----:|:---:|:---------:|
| `static` | ✅ | ✅ | ✅ |
| `Rainbow` | ✅ | ❌ (dùng `rainbow wave`) | ✅ |
| `breath`/`breathing` | `breath` | ❌ (fallback `static`) | `breathing` |
| `Spectrum` | ✅ | `spectrum cycle` | `spectrum cycle` |

---

## 🔁 Cập nhật file từ repo sau khi sửa

> Clone repo về `~/rgb-config` (để các lệnh dưới đúng) — nếu clone chỗ khác, thay `~/rgb-config` bằng đường dẫn của bạn:
> ```bash
> git clone https://github.com/slimulv1/rgb-config ~/rgb-config
> ```

Sửa file trên máy → copy đè vào repo → commit + push:

```bash
# OpenRGB
cp ~/.local/lib/openrgb-wrapper.sh ~/rgb-config/local/lib/
cp ~/.local/lib/lianli-wrapper.sh  ~/rgb-config/local/lib/
cp ~/.local/bin/apply-rgb          ~/rgb-config/local/bin/
cp ~/.config/openrgb/schemes/*.rgb ~/rgb-config/schemes/
cp ~/.config/systemd/user/openrgb.service        ~/rgb-config/systemd/
cp ~/.config/systemd/user/lianli-daemon.service.d/retry-open-acl.conf \
   ~/rgb-config/systemd/lianli-daemon.service.d/retry-open-acl.conf

# Lian Li config
cp ~/.config/lianli/config.json       ~/rgb-config/lianli/
cp ~/.config/lianli/rgb_presets.json  ~/rgb-config/lianli/

cd ~/rgb-config
git commit -am "update: ..."
git push
```
