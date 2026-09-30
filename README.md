# OpenRGB — RGB Lighting Config (Core64)

Đổi màu RGB trên máy **Core64** bằng cách sửa file text rồi chạy một lệnh — không
cần mở OpenRGB GUI và load profile `.orp` nữa.

Hai hệ thống chạy song song:

- **OpenRGB** — RAM, GPU, mainboard, pump AIO
- **`lianli-daemon`** — fan hub và LED fan Lian Li

---

## 🖥️ Thiết bị

**Qua OpenRGB**

| # | Thiết bị | Tên trong `openrgb -l` |
|---|----------|------------------------|
| 0 | Kingston Fury DDR5 | `Kingston Fury DDR5 DRAM` |
| 1 | Sapphire RX 7800 XT Nitro+ | `Sapphire Radeon RX 7800 XT Nitro+` |
| 2 | Lian Li Uni Hub SL | `Lian Li Uni Hub - SL` — bỏ qua, xem [ghi chú](#-lian-li-hub) |
| 3 | ASUS ROG STRIX Z690-A | `ASUS ROG STRIX Z690-A GAMING WIFI` |

**Ngoài OpenRGB**

| Thiết bị | Điều khiển bằng | Ghi chú |
|----------|-------------------|---------|
| Deepcool LT720 pump | zone 1 của mainboard (ARGB Header 1) | cần `SIZE=22`; fan FK120 **không** có RGB |
| Lian Li Uni Hub SL v1 + 5× SL120 v1 | `lianli-daemon` qua hidraw | màu nằm trong `~/.config/lianli/config.json` |

---

## 📦 Cấu trúc repo

```
systemd/openrgb.service          service OpenRGB lúc boot
systemd/lianli-daemon.service     override unit, không phụ thuộc graphical session
systemd/lianli-daemon.service.d/  drop-in: đi qua wrapper
udev/60-aura-led.rules            AURA (0b05:19af) → GROUP=i2c
local/bin/apply-rgb               đổi màu theo scheme
local/lib/openrgb-wrapper.sh      chờ I801 + AURA rồi mới launch OpenRGB
local/lib/lianli-wrapper.sh       chờ hidraw rồi mới launch daemon
lianli/config.json                fan curve + màu hub
lianli/rgb_presets.json           preset màu
schemes/*.rgb                     7 scheme
```

Sau khi clone về `~/rgb-config`, copy sang chỗ này:

| Từ repo | Đến |
|---------|-----|
| `systemd/*.service`, `systemd/**/*.conf` | `~/.config/systemd/user/` |
| `local/bin/apply-rgb` | `~/.local/bin/apply-rgb` |
| `local/lib/*-wrapper.sh` | `~/.local/lib/` |
| `lianli/*.json` | `~/.config/lianli/` |
| `schemes/*.rgb` | `~/.config/openrgb/schemes/` |
| `udev/60-aura-led.rules` | `/etc/udev/rules.d/` |

---

## 🛠️ Cài đặt

> Đã chạy sẵn trên Core64. Phần dưới dành cho máy mới hoặc cài lại.

### Bước 1 — I2C

OpenRGB nói chuyện với RAM/GPU/mainboard qua bus I2C/SMBus.

```bash
sudo pacman -S i2c-tools
sudo modprobe i2c-dev i2c-i801

sudo tee /etc/modules-load.d/i2c.conf > /dev/null <<'EOF'
i2c-dev
i2c-i801
EOF
```

### Bước 2 — Nhường bus SPD cho RAM DDR5

Module kernel `spd5118` giữ bus SPD nên OpenRGB không thấy RAM. Chặn nó:

```bash
echo "blacklist spd5118" | sudo tee /etc/modprobe.d/blacklist-spd5118.conf
sudo mkinitcpio -P
sudo reboot
```

Sau reboot, `lsmod | grep spd5118` phải **không có** dòng nào.

> Muốn giữ `spd5118` thì làm ngược: thay dòng trên bằng service `rmmod` lúc
> boot. Nhưng `rmmod` chỉ có tác dụng tới khi module được nạp lại, còn blacklist
> chặn từ gốc nên bền hơn — và chỉ nên chọn một trong hai.

### Bước 3 — Gói phần mềm

```bash
paru -S openrgb-git lianli-linux-git
```

Hai gói này đều có trong AUR, `paru` tự lo phần phụ thuộc.

**`mbedtls3` là bắt buộc.** OpenRGB chỉ chạy với mbedtls 3.x — chính
`OpenRGB.pro` của upstream ghi rõ *"will not work with mbedtls 4.x"*, còn Arch
đang ở 4.x. Gói `mbedtls3` đặt thư viện thật vào `/usr/lib/mbedtls3/` rồi thả
symlink tương thích vào `/usr/lib` (`libmbedx509.so.7` →
`/usr/lib/mbedtls3/libmbedx509.so.3.6.7`); nhờ vậy binary openrgb vẫn tìm thấy
thư viện. Hai gói không đụng file nhau nên cài song song, không cần gỡ
`mbedtls` 4.x.

Bản AUR của `openrgb-git` đã khai báo sẵn dependency này. Nếu lỡ cài `openrgb`
từ kho nhị phân thì phải tự thêm: `sudo pacman -S mbedtls3`.

> ⚠️ `paru -S <gói>` sẽ lấy bản trong kho sync nếu kho đó có đúng tên gói — khi đó
> nó **không** build từ AUR. Muốn ép build từ AUR thì thêm `-a`:
> `paru -a -S <gói>`. Đã dính: `paru -S openrgb-git` lấy bản trong `cachyos`, còn
> `paru -a -S lianli-linux-git` mới build thật.

### Bước 4 — Copy file, phân quyền, cấp quyền device

```bash
cd ~/rgb-config || exit 1
mkdir -p ~/.config/systemd/user/lianli-daemon.service.d ~/.config/lianli \
         ~/.local/{lib,bin} ~/.config/openrgb/schemes

cp systemd/openrgb.service systemd/lianli-daemon.service ~/.config/systemd/user/
cp systemd/lianli-daemon.service.d/retry-open-acl.conf \
   ~/.config/systemd/user/lianli-daemon.service.d/
cp local/lib/*.sh ~/.local/lib/
cp local/bin/apply-rgb ~/.local/bin/
cp lianli/*.json ~/.config/lianli/
cp schemes/*.rgb ~/.config/openrgb/schemes/
chmod +x ~/.local/bin/apply-rgb ~/.local/lib/*.sh

# tạo sẵn 2 file lock trong /run, nếu không daemon phải đợi tới lần boot sau
sudo systemd-sysusers
sudo usermod -aG i2c,lianli $USER

sudo cp udev/60-aura-led.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=hidraw
```

> User `lianli` và nhóm cùng tên do hook cài gói tạo qua `sysusers.d`, nên
> `usermod` không bị lỗi. Dòng `systemd-sysusers` ở trên tạo hai file lock
> `/run/lianli-{daemon,control}.lock` — bỏ thì daemon phải chờ tới lần boot sau
> mới lên được, vì rule trong `tmpfiles.d` chỉ chạy lúc boot.
>
> `usermod` chỉ ghi vào `/etc/group`, nên **thoát hẳn rồi đăng nhập lại** thì nhóm
> mới có hiệu lực. Reboot ở Bước 2 không giúp được — nó nằm trước `usermod`.

`~/.local/bin` phải nằm trong `PATH` (`echo $PATH | grep .local/bin`). Nếu chưa
có thì thêm vào `~/.bashrc` hoặc `~/.zshrc`:
```bash
export PATH="$HOME/.local/bin:$PATH"
```

### Bước 5 — Bật service

```bash
sudo loginctl enable-linger $USER   # để service lên trước lúc đăng nhập

systemctl --user daemon-reload
systemctl --user enable --now openrgb.service lianli-daemon.service
```

### Bước 6 — Kiểm tra

```bash
openrgb -l                    # phải thấy đủ 4 mục
apply-rgb white               # thử đổi màu
```

Màu mặc định đã được wrapper áp lúc boot, nên `apply-rgb` ở đây chỉ để thử tay.

### Bước 7 — Cấu hình Lian Li

`config.json` trong repo là của máy tham chiểu, chứa device ID riêng. Daemon sẽ
tự sửa về ID thật khi chạy lần đầu (`Migrating wired device identities`), nhưng
nếu bạn dùng phần cứng khác hãy để daemon tự sinh config rồi chỉnh lại theo
tài liệu upstream.

---

## 🎨 Đổi màu

```bash
apply-rgb              # scheme mặc định (white)
apply-rgb rainbow      # rainbow mọi thiết bị
apply-rgb breath       # nhấp nháy xanh mint
apply-rgb red          # đỏ
apply-rgb xanhtim      # xanh dương
apply-rgb vangxanh     # cam
apply-rgb --list       # liệt kê scheme
```

`apply-rgb` cần OpenRGB server đang chạy. Nếu nó báo
`error: no devices listed by openrgb -l` thì service chưa lên:
`systemctl --user status openrgb`.

### 🔹 Lian Li hub

`apply-rgb` mặc định **bỏ qua** hub. Không chỉ vì hub thuộc về `lianli-daemon` —
mode `static` của hub làm `openrgb` **abort** (`rc=134`), kéo theo `apply-rgb`
trả 1 và `openrgb.service` restart vô hạn (`Restart=always`).

Bỏ thêm thiết bị khác nếu cần:

```bash
APPLY_RGB_SKIP="Lian Li,Tên thiết bị" apply-rgb white
```

### Tạo scheme riêng

Sửa file `.rgb` trong `schemes/`:

```ini
MODE=static              # key global, phải đặt TRƯỚC section đầu tiên
COLORS=FFFFFF
BRIGHTNESS=80

[kingston]               # tên section = substring tên thiết bị
BRIGHTNESS=40

[sapphire]               # GPU để rainbow tự do
MODE=rainbow wave

[asus|0]                 # mainboard onboard
MODE=static
COLORS=FFFFFF
BRIGHTNESS=100

[asus|1]                 # ARGB Header 1 = pump
SIZE=22                  # resize về 22 LED
MODE=static
COLORS=FFFFFF
BRIGHTNESS=100
```

### Mode nào dùng được

| Mode | DRAM | GPU | Mainboard |
|------|:----:|:---:|:---------:|
| `static` | ✅ | ✅ | ✅ |
| `Rainbow` | ✅ | `rainbow wave` | ✅ |
| `breath` | `breath` | fallback `static` | `breathing` |
| `Spectrum` | ✅ | `spectrum cycle` | `spectrum cycle` |

---

## 🔧 Xử lý sự cố

| Triệu chứng | Nguyên nhân | Cách sửa |
|-------------|-------------|----------|
| `openrgb -l` không thấy Kingston | `spd5118` còn giữ bus SPD | blacklist + `mkinitcpio -P` + reboot |
| `openrgb failed for [2] zone=all` rồi restart lặp lại | `apply-rgb` set `static` lên Lian Li hub (abort) | dùng bản `apply-rgb` có `APPLY_RGB_SKIP` |
| `error while loading shared libraries: libmbedx509.so.7` | thiếu mbedtls 3.x | `sudo pacman -S mbedtls3` |
| `no devices listed by openrgb -l` khi chạy `apply-rgb` | service chưa lên | `systemctl --user status openrgb` |

Dòng log này **bình thường, đừng điều tra**:

- `I2C/SMBus interfaces failed to initialize` — bus không tồn tại
- `NetworkServer recv_select failed ... closing listener` — client ngắt kết nối
- `[Kingston ...] 61 failed to set register &30=01` — controller DDR5 từ chối vài
  thanh ghi; wrapper vẫn coi là thành công. Chỉ lo nếu đèn RAM hiển thị sai.

Nếu log Lian Li không có dòng RGB ở mức `info`, đó không phải lỗi — mức đó
không in. Xem mục dưới.

### Nhìn thấy LED đang chạy

Log mức `info` **không** in dòng RGB, nên log trông như chưa hoạt động dù đèn
đã đổi. Muốn thấy rõ thì bật debug tạm:

```bash
mkdir -p ~/.config/systemd/user/lianli-daemon.service.d
echo '[Service]
Environment=RUST_LOG=info,lianli_devices=debug' \
  > ~/.config/systemd/user/lianli-daemon.service.d/debug.conf
systemctl --user daemon-reload && systemctl --user restart lianli-daemon
journalctl --user -u lianli-daemon -n 30 --no-pager | grep -i 'set group'
```

Màu in ra phải khớp `~/.config/lianli/config.json`. Xong thì gỡ drop-in, log
debug rất ồn:

```bash
rm ~/.config/systemd/user/lianli-daemon.service.d/debug.conf
systemctl --user daemon-reload && systemctl --user restart lianli-daemon
```

### Module kernel `evdi` (tuỳ chọn)

Panel **Installation Health** trong `lianli-gui` sẽ báo ô *Optional display
module: evdi* ở trạng thái **Failed** nếu bạn chưa cài module kernel này. Đây là
dự kiến với phần cứng trong repo này — bỏ qua được.

**Khi nào thật sự cần.** `evdi` tạo một màn hình ảo để đẩy hình desktop lên
**LCD của hub**. Repo này dùng **Uni Hub SL** — chỉ có quạt và LED, không có
màn hình, và `config.json` để `lcds: []`. Nên fan và RGB không cần `evdi`:

> *"Ordinary fan/RGB control does not require an EVDI kernel module."*
> — [docs/troubleshooting.md](https://github.com/sgtaziz/lian-li-linux/blob/main/docs/troubleshooting.md)

Máy chạy dwm/X11 cũng không dùng tới: `evdi` chỉ là đường dự phòng cho
desktop không phải Hyprland, mà Hyprland native headless vốn đã không cần nó.

**Vì sao vẫn báo Failed.** Health phân biệt hai ca:

| Ca | Health hiện |
|----|------------|
| Module **chưa** đăng ký | `N/A` |
| Module **đã đăng ký** nhưng chưa build cho kernel đang chạy | `Failed` |

`evdi-dkms` đăng ký module rồi, nhưng DKMS chỉ build cho những kernel có sẵn
headers. Kiểm tra:

```bash
uname -r
dkms status | grep evdi
ls -d /lib/modules/$(uname -r)/build    # không có thì thiếu kernel dev package
```

**Cài thì làm sao.** Chỉ cần gói headers của kernel đang chạy. DKMS tự build
lại qua hook `70-dkms-install.hook` nên không phải gọi tay:

```bash
sudo pacman -S linux-arisa-headers
sudo modprobe evdi
dkms status | grep evdi                  # phải có dòng khớp uname -r
```

Đổi kernel khác thì thay tên gói: `linux-cachyos-headers`,
`linux-cachyos-lts-headers`. Sau khi cài headers, log pacman sẽ có:

```
running '70-dkms-install.hook'...
==> dkms install --no-depmod evdi/1.15.1 -k 7.2.8-lqx1-1-arisa
```

Module đã build thì vẫn phải `modprobe` hoặc reboot mới nạp — hook không tự
`modprobe`. Với kernel mới cài sau này, DKMS build sẵn lúc cài kernel, chỉ cần
reboot.

> Không cài cũng không sao. Thêm module kernel vào máy đang chạy ổn chỉ để một
> ô trong bảng Health chuyển từ Failed sang Passed thì không đáng.

### Vài điểm dễ nhầm

- **Có GDM thì lúc boot daemon restart vài lần** rồi mới ổn định. Greeter
  chạy session tạm nên daemon không tìm thấy config. Không phải lỗi cấu hình.
- **`rgb.openrgb_port` trong config Lian Li không được dùng** khi
  `rgb.openrgb_server: false` — lúc đó daemon tự điều khiển hub, không qua
  OpenRGB server. Chỉ đổi khi bật `openrgb_server: true`.
- **Override unit của `lianli-daemon`** copy nguyên file vào
  `~/.config/systemd/user/` chứ không dùng drop-in, vì drop-in không xoá được
  `After=`/`PartOf=graphical-session.target` của package unit. Nhờ vậy daemon
  không bị stop khi logout.

---

## 🔁 Đưa thay đổi trên máy về repo

```bash
cp ~/.local/bin/apply-rgb          ~/rgb-config/local/bin/
cp ~/.local/lib/*-wrapper.sh       ~/rgb-config/local/lib/
cp ~/.config/openrgb/schemes/*.rgb ~/rgb-config/schemes/
cp ~/.config/lianli/*.json         ~/rgb-config/lianli/
cp ~/.config/systemd/user/openrgb.service        ~/rgb-config/systemd/
cp ~/.config/systemd/user/lianli-daemon.service.d/retry-open-acl.conf \
   ~/rgb-config/systemd/lianli-daemon.service.d/

cd ~/rgb-config && git commit -am "update: ..." && git push
```
