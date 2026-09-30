# OpenRGB — RGB

Đổi màu đèn trên máy bằng cách sửa một file text rồi chạy một lệnh.
Không cần mở OpenRGB GUI, không cần load profile `.orp`.

Hai hệ thống chạy song song:

- **OpenRGB** — RAM, GPU, mainboard, pump AIO
- **`lianli-daemon`** — fan hub và LED fan Lian Li

---

## 🖥️ Thiết bị

| # | Thiết bị | `openrgb -l` hiện ra | Ghi chú |
|---|----------|-----------------------|---------|
| 0 | Kingston Fury DDR5 | `Kingston Fury DDR5 DRAM` | chỉ thấy sau khi làm Bước 2 |
| 1 | Sapphire RX 7800 XT Nitro+ | `Sapphire Radeon RX 7800 XT Nitro+` | |
| 2 | Lian Li Uni Hub SL | `Lian Li Uni Hub - SL` | `apply-rgb` **bỏ qua**, xem [đây](#lian-li-hub) |
| 3 | ASUS ROG STRIX Z690-A | `ASUS ROG STRIX Z690-A GAMING WIFI` | |

Ngoài ra:

- **Deepcool LT720 pump** nối vào ARGB Header 1 của mainboard, tức zone 1 — scheme phải resize về `SIZE=22` mới khớp. Fan FK120 đi kèm không có đèn.
- **Lian Li Uni Hub SL v1 + 5× fan SL120** do `lianli-daemon` điều khiển qua hidraw. Màu nằm trong `~/.config/lianli/config.json`, không qua OpenRGB.

---

## 🛠️ Cài đặt

> Máy tham chiếu đã chạy sẵn. Phần dưới dành cho máy mới hoặc khi cài lại.

```bash
git clone https://github.com/slimulv1/rgb-config ~/rgb-config
```

### 1. I2C

OpenRGB nói chuyện với RAM, GPU, mainboard qua bus I2C.

```bash
sudo pacman -S i2c-tools
sudo modprobe i2c-dev i2c-i801

sudo tee /etc/modules-load.d/i2c.conf > /dev/null <<'EOF'
i2c-dev
i2c-i801
EOF
```

### 2. Nhường bus SPD cho RAM DDR5

Module kernel `spd5118` giữ bus SPD, nên OpenRGB không thấy RAM. Chặn nó rồi
reboot:

```bash
echo "blacklist spd5118" | sudo tee /etc/modprobe.d/blacklist-spd5118.conf
sudo mkinitcpio -P
sudo reboot
```

Sau reboot, `lsmod | grep spd5118` phải không có dòng nào.

> Cần `spd5118` cho việc khác thì đổi sang service `rmmod` lúc boot. Nhưng
> `rmmod` chỉ có tác dụng khi module được nạp lại, còn blacklist chặn từ gốc —
> hai cách không dùng cùng lúc.

### 3. Gói phần mềm

```bash
paru -S openrgb-git lianli-linux-git
```

Hai gói nằm trong AUR, `paru` tự kéo phụ thuộc. Riêng **`mbedtls3` là bắt
buộc**: OpenRGB chỉ chạy với mbedtls 3.x, còn Arch đang ở 4.x. Gói này đặt thư
viện vào `/usr/lib/mbedtls3/` rồi thả symlink tương thích vào `/usr/lib`, nên
binary openrgb vẫn tìm thấy. Hai gói không đụng file nhau — không cần gỡ
`mbedtls` 4.x đi.

> Nếu lỡ cài `openrgb` từ kho nhị phân, tự thêm `sudo pacman -S mbedtls3`.
>
> `paru -S <gói>` sẽ lấy bản trong kho sync nếu kho đó có đúng tên gói, lúc đó
> **không** build từ AUR. Ép build thì thêm `-a`: `paru -a -S <gói>`.

### 4. Copy file và cấp quyền

```bash
cd ~/rgb-config || exit 1
mkdir -p ~/.config/systemd/user/lianli-daemon.service.d ~/.config/lianli \
         ~/.local/{lib,bin} ~/.config/openrgb/schemes

cp systemd/openrgb.service systemd/lianli-daemon.service ~/.config/systemd/user/
cp systemd/lianli-daemon.service.d/retry-open-acl.conf \
   ~/.config/systemd/user/lianli-daemon.service.d/
cp local/lib/*.sh ~/.local/lib/
cp local/bin/apply-rgb ~/.local/bin/
cp schemes/*.rgb ~/.config/openrgb/schemes/
chmod +x ~/.local/bin/apply-rgb ~/.local/lib/*.sh

# KHÔNG copy đè config.json: file đó chứa màu LED và device ID của máy này.
# Chỉ copy khi máy chưa có, còn không giữ nguyên.
[ -e ~/.config/lianli/config.json ] || cp lianli/config.json ~/.config/lianli/
cp lianli/rgb_presets.json ~/.config/lianli/

# tạo /run/lianli-{daemon,control}.lock nếu thiếu
sudo systemd-tmpfiles --create /usr/lib/tmpfiles.d/lianli.conf
sudo usermod -aG i2c,lianli "$(id -un)"

sudo cp udev/60-aura-led.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=hidraw
```

`chmod +x` là bắt buộc, không phải cho đẹp: `openrgb-wrapper.sh` gọi thẳng
`"$APPLY_RGB"`, và cả hai unit đều đặt `ExecStart=` trỏ vào wrapper. Thiếu `+x`
là `Permission denied` ngay lúc service khởi động.

**Vì sao không copy đè `config.json`.** File trong repo là của máy tham chiếu.
Màu LED bạn đang để nằm ở `rgb.devices[].effect_memory`, và lệnh copy sẽ xoá
mất nó. Device ID thì daemon tự sửa lại được, còn màu thì không — phải chạy
`apply-rgb` lại. Muốn cài lại từ đầu thì `rm` file cũ đi trước.

**Hai file lock trong `/run`** do `/usr/lib/tmpfiles.d/lianli.conf` tạo, nên
lệnh phải là `systemd-tmpfiles`, không phải `systemd-sysusers` — cái sau chỉ
lo `/etc/passwd` và `/etc/group`, không đụng `/run`. Lệnh ở trên còn chỉ đích
danh đúng file của lianli, thay vì quét toàn bộ `tmpfiles.d` của hệ thống.

Thực ra thường không cần chạy: `21-systemd-tmpfiles.hook` đã tạo sẵn lúc cài
gói, và `/run` là tmpfs nên `systemd-tmpfiles-setup.service` tạo lại mỗi lần
boot. Dòng này là để phòng khi cài lại mà không reboot.

`usermod` chỉ ghi vào `/etc/group`, nên **thoát hẳn rồi đăng nhập lại** thì nhóm
mới có hiệu lực. Reboot ở bước 2 không giúp được, vì nó nằm trước `usermod`.

`udevadm trigger` bắn lại rule cho **mọi** node hidraw, không riêng node AURA —
không thu hẹp được, vì `idVendor` nằm trên device chứ không nằm trên
`/sys/class/hidraw/hidrawN/`, mà `--attr-match` chỉ đọc chỗ sau. Chạy lại rule
cũ chỉ cho kết quả y hệt nên vô hại.

`~/.local/bin` phải nằm trong `PATH` (`echo $PATH | grep .local/bin`). Nếu chưa
có thì thêm vào `~/.bashrc`:

```bash
export PATH="$HOME/.local/bin:$PATH"
```

### 5. Bật service

```bash
sudo loginctl enable-linger "$(id -un)"   # để service lên trước lúc đăng nhập

systemctl --user daemon-reload
systemctl --user enable --now openrgb.service lianli-daemon.service

openrgb -l                            # phải thấy đủ 4 mục
```

Lệnh nào cũng phải có `--user`. Máy này có **hai** unit cùng tên
`openrgb.service`: gói `openrgb-git` đặt một bản ở
`/usr/lib/systemd/system/` (system, `disabled`), còn repo đặt bản user ở
`~/.config/systemd/user/` (`enabled`). `systemctl status openrgb` không có
`--user` sẽ ra bản system — đang `disabled`, trông như service chết trong khi
bản đang chạy vẫn ổn.

`~/.config/systemd/user/` cũng đè lên bản của gói, nên `lianli-daemon.service`
trong repo phải là bản sao **đầy đủ** chứ không phải drop-in. Nếu gói nâng cấp
thì bản trong `~/.config/` vẫn thắng, nên nhớ đồng bộ lại từ
`/usr/lib/systemd/user/lianli-daemon.service`.

Dùng phần cứng khác máy tham chiếu thì để daemon tự sinh `config.json` rồi chỉnh
theo tài liệu upstream. Lần chạy đầu daemon sẽ tự sửa device ID về ID thật và
báo `Migrating wired device identities`.

---

## 🎨 Đổi màu

```bash
apply-rgb                 # scheme mặc định (white)
apply-rgb rainbow         # rainbow mọi thiết bị
apply-rgb breath          # nhấp nháy xanh mint
apply-rgb daquang2        # xanh mint đặc
apply-rgb red             # đỏ
apply-rgb xanhtim         # xanh dương
apply-rgb vangxanh        # cam
apply-rgb --list          # liệt kê scheme
```

`apply-rgb` cần OpenRGB server đang chạy. Nếu nó báo
`error: no devices listed by openrgb -l` thì service chưa lên:
`systemctl --user status openrgb`.

### Lian Li hub

`apply-rgb` mặc định **bỏ qua** hub. Không chỉ vì hub thuộc về
`lianli-daemon` — mode `static` của hub làm `openrgb` abort (`rc=134`), kéo theo
`apply-rgb` trả 1, mà `openrgb.service` có `Restart=always` nên thành restart
vô hạn.

Bỏ thêm thiết bị khác nếu cần:

```bash
APPLY_RGB_SKIP="Lian Li,Tên thiết bị" apply-rgb white
```

### Tạo scheme riêng

Sửa file `.rgb` trong `schemes/`:

```ini
MODE=static              # key global — phải đặt TRƯỚC section đầu tiên
COLORS=FFFFFF
BRIGHTNESS=80

[kingston]               # tên section là substring của tên thiết bị
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

Mode không phải thiết bị nào cũng có, nên mới cần section riêng:

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
| `! openrgb failed for [2] zone=all` rồi restart lặp lại | `apply-rgb` set `static` lên hub (abort) | xem [Lian Li hub](#lian-li-hub) |
| `error while loading shared libraries: libmbedx509.so.7` | thiếu mbedtls 3.x | `sudo pacman -S mbedtls3` |
| `no devices listed by openrgb -l` | service chưa lên | `systemctl --user status openrgb` |

Những dòng log này **bình thường, đừng điều tra**:

- `I2C/SMBus interfaces failed to initialize` — bus không tồn tại
- `NetworkServer recv_select failed ... closing listener` — client ngắt kết nối
- `[Kingston ...] 61 failed to set register &30=01` — controller DDR5 từ chối vài
  thanh ghi, wrapper vẫn coi là thành công. Chỉ lo nếu đèn RAM hiện sai.

**Log Lian Li không có dòng RGB ở mức `info` là bình thường** — mức đó không
in. Muốn thấy daemon có thật sự ghi LED không thì bật debug tạm:

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

### `evdi` trong Installation Health

Panel Health báo ô *Optional display module: evdi* ở trạng thái **Failed** là
chuyện bình thường ở đây. `evdi` tạo màn hình ảo để đẩy desktop lên **LCD của
hub** — mà Uni Hub SL không có LCD, và `config.json` để `lcds: []`. Tài liệu
cũng nói thẳng: *"Ordinary fan/RGB control does not require an EVDI kernel
module."* Máy chạy dwm/X11 cũng không dùng tới.

Health hiện `Failed` (chứ không phải `N/A`) vì `evdi-dkms` **đã đăng ký** module
nhưng DKMS chỉ build cho kernel có sẵn headers. Muốn xanh ô đó thì:

```bash
sudo pacman -S linux-arisa-headers   # đổi theo kernel đang chạy
sudo modprobe evdi
dkms status | grep evdi              # phải có dòng khớp uname -r
```

Cài headers là hook `70-dkms-install.hook` tự gọi `dkms install`, nên không cần
gọi tay. Hook không `modprobe` — vẫn phải modprobe hoặc reboot.

### Ghi chú

- **Có GDM thì lúc boot daemon restart vài lần** rồi mới ổn định. Greeter chạy
  session tạm nên daemon không tìm thấy config. Không phải lỗi cấu hình.
- **`rgb.openrgb_port`** không được dùng khi `rgb.openrgb_server: false` — lúc đó
  daemon tự điều khiển hub, không qua OpenRGB server. Chỉ đổi khi bật
  `openrgb_server: true`.
- **Unit của `lianli-daemon`** phải copy nguyên file vào `~/.config/systemd/user/`
  chứ không dùng drop-in: systemd không cho drop-in xoá `After=` và
  `PartOf=graphical-session.target` của package unit. Nhờ vậy daemon không bị
  stop khi logout.

---

## 🔁 Đưa thay đổi trên máy về repo

```bash
cp ~/.local/bin/apply-rgb           ~/rgb-config/local/bin/
cp ~/.local/lib/*-wrapper.sh        ~/rgb-config/local/lib/
cp ~/.config/openrgb/schemes/*.rgb  ~/rgb-config/schemes/
cp ~/.config/lianli/*.json          ~/rgb-config/lianli/
cp ~/.config/systemd/user/openrgb.service        ~/rgb-config/systemd/
cp ~/.config/systemd/user/lianli-daemon.service.d/retry-open-acl.conf \
   ~/rgb-config/systemd/lianli-daemon.service.d/

cd ~/rgb-config && git commit -am "update: ..." && git push
```
