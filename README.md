# Arch Linux on GPD Pocket(Z8750)

## Description
GPD Pocket初代にArchLinuxをインストールしたときの設定と端末固有問題のメモ

## Packages

必須パッケージ群

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

## AUR

AURのパッケージマネージャー`paru`のビルド

```bash
rustup default stable
git clone https://aur.archlinux.org/paru.git
cd paru
makepkg -si
```
## Service

パッケージインストール後 サービスだけ開始しておきます
systemd-networkdとsystemd-resolvedは既に実行されているかもしれませんが一応

```bash
sudo systemctl enable iwd
sudo systemctl start iwd

sudo systemctl enable systemd-networkd
sudo systemctl start systemd-networkd

sudo systemctl enable systemd-resolved
sudo systemctl start systemd-resolved
```

## Network

```bash
iwctl
```

```
[iwd]# device list
[iwd]# station wlan0 scan
[iwd]# station wlan0 get-networks
[iwd]# station wlan0 connect SSID_NAME
```

接続成功すると`/var/lib/iwd/SSID_NAME.psk`に設定が保存されます.


## Config

#### Intel i915 GPUドライバの設定  
`sudo vim /etc/modprobe.d/i915.conf`

```
options i915 enable_psr=0 enable_fbc=0 enable_dc=0
```

- enable_psr=0
  - Panel Self Refreshの無効化
  - eDP/DSIパネルの省電力機能, 復帰不能のブラックアウト対策

- enable_fbc=0
  - Frame Buffer Compressionの無効化
  - フレームバッファの圧縮, xrandrのrotale/scaleと相性が悪いようなのでこれも停止

- enable_dc=0
  - Display C-statesの無効化
  - DPMSがONでも画面が戻らない問題回避

`sudo mkinitcpio -P`

#### 

`sudo vim /boot/loader/entries/yyyy-MM-dd_HH-mm-ss_linux.conf
`

```
options root=PARTUUID=aaaaaaaa-bbbb-cccc-dddd-eeeeeeeeeeee rw rootfstype=ext4 acpi_backlight=vendor
```

- acpi_backlight=vendor
  - ACPIの汎用バックライト制御を無効化します
  - acpi_video[i]を見つけてしまうので明示的に無視させます
  - 使えない を 使わない と明示しているだけなのでこの項目に関してはやらなくても問題ナシ

`sudo vim /etc/systemd/logind.conf
`

```
HandleSuspendKey=ignore
HandleLidSwitch=ignore
IdleAction=ignore
```

- HandleSuspendKey
  - 物理キーでのサスペンドを無効化
- HandleLidSwitch=ignore
  - 画面を閉じたときのサスペンドを無効化
- IdleAction=ignore
  - 放置したときの自動サスペンドを無効化

総じて全てサスペンド(スリープ)の無効化です  
スリープ復帰時に画面表示が戻ってこない症状が起きるため

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

xset s off
xset -dpms
xset s noblank

setxkbmap -option caps:none

export GTK_IM_MODULE=fcitx
export QT_IM_MODULE=fcitx
export XMODIFIERS=@im=fcitx

fcitx5 &

exec i3
```


# Battery Charge

給電ケーブル接続時に十分な給電が行われず 接続中でもバッテリーが減る問題の対処  
電源OFF時と接続のままOS起動時には9Wを維持していましたが, 一度抜いて再度挿すと2W(0.5A)まで落ちていることから電源制御関連の修正  

`/sys/class/power_supply/bq24190-charger`の値を変更すると給電量が変化したことを確認できたのでudevでこれを制御させています  
あと最大の9Wではなく4Wが安定して良さそうだったので2Aに設定 充電は遅いですが刺したままなら減らないライン

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

