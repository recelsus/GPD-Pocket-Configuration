# Packages

```
sudo
base-devel
iwd

vim
neovim
git
rustup

xorg-init
xorg
i3
dmenu

noto-fonts-cjk
ttf-noto-nerd

fcitx5
fcitx5-configtool
fcitx5-mozc
fcitx5-qt
fcitx5-gtk

ghostty
chromium

wget
curl
unzip
zip
rsync
```

# AUR

`paru`

```bash
rustup default stable
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```

# Config

`sudo vim /etc/modprobe.d/i915.conf`

```
options i915 enable_psr=0 enable_fbc=0 enable_dc=0
```

`sudo mkinit cpio -P`


`sudo vim /boot/loader/entries/yyyy-MM-dd_HH-mm-ss_linux.conf
`

```
options root=PARTUUID=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee rw rootfstype=ext4 acpi_backlight=vendor
```

`sudo vim /etc/systemd/logind.conf
`

```
HandleSuspendKey=ignore
HandleLidSwitch=ignore
IdleAction=ignore
```

`vim $HOME/.xinitrc`

```
primary_out="$(xrandr --query | awk '/ connected primary/{print $1; exit}')"
if [ -z "${primary_out:-}" ]; then
  primary_out="$(xrandr --query | awk '/ connected/{print $1; exit}')"
fi

scale_xy="0.7x0.7"

xrandr --output "$primary_out" \
       --rotate right \
       --scale "${scale_xy}" \
       --filter nearest \
       || true
       # --panning 1920x1200+0+0 || true

xset s off
xset -dpms
xset s noblank

export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx

fcitx5 &

exec i3
```

```bash
sudo systemctl enable iwd
sudo systemctl start iwd

sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd

sudo systemctl enable systemd-resolved
sudo systemctl start systemd-resolved
```

# Battery Charge

```
sudo install -d /usr/local/sbin
sudo tee /usr/local/sbin/gpd_charge_fix >/dev/null <<'EOF'
#!/bin/sh
# GPD Pocket: restore bq24190 input current limit

CHG=/sys/class/power_supply/bq24190-charger

if [ -r "$CHG/online" ] && [ "$(cat "$CHG/online")" = "1" ]; then
  echo 2000000 > "$CHG/input_current_limit" # 2A
fi
EOF
sudo chmod +x /usr/local/sbin/gpd_charge_fix

```

```
sudo tee /etc/udev/rules.d/90-gpd-bq24190.rules >/dev/null <<'EOF'
SUBSYSTEM=="power_supply", KERNEL=="bq24190-charger", ACTION=="change", RUN+="/usr/local/sbin/gpd_charge_fix"
EOF
```

```
sudo udevadm control --reload-rules
sudo udevadm trigger -s power_supply
```

