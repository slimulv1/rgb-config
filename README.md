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
- **Lian Li Uni Hub SL v1 + 5× fan SL120** do `lianli-daemon` điều khiển qua hidraw. Màu nằm trong `~/.config/lianli/config.json`, không qua OpenRGB. Lưu ý màu `#496DFF` trong file đó là màu trắng *đã bù lỗi LED*, xem [`#496DFF` trên fan SL120 v1](#496dff-trên-fan-sl120-v1--cân-bằng-lỗi-phần-cứng).

---

## 🛠️ Cài đặt

> Máy tham chiếu đã chạy sẵn. Phần dưới dành cho máy mới hoặc khi cài lại.

```bash
git clone https://github.com/slimulv1/rgb-config ~/rgb-config
```

### 1. I2C

RAM đi qua bus I2C (`i2c-i801` trên mainboard), nên OpenRGB cần quyền đọc bus đó.
Mainboard thì **không** qua I2C — nó nhận qua USB HID của AURA LED Controller
(`0b05:19af`), xử lý ở Bước 4. GPU thì đi USB.

```bash
sudo pacman -S i2c-tools
sudo modprobe i2c-dev i2c-i801 2>/dev/null || true

sudo tee /etc/modules-load.d/i2c.conf > /dev/null <<'EOF'
i2c-dev
i2c-i801
EOF
```

`modprobe` có thể báo "module not found" trên kernel build sẵn (`CONFIG_I2C_CHARDEV=y`,
`CONFIG_I2C_I801=y` — đây là mặc định của kernel Arch/CachyOS). Không phải lỗi:
module đã nằm sẵn trong kernel. Chỉ cần đảm bảo `modprobe i2c-dev` tạo ra
`/dev/i2c-*`.

**`i2c-tools` không phải lúc nào cũng thừa** — nó cài kèm hai thứ mà Bước 4 cần:

- `/usr/lib/sysusers.d/i2c-tools.conf` → tạo group `i2c`. Thiếu group này thì
  `usermod -aG i2c` báo `group 'i2c' does not exist`.
- `/usr/lib/udev/rules.d/45-i2c-tools.rules` → `GROUP="i2c", MODE="0660"` cho
  `/dev/i2c-*`. Đây là đường truy cập **không cần login**, khác với ACL
  `uaccess` chỉ có sau khi mở phiên đồ hoạ.

```bash
getent group i2c                              # phải có dòng i2c:x:...:
ls -l /dev/i2c-9                             # root:i2c crw-rw---- (major 89)
```

### 2. Nhường bus SPD cho RAM DDR5

Module kernel `spd5118` giữ bus SPD, nên OpenRGB không thấy RAM. Chặn nó:

```bash
echo "blacklist spd5118" | sudo tee /etc/modprobe.d/blacklist-spd5118.conf
```

Initramfs có cần build lại không? **Tùy.** Hook `modconf` chép
`/etc/modprobe.d/*.conf` vào initramfs, nên blacklist có hiệu lực lúc early boot
mà không cần build lại — **trừ khi** `spd5118.ko` nằm sẵn trong initramfs, lúc đó
phải build lại để blacklist vào (hoặc để module biến mất khỏi đó). Kiểm:

```bash
# 1. Layout /boot khác nhau theo bootloader — tìm file initramfs:
sudo find /boot -name 'initramfs*' -o -name 'initrd*'

# 2. Dán path tìm được vào biến, rồi xem bên trong (cần sudo; /boot là 0700):
INITRD=/boot/$(cat /etc/machine-id)/linux-arisa/initramfs
sudo lsinitcpio "$INITRD" | grep -i spd5118
```

Kết quả trên máy tham chiếu (bootloader Limine, initramfs theo machine-id):

```bash
$ sudo lsinitcpio /boot/$(cat /etc/machine-id)/linux-arisa/initramfs | grep -i spd5118
etc/modprobe.d/blacklist-spd5118.conf
```

Chỉ có **file blacklist**, không có `spd5118.ko` — hook `autodetect` chỉ nhét
module cần để boot. Vậy blacklist trên root fs là đủ để chặn, và
`mkinitcpio` **không cần chạy**. Nếu bạn thấy cả `spd5118.ko` trong đó thì mới
cần:

```bash
sudo mkinitcpio -P
```

> **`-P` nghĩa là `--allpresets`, không phải "preset mặc định".** Nó chỉ rebuild
> preset có trong `/etc/mkinitcpio.d/`. Máy tham chiếu chạy kernel `linux-arisa`
> nhưng preset trong thư mục đó là `linux-cachyos.preset` +
> `linux-cachyos-lts.preset` — **không preset nào cho arisa**, nên `mkinitcpio -P`
> không đụng tới initramfs kernel đang chạy. Kiểm trước:
>
> ```bash
> uname -r
> ls /etc/mkinitcpio.d/          # phải có preset khớp kernel trên
> ```

Sau khi blacklist xong (và build lại nếu cần) thì reboot:

```bash
sudo reboot
```

Rồi kiểm:

```bash
lsmod | grep spd5118            # phải không có dòng nào
openrgb -l | grep Kingston      # phải thấy RAM
```

> Cần `spd5118` cho việc khác thì đổi sang service `rmmod` lúc boot. Nhưng
> `rmmod` chỉ có tác dụng khi module được nạp lại, còn blacklist chặn từ gốc —
> hai cách không dùng cùng lúc.

### 3. Gói phần mềm

```bash
sudo pacman -S openrgb
paru -S lianli-linux-git
```

`openrgb` lấy từ kho nhị phân, không phải AUR. Đừng dùng `openrgb-git` trong AUR:
bản đó build với mbedtls 4.x còn OpenRGB chỉ chạy được với mbedtls 3.x. Gói
`openrgb` của kho chính đã khai báo `mbedtls3` là phụ thuộc cứng nên `pacman`
tự kéo theo — không cài thêm gì:

```bash
$ pacman -Qi openrgb | grep -E '^(Name|Version|Depends)'
Name        : openrgb
Version     : 1.0-2.1
Depends On  : glibc libgcc libstdc++ qt6-base libusb hidapi mbedtls3 hicolor-icon-theme
```

`lianli-linux-git` kéo theo `evdi-dkms` và `ffmpeg` làm phụ thuộc. `evdi` ở
đây **không cần thiết** cho fan/RGB — xem [mục evdi](#evdi-trong-installation-health).

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

# KHÔNG copy đè 2 file JSON: chúng chứa cấu hình riêng của máy này.
# Chỉ copy khi máy chưa có, còn không thì giữ nguyên.
[ -e ~/.config/lianli/config.json ]      || cp lianli/config.json      ~/.config/lianli/
[ -e ~/.config/lianli/rgb_presets.json ] || cp lianli/rgb_presets.json ~/.config/lianli/

# --- Group `lianli` + 2 file lock ---
#
# Gói lianli-linux-git 1.1.4-1 KHÔNG ship 2 file rule này (lỗi đóng gói AUR).
# Thiếu thì daemon không lên được, xem [Vì sao phải tự cài 2 file rule].
# `||` để không đè bản gói, phòng khi bản gói sau này bổ sung.
[ -f /usr/lib/tmpfiles.d/lianli.conf ] || \
  sudo install -Dm644 packaging/tmpfiles.d/lianli.conf /usr/lib/tmpfiles.d/lianli.conf
[ -f /usr/lib/sysusers.d/lianli.conf ] || \
  sudo install -Dm644 packaging/sysusers.d/lianli.conf /usr/lib/sysusers.d/lianli.conf

sudo systemd-sysusers /usr/lib/sysusers.d/lianli.conf   # tạo user + group lianli
sudo systemd-tmpfiles --create /usr/lib/tmpfiles.d/lianli.conf
sudo usermod -aG i2c,lianli "$(id -un)"

# --- Rules udev ---
#
# Đặt SAU khi đã tạo group: udev nạp rules một lần lúc start, và group phải
# tồn tại trước khi nạp. Nạp lúc group còn chưa có thì token GROUP="lianli"
# không resolve được gid và bị bỏ qua im lặng — xem [Node hidraw vẫn root:root].
sudo cp udev/60-aura-led.rules /etc/udev/rules.d/
sudo udevadm control --reload-rules
sudo udevadm trigger --subsystem-match=hidraw
```

`chmod +x` là bắt buộc, không phải cho đẹp: `openrgb-wrapper.sh` gọi thẳng
`"$APPLY_RGB"`, và cả hai unit đều đặt `ExecStart=` trỏ vào wrapper. Thiếu `+x`
là `Permission denied` ngay lúc service khởi động.

**Vì sao không copy đè 2 file JSON trong `lianli/`.** Chúng là cấu hình riêng của
máy tham chiếu, không phải template:

- `config.json` — màu LED nằm ở `rgb.devices[].zones[].effect.colors`, lệnh copy
  sẽ xoá mất màu bạn đang để. Device ID thì daemon tự sửa lại được, còn màu thì
  không — phải chạy `apply-rgb` lại.
- `rgb_presets.json` — là bảng màu bạn tự tạo trong GUI. Copy đè là mất.

Cài lại từ đầu thì `rm` file cũ đi trước:

```bash
rm ~/.config/lianli/config.json ~/.config/lianli/rgb_presets.json
```

#### Vì sao phải tự cài 2 file rule

Hai file trong `packaging/` là **bắt buộc**, không phải tuỳ chọn. Gặp lỗi này:

```
Failed to read '/usr/lib/tmpfiles.d/lianli.conf': No such file or directory
usermod: group 'lianli' does not exist
```

Nguyên nhân là lỗi đóng gói AUR, không phải máy bạn: `lianli-linux-git 1.1.4-1`
chỉ cài 7 path (`pacman -Ql lianli-linux-git` xem lại), **không** kèm
`tmpfiles.d/lianli.conf` lẫn `sysusers.d/lianli.conf`. Trong khi đó:

| Thiếu | Hậu quả |
|-------|----------|
| `tmpfiles.d/lianli.conf` | daemon **từ chối khởi động**, restart loop |
| `sysusers.d/lianli.conf` | không có group `lianli`, `usermod` abort |

Hai chỗ đều là điều kiện tiên quyết mà bản thân gói đòi hỏi:

- `60-lianli.rules` của gói dùng `GROUP="lianli"` ở **cả 22 dòng**.
- daemon 1.1.4 giữ shared hardware lock ở `/run/lianli-daemon.lock` và **exit 1
  nếu mở không được**. Log lúc đó là:

      Error: Shared daemon lock /run/lianli-daemon.lock is unavailable; refusing
             to start a competing hardware owner. Install the tmpfiles rule on
             the host and run `sudo systemd-tmpfiles --create lianli.conf`.

  Thấy đoạn này là nghĩa là thiếu `tmpfiles.d/lianli.conf`.

Nội dung 2 file copy **nguyên văn** từ
[upstream `sgtaziz/lian-li-linux`](https://github.com/sgtaziz/lian-li-linux)
(`packaging/{tmpfiles,sysusers}.d/lianli.conf`). Đã kiểm chứng trên máy thật:
daemon lên được ngay, 5/5 lần nạp lại rule đều áp đúng gid.

**Hai lệnh này không thừa.** `/run` là tmpfs nên lock mất sau mỗi lần reboot;
`systemd-tmpfiles-setup.service` sẽ tạo lại từ rule trong `/usr/lib/tmpfiles.d/`.
Đừng `touch` tay — touch không sống sót qua reboot, còn rule thì có.

**Hai lệnh phải đúng thứ tự.** `systemd-sysusers` tạo group, `systemd-tmpfiles` tạo
lock. Ngược lại là `usermod: group 'lianli' does not exist`. Và `sysusers` phải
chạy **trước** phần `udevadm` ở trên — xem mục kế.

#### Node hidraw vẫn `root:root`

Sau khi tạo group, `/dev/hidrawN` của Lian Li phải ra:

```bash
$ ls -l /dev/hidraw*    # dòng 0cf2:a100
crw-rw----+ 1 root lianli 240, 10 ... /dev/hidraw10
```

Nếu vẫn `crw-------  root root` (group `root`) thì token `GROUP="lianli"` đã bị
bỏ qua. Lý do: udev nạp rules **một lần lúc start**, và group phải tồn tại
trước lúc nạp. Nạp lúc group chưa có → không resolve được gid → bỏ qua **im lặng**,
không có cảnh báo nào trong journal.

Dấu hiệu nhận biết là `ls -l` hiện `crw-rw----+` cho group `root`, tức `0660`
đúng mà group sai — mode đó chỉ là hiệu ứng của ACL mask do `uaccess` builtin
(`73-seat-late.rules`) đặt, **không phải** `MODE=` của rule có hiệu lực.

Cách kiểm chứng:

```bash
journalctl --user -b -o cat | grep 'lianli.rules.*Set group'
# có dòng  ->  hidraw10: .../60-lianli.rules:100 GROUP="lianli": Set group ID: 957
# trống    ->  rule chưa từng khớp
```

Cách sửa: tạo group → `udevadm control --reload-rules` → trigger lại. Đã thử 5
lần liên tiếp, cả có lẫn không có `sleep` giữa hai lệnh, đều ra kết quả đúng.

`usermod` chỉ ghi vào `/etc/group`, nên **thoát hẳn rồi đăng nhập lại** thì nhóm
mới có hiệu lực. Reboot ở bước 2 không giúp được, vì nó nằm trước `usermod`.

Kiểm tra sau khi login lại:

```bash
id -nG | grep -E '^(i2c|lianli)$'    # phải thấy cả hai
```

Chưa thấy thì `systemd --user` hiện tại vẫn mang group cũ — service vẫn chạy
được nhờ ACL `uaccess`, chỉ là chưa dùng được đường group "không cần login"
mà thiết kế nhắm tới.

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
```

`openrgb` cần ~10s để dò xong 4 controller (wrapper poll mỗi 3s), nên `openrgb -l`
chạy ngay sẽ **trả về rỗng** — đó không phải lỗi. Theo dõi tới khi nó báo OK:

```bash
journalctl --user -t openrgb-wrapper -f     # chờ dòng "OK: 4 controllers"
openrgb -l                                  # sau đó phải thấy đủ 4 mục
```

Lệnh nào cũng phải có `--user`. Máy này có **hai** unit cùng tên
`openrgb.service`: gói `openrgb` đặt một bản ở `/usr/lib/systemd/system/`
(system, `disabled`), còn repo đặt bản user ở `~/.config/systemd/user/`
(`enabled`). `systemctl status openrgb` không có `--user` sẽ ra bản system —
đang `disabled`/`inactive`, trông như service chết trong khi bản đang chạy vẫn
ổn:

```bash
systemctl is-active openrgb.service          # inactive — ĐÚNG, đây là bản system
systemctl --user is-active openrgb.service   # active   — bản này mới là bản đang chạy
```

`~/.config/systemd/user/` đè lên bản của gói ở cả hai tên unit, nên
`lianli-daemon.service` trong repo phải là bản sao **đầy đủ** chứ không phải
drop-in — xem [Ghi chú](#ghi-chú).

Dùng phần cứng khác máy tham chiếu thì để daemon tự sinh `config.json` rồi chỉnh
theo tài liệu upstream. Lần chạy đầu daemon sẽ tự sửa device ID về ID thật và
báo `Migrating wired device identities`.

### 6. Xác minh

Chạy hết 9 lệnh này. Lệnh nào không khớp với cột "kỳ vọng" thì cài đặt chưa xong.

| # | Lệnh | Kỳ vọng |
|---|------|---------|
| 1 | `getent group lianli` | `lianli:x:957:<bạn>` |
| 2 | `ls -l /run/lianli-*.lock` | 2 file, `root root`, `rw-rw-rw-` |
| 3 | `systemctl --user is-active openrgb.service lianli-daemon.service` | `active` ×2 |
| 4 | `openrgb -l` | đủ 4 mục |
| 5 | `journalctl --user -t openrgb-wrapper -n 20 --no-pager \| grep 'OK:'` | có dòng `OK: 4 controllers` |
| 6 | `journalctl --user -u lianli-daemon -n 20 --no-pager \| grep 'shared daemon lock'` | `Acquired shared daemon lock` |
| 7 | `ls -l /dev/hidraw1 /dev/hidraw10` | `root:i2c` rồi `root:lianli` |
| 8 | `id -nG \| grep -E '^(i2c\|lianli)$'` | thấy cả hai — **chỉ sau khi login lại** |
| 9 | vòng `for` bên dưới | 7 dòng `OK`, không `FAIL` |

```bash
getent group lianli
ls -l /run/lianli-daemon.lock /run/lianli-control.lock
systemctl --user is-active openrgb.service lianli-daemon.service
openrgb -l
journalctl --user -t openrgb-wrapper -n 20 --no-pager | grep 'OK:'
journalctl --user -u lianli-daemon -n 20 --no-pager | grep 'shared daemon lock'
ls -l /dev/hidraw1 /dev/hidraw10
id -nG | grep -E '^(i2c|lianli)$'
for s in white rainbow breath red xanhtim vangxanh daquang2; do
  apply-rgb "$s" && echo "$s OK" || echo "$s FAIL"
done
```

Kết quả đo thật trên máy tham chiếu:

```
lianli:x:957:frost-auslese
-rw-rw-rw- 1 root root 0 ... /run/lianli-control.lock
-rw-rw-rw- 1 root root 6 ... /run/lianli-daemon.lock
active
active
0: Kingston Fury DDR5 DRAM
1: Sapphire Radeon RX 7800 XT Nitro+
2: Lian Li Uni Hub - SL
3: ASUS ROG STRIX Z690-A GAMING WIFI
openrgb-wrapper: OK: 4 controllers (Kingston present); applied scheme 'white'
lianli_daemon::pidlock: Acquired shared daemon lock at /run/lianli-daemon.lock
crw-rw----+ 1 root i2c    240,  1 ... /dev/hidraw1
crw-rw----+ 1 root lianli 240, 10 ... /dev/hidraw10
i2c
lianli
white OK  rainbow OK  breath OK  red OK
xanhtim OK  vangxanh OK  daquang2 OK
```

Lệnh 3 in đúng hai dòng chữ `active` — `systemctl is-active` không kèm tên unit.
Dòng `bỏ qua [2] Lian Li Uni Hub - SL` xen giữa các scheme là **đúng**, xem
[Lian Li hub](#lian-li-hub).

Lệnh 8 là lệnh duy nhất **phải đăng nhập lại** mới đúng: `usermod` chỉ ghi
`/etc/group`, còn `systemd --user` đang chạy thì giữ group cũ. Trước khi login
lại, service vẫn chạy được nhờ ACL `uaccess` — xem
[Node hidraw vẫn root:root](#node-hidraw-vẫn-rootroot).

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

### `#496DFF` trên fan SL120 v1 — cân bằng lỗi phần cứng

Màu LED của hub **không** đổi bằng `apply-rgb`. Nó nằm trong
`~/.config/lianli/config.json`, sửa tay hoặc qua GUI của `lianli-daemon`.

Nhìn kỹ thì thấy một con số đáng ngờ:

| Zone | Mode | Màu trong config | Nhìn ra |
|------|------|------------------|---------|
| `group0` zone 0-2 | `Runway` | `#496DFF` | trắng |
| `group1` zone 1-2 | `Tide` | `#496DFF` | trắng |
| `group2` / `group3` zone 0-2 | `Static` | `#FFFFFF` | trắng |
| `group1` zone 0 | `Staggered` | `#FFDDDD` | hồng nhạt |

`#496DFF` **không phải màu xanh** — nó là màu trắng *bù lỗi quạt*. Trên fan
Lian Li SL120 v1 của máy này, **bóng LED kênh xanh (blue) bị yếu hơn hẳn kênh
đỏ và xanh lá**. Đặt `#496DFF` = R73 G109 B255 sẽ bù lại: ba kênh lệch nhau đúng
lượng, mắt thấy thành trắng. Nếu đặt `#FFFFFF` trên những zone đó thì xanh
chiếm ít hơn, ra **tím nhạt/xám xanh** chứ không phải trắng.

Đây là lỗi phần cứng của quạt, không phải cấu hình sai. Nó chỉ ảnh hưởng cảm
nhận màu — fan vẫn điều khiển tốc độ bình thường.

Hệ quả thực tế:

- **Đổi màu qua `apply-rgb` không đụng tới phần này** — đó là chủ ý, xem
  [Lian Li hub](#lian-li-hub).
- **Đừng "sửa cho đẹp" `#496DFF` thành `#FFFFFF`** trong `group0`/`group1`, sẽ
  thấy màu lệch. Muốn tự hiệu chỉnh thì tăng dần R/G cho tới khi mắt thấy cân
  bằng, không có công thức cố định vì độ yếu phụ thuộc từng bóng.
- **Đừng áp cùng một giá trị cho mọi group.** `group2`/`group3` đang dùng
  `#FFFFFF` thật và nhìn ra trắng bình thường — cứ để nguyên. Chỉ hiệu chỉnh
  những zone mà bạn tự thấy màu lệch, từng zone một.
- **`brightness` và `speed` trong file này là thang 0–4**, khác hẳn thang 0–100
  của `BRIGHTNESS`/`SPEED` trong scheme `.rgb`. Daemon từ chối giá trị ngoài
  `0..=4` (`RGB brightness must be 0..=4`). Config hiện đặt `brightness: 4` —
  mức cao nhất — và `speed: 0` cho các zone tĩnh.

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
| `Failed to read '/usr/lib/tmpfiles.d/lianli.conf'` | gói AUR không ship file này | [Bước 4 → Vì sao phải tự cài 2 file rule](#vì-sao-phải-tự-cài-2-file-rule) |
| `usermod: group 'lianli' does not exist` | chưa có group `lianli` | `sudo systemd-sysusers /usr/lib/sysusers.d/lianli.conf` rồi chạy lại `usermod` |
| `usermod: group 'i2c' does not exist` | bỏ qua Bước 1, hoặc chưa nạp sysusers | `sudo pacman -S i2c-tools && sudo systemd-sysusers` rồi chạy lại `usermod` |
| `Shared daemon lock /run/lianli-daemon.lock is unavailable` | thiếu tmpfiles rule → daemon restart loop | `sudo systemd-tmpfiles --create /usr/lib/tmpfiles.d/lianli.conf` rồi `systemctl --user restart lianli-daemon` |
| `/dev/hidraw10` của Lian Li là `root:root` | udev nạp rule trước khi có group | [Bước 4 → Node hidraw vẫn root:root](#node-hidraw-vẫn-rootroot) |
| `openrgb -l` không thấy Kingston | `spd5118` còn giữ bus SPD | [Bước 2](#2-nhường-bus-spd-cho-ram-ddr5) |
| `! openrgb failed for [2] zone=all` rồi restart lặp lại | `apply-rgb` set `static` lên hub (abort) | [Lian Li hub](#lian-li-hub) |
| `systemctl status openrgb` báo inactive | đang xem bản system của gói | thêm `--user`, xem [Bước 5](#5-bật-service) |
| `openrgb -l` rỗng ngay sau `enable --now` | wrapper chưa dò xong (~10s) | chờ, xem [Bước 5](#5-bật-service) |
| `no devices listed by openrgb -l` | service chưa lên | `systemctl --user status openrgb` |
| `Permission denied` lúc service khởi động | thiếu `chmod +x` cho wrapper | chạy lại dòng `chmod +x` ở [Bước 4](#4-copy-file-và-cấp-quyền) |

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

`evdi` **không liên quan gì** tới RGB hay fan. Nó tạo màn hình ảo để đẩy desktop
lên **LCD của hub** — mà Uni Hub SL không có LCD, và `config.json` để `lcds: []`.
Tài liệu upstream nói thẳng: *"Ordinary fan/RGB control does not require an EVDI
kernel module."* Máy chạy dwm/X11 cũng không dùng tới.

`lianli-linux-git` kéo `evdi-dkms` làm phụ thuộc nên không tránh được. Nó **không
ảnh hưởng** tới hai service này kể cả khi hỏng — cứ để nguyên. Trên máy tham
chiếu nó build và load thành công:

```bash
$ dkms status | grep evdi
evdi/1.15.1, 6.18.52-1-cachyos-lts, x86_64: installed
evdi/1.15.1, 7.2.8-1-cachyos, x86_64: installed
evdi/1.15.1, 7.2.8-lqx1-1-arisa, x86_64: installed     ← kernel đang chạy

$ lsmod | grep -c '^evdi'
1
```

Nếu kernel của bạn **chưa** có dòng khớp `uname -r` (thường là do lúc cài gói
chưa có headers), ô Health sẽ báo `Failed` thay vì `N/A`. Khắc phục:

```bash
sudo pacman -S linux-arisa-headers   # đổi theo kernel đang chạy
sudo modprobe evdi
dkms status | grep evdi              # phải có dòng khớp uname -r
```

Cài headers là hook `70-dkms-install.hook` tự gọi `dkms install`, nên không cần
gọi tay. Hook không `modprobe` — vẫn phải modprobe hoặc reboot. Tên gói headers
phải theo đúng kernel: `linux-cachyos-headers` cho `linux-cachyos`,
`linux-arisa-headers` cho `linux-arisa`, v.v.

### Ghi chú

- **Máy không có display manager** (`systemctl get-default` → `graphical.target`,
  không có `display-manager.service`), chạy dwm trên X11. Nhờ vậy service không
  phụ thuộc phiên đồ hoạ. Đây cũng là lý do `lianli-daemon.service` trong repo
  phải là **bản sao đầy đủ**, không phải drop-in: package unit có
  `After=` + `PartOf=graphical-session.target`, mà systemd không cho drop-in xoá
  `PartOf=` — nên daemon sẽ bị stop mỗi lần logout nếu chỉ override một phần.
  Bản trong `~/.config/systemd/user/` thắng bản của gói, nên nhớ đồng bộ lại từ
  `/usr/lib/systemd/user/lianli-daemon.service` sau khi nâng cấp gói.
- **`rgb.openrgb_port`** không được dùng khi `rgb.openrgb_server: false` — lúc đó
  daemon tự điều khiển hub, không qua OpenRGB server. Chỉ đổi khi bật
  `openrgb_server: true`.

---

## 🔁 Đưa thay đổi trên máy về repo

```bash
cp ~/.local/bin/apply-rgb           ~/rgb-config/local/bin/
cp ~/.local/lib/*-wrapper.sh        ~/rgb-config/local/lib/
cp ~/.config/openrgb/schemes/*.rgb  ~/rgb-config/schemes/
cp ~/.config/lianli/*.json          ~/rgb-config/lianli/
cp ~/.config/systemd/user/openrgb.service          ~/rgb-config/systemd/
cp ~/.config/systemd/user/lianli-daemon.service     ~/rgb-config/systemd/
cp ~/.config/systemd/user/lianli-daemon.service.d/retry-open-acl.conf \
   ~/rgb-config/systemd/lianli-daemon.service.d/

cd ~/rgb-config && git commit -am "update: ..." && git push
```

`packaging/` không cần đồng bộ — 2 file đó copy nguyên văn từ upstream, không
đổi theo máy.
