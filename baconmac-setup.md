# baconmac (MacBookAir4,1) Linux 化 セットアップ記録

**最終更新**: 2026-05-11 (F1/F2/F5/F6 ハードキー対応 / brightnessctl + UPower KbdBacklight / panel に power-manager-plugin 追加)
**機種**: Apple MacBook Air (Mid 2011, MacBookAir4,1)
**スペック**: i7-2677M (Sandy Bridge) / Intel HD3000 / BCM43224 WiFi / 3.7GiB / SanDisk SD9TN8W M.2 SSD (変換アダプタ)
**OS**: Ubuntu 26.04 LTS Server (kernel 7.0.0-14-generic) / XFCE4

---

## 結論: 安定化に必要だった対策一覧

| 問題 | 原因 | 対策 |
|---|---|---|
| ランダム hang (suspend) | resume 失敗 | sleep 系 target 全 mask + logind ignore |
| ランダム hang (ASPM) | Sandy Bridge PM bug | GRUB cmdline: `pcie_aspm=force pcie_aspm.policy=performance` |
| SSD hang (1分毎 cycle) | DIPM resume failure (SanDisk + Apple AHCI) | GRUB: `libata.force=nolpm` |
| 電源 hang (AC flap) | ACPI GPE17 storm (USB-C→MagSafe ケーブル) | AC driver unbind systemd unit + GPE17 mask |
| **WiFi (BCM43224) hang** | **PCI MSI 互換性** | **GRUB: `pci=nomsi`** (MacBookAir4,1 既知 workaround) |
| CPU 2.6GHz 張り付き (旧記載、現状は解消) | kanata/fcitx5 input 待機が原因と推定していた | 2026-05-23 powertop 15分計測で **idle 中 800MHz 76.9% / 平均 920-1000MHz / Package C7 86.8% / CPU C7 98.0-98.5%** を実測。kernel 7.0.0-14 + schedutil で再現せず |
| fan 2000RPM 固定 | applesmc bug Ubuntu #461184 | mbpfan で coretemp ベース自動制御 |
| 充電キレ (HW) | サードパーティ USB-C→MagSafe ケーブル | SW 修復不可、純正 Apple アダプタ交換が根治 |
| 蓋閉じても LCD/リンゴ/KBD LED 点灯 | logind ignore + xfce4-pm lid-action=0 + SW_LID 不安定 + Apple ACPI _BCM 不動作 | GRUB に `acpi_backlight=native` 追加 → `intel_backlight` 露出、polling daemon (`baconmac-lid-poll.service`) で `/proc/acpi/button/lid/LID0/state` を 1秒間隔監視し intel_backlight + smc::kbd_backlight を退避/復元 |
| F1/F2 で明るさ調整できない | `intel_backlight` (type=raw) は xrandr Backlight プロパティ未公開 + `0644 root:root` で user 書込不可 → xfce4-power-manager が key を受けても書く先が無い | `brightnessctl` 導入 (udev rule で `video` group g+w 付与) + `bacon` を video group へ + xfce keyboard shortcut で `XF86MonBrightnessUp/Down` → `brightnessctl set ±5%`、xfce4-power-manager 側は `handle-brightness-keys=false` で無効化 |
| F5/F6 でキーボードバックライト調整できない | `smc::kbd_backlight` 書込は `input` group 必要 (キーロガ権限の懸念) | UPower D-Bus (`org.freedesktop.UPower.KbdBacklight.SetBrightness`) 経由なら group 不要 → wrapper `/usr/local/bin/baconmac-kbd-backlight` + xfce shortcut で `XF86KbdBrightnessUp/Down` にバインド |
| 明るさ調整 UI が無い | systray の battery アイコンには輝度スライダー無し | `xfce4-power-manager-plugins` 導入 + 専用 panel plugin (`power-manager-plugin`) を `/panels/panel-1/plugin-ids` に追加 (12 番、systray と separator の間に挿入)。クリックで battery 情報 + 輝度スライダー |

## 重要な失敗 (やってはいけない)

- `ahci.mobile_lpm_policy=1` cmdline: Apple AHCI は `AHCI_HFLAG_IS_MOBILE` フラグ無いので無視される → `libata.force=nolpm` が確実
- `acpi_mask_gpe=0x17` cmdline: boot 早期に SBS HC enumeration 阻害、battery 消失 → AC driver unbind systemd unit が正解
- TLP 標準設定: USB_AUTOSUSPEND=1 で USB Ethernet 切断、SATA_LINKPWR で SSD hang → TLP 完全停止 (mask) 推奨
- b43 driver で BCM43224: brcmsmac と同様 hang する → pci=nomsi で brcmsmac の方が安定
- `intel_idle.max_cstate=1` 残置: 真の hang 主因は brcmsmac、削除で battery + heat 改善
- `acpi_video0/bl_power=4` で LCD off: Apple ACPI `_PSx`/`_BCM` は no-op で sysfs 値だけ変わり HW は変化なし → `acpi_backlight=native` で `intel_backlight` を露出させ HW PWM 直接制御
- `acpi_video0/brightness=0` 直書き: 戻りを `brightness=15` で書いても復帰しない (Apple ファーム既知バグ)、再起動以外回復不可 → `intel_backlight` でも安全側で `brightness=1` (最小非ゼロ) を退避値として使用
- `xset dpms force off` だけ: 画面 blank のみでバックライト OFF にはならない、加えて任意 input event で auto-wake → 蓋閉じ中の途中復帰の元凶
- acpid + lid event handler: SW_LID input event がカーネル warning (`lid device is not compliant to SW_LID`) で死ぬ瞬間があり信頼できない → `/proc/acpi/button/lid/LID0/state` の polling が確実
- `xfce4-power-manager` 単独で brightness key 処理を任せる: `acpi_backlight=native` で `intel_backlight` (type=raw) のみになると **xrandr Backlight プロパティが未公開** + sysfs は user 書込不可 → power-manager は key を受けても何もできない (silent fail)。`brightnessctl` のような専用 helper を keybinding に直接バインドする方が確実
- ユーザーを `video` group に追加せず brightnessctl だけ入れる: udev rule (`/lib/udev/rules.d/...brightness*`) で sysfs を `g+w` するだけなので **group 所属が無いと書込不可**。`usermod -aG video bacon` 必須 + 再ログイン
- `xfce4-power-manager` の `brightness-switch-restore-on-exit=1` 放置: セッション終了時の値を次回復元するため、低い値で終了したセッションがあると以後ずっと暗く起動 (Apple ロゴも暗くなる) → `0` にして kernel 初期値 (起動時の HW デフォルト) で開始させる

## 現在の構成 (2026-05-07 動作確認済)

### GRUB cmdline (`/etc/default/grub`)

```
GRUB_CMDLINE_LINUX_DEFAULT="pcie_aspm=force pcie_aspm.policy=performance libata.force=nolpm pci=nomsi acpi_backlight=native"
```

`acpi_backlight=native` で `acpi_video0` を抑制し `intel_backlight` (i915 直接 PWM、type=raw、max_brightness=1808) を露出させる。Apple ファーム ACPI _BCM 不動作の根本回避。

### systemd 設定

**masked (停止状態維持):**
- `sleep.target / suspend.target / hibernate.target / hybrid-sleep.target`
- `tlp.service` (互換性問題)

**enabled + active:**
- `baconmac-mask-gpe17.service` (AC unbind と冗長だが残置)
- `baconmac-ac-unbind.service` (boot 後 ACPI AC driver detach)
- `baconmac-lid-poll.service` (蓋閉じ時に intel_backlight + smc::kbd_backlight を退避/復元する polling daemon、`/usr/local/bin/baconmac-lid-poll.sh`)
- `mbpfan.service` (fan 自動制御 / `/etc/mbpfan.conf` を `low_temp=55 / high_temp=60 / max_temp=80` にチューニング、デフォルト 63/66/86 から引き下げ)
- `kanata.service` (キーボード remap)
- `NetworkManager.service` (WiFi + 有線管理)
- `systemd-zram-setup@zram0.service` (zram swap)
- `acpid.service` (現状ハンドラなし、将来用に残置)

**追加パッケージ:**
- `brightnessctl` (apt) — 同梱 udev rule で `/sys/class/backlight/*/brightness` を `video` group g+w、`/sys/class/leds/*/brightness` を `input` group g+w に変更
- `xfce4-power-manager-plugins` (apt) — `power-manager-plugin` パネル item (battery + 輝度スライダー) を提供
- `libfuse2t64` (apt, 2026-05-17 追加) — 旧来型 AppImage (`libfuse.so.2` 要求) を `--appimage-extract-and-run` 無しで直接実行可能にする。Ubuntu 24.04 t64 transition で `libfuse2` から改名。`fuse3` 系 (`libfuse3.so.4`) は既に入っているが AppImage の多くは依然 fuse2 要求
- `appimagelauncher` v3.0.0-beta-3 (TheAssassin/AppImageLauncher GitHub release deb, 2026-05-17 追加) — AppImage ダブルクリック時に「Integrate and run?」ダイアログ → `~/Applications` に移動 + `.desktop` 生成 + アイコン抽出を自動化。仕組みは `binfmt_misc` 経由で `/opt/appimagelauncher.AppDir/.../binfmt-interpreter` を AppImage の interpreter として登録 (`/proc/sys/fs/binfmt_misc/appimage-type1`, `appimage-type2`)。`appimagelauncherd.service` (user systemd) で daemon 常駐。**apt には無いので deb 取得が必要** (`gh release download v3.0.0-beta-3 -R TheAssassin/AppImageLauncher --pattern '*amd64.deb'` → `sudo apt install -y /tmp/appimagelauncher_*.deb` → `systemctl --user enable --now appimagelauncherd.service`)

**ユーザー groups (`bacon`):**
- `video` 追加済 (2026-05-11) — `brightnessctl` で `intel_backlight` を user 権限書込可
- `input` 追加しない — キーボード LED は UPower D-Bus 経由 (`/usr/local/bin/baconmac-kbd-backlight`) で制御し group 不要

**`/usr/local/bin/baconmac-kbd-backlight`** (キーボード LED 制御 wrapper):
- 現在値を `/sys/class/leds/smc::kbd_backlight/brightness` から読取 (read は誰でも可)
- `busctl call org.freedesktop.UPower /org/freedesktop/UPower/KbdBacklight ... SetBrightness i N` で書込 (polkit でアクティブセッションに許可)
- STEP=26 (~10% of max 255)

**`/panels/panel-1/plugin-ids`** (2026-05-11 更新): `[1,2,3,4,5,6,12,7,11,8,9,10]`
- `plugin-12 = power-manager-plugin` (右クリで battery + 輝度スライダー、F1/F2 で動く intel_backlight に連動)

### `/etc/systemd/logind.conf`

```
HandlePowerKey=ignore
HandleLidSwitch=ignore
HandleLidSwitchExternalPower=ignore
HandleLidSwitchDocked=ignore
```

### `/etc/modprobe.d/`

- `blacklist-bluetooth.conf`: btusb, bluetooth, btbcm, btintel, btrtl

注: brcmsmac/b43 blacklist は **削除済** (pci=nomsi で WiFi 安定動作)。

### `/etc/sysctl.d/`

- `99-baconmac-debug.conf`: `kernel.nmi_watchdog=1`
- `99-zram-swappiness.conf`: `vm.swappiness=100`, `vm.vfs_cache_pressure=50`

### `/etc/systemd/zram-generator.conf`

```ini
[zram0]
zram-size = ram / 2
compression-algorithm = lz4
swap-priority = 100
```

### kanata config (`/etc/kanata/baconmac.kbd`)

```lisp
;; v9: Cmd+Arrow = Windows風スプリット、Cmd+Ctrl+V = クリップボード履歴
(defcfg
  process-unmapped-keys yes
  concurrent-tap-hold yes)

(defsrc
  caps lmet rmet spc left right up down v)

(defalias
  lcmd-mod (tap-hold 200 200 lmet (layer-while-held modlayer))
  rcmd-im (tap-hold 200 200 henk rmet))

(deflayer base
  lctl @lcmd-mod @rcmd-im spc left right up down v)

(deflayer modlayer
  _ _ bspc ret M-left M-right M-up M-down M-v)
```

**動作:**
- CapsLock → Ctrl
- 左Cmd タップ → Super_L (XFCE menu 等)
- 左Cmd ホールド + Space → Enter
- 左Cmd ホールド + 右Cmd → BackSpace
- 左Cmd ホールド + ← / → → Super+←/→ (xfwm4 で左/右半分タイル)
- 左Cmd ホールド + ↑ → Super+↑ (xfwm4 で **最大化トグル** に再バインド済)
- 左Cmd ホールド + ↓ → Super+↓ (xfwm4 で **最小化** に再バインド済)
- 左Cmd ホールド + V → Super+V (Ctrl と合わせ Cmd+Ctrl+V でクリップボード履歴 popup)
- 右Cmd タップ → Henkan (IME 切替)
- 右Cmd ホールド → Super_R

バージョン履歴: v6 までは Cmd+C/V → Super+C/V (xfce4-terminal コピペ用)、v7 で C/V 削除 (GNOME Terminal 移行)、v8 で Arrow 追加 (Windows 風スプリット)、v9 で V を Super+V として再追加 (clipman popup 用)。

### fcitx5 設定 (`~/.config/fcitx5/`)

`config`:
```ini
[Hotkey]
EnumerateForwardKeys=Ctrl+space
EnumerateBackwardKeys=Ctrl+Shift+space

[Hotkey/TriggerKeys]
0=Control+space
1=Zenkaku_Hankaku
2=Hangul
3=Henkan_Mode
```

**重要罠**: TriggerKeys は **必ず `[Hotkey/TriggerKeys]` 別 section + numbered entries**。`[Hotkey] TriggerKeys=...` (1 行 space 区切り) は **無効構文**。

### `/etc/environment` (IM 環境変数)

```
GTK_IM_MODULE=fcitx
QT_IM_MODULE=fcitx
XMODIFIERS=@im=fcitx
SDL_IM_MODULE=fcitx
```

注: 環境変数は **logout → login 後** にしか反映しない。

### Locale + Timezone

- `/etc/default/locale`: `LANG=ja_JP.UTF-8`
- timezone: `Asia/Tokyo` (`timedatectl set-timezone Asia/Tokyo`)

### XFCE Theme

- GTK theme: `Arc-Dark` (`xfconf-query -c xsettings -p /Net/ThemeName`)
- xfwm4 theme: `Arc-Dark` (`xfconf-query -c xfwm4 -p /general/theme`)
- Icon theme: `elementary-xfce-dark`

### Panel plugins

`/panels/panel-1/plugin-ids = [1,2,3,4,5,6,7,11,8,9,10]`:
- 1=applicationsmenu, 2=tasklist, 3=separator, 4=pager, 5=separator
- 6=systray (nm-applet, xfce4-power-manager 表示)
- 7=separator
- **11=pulseaudio (音量制御 + 出力切替)** ← 2026-05-07 追加
- 8=clock, 9=separator, 10=actions

### NetworkManager (`/etc/netplan/00-installer-config.yaml`)

```yaml
network:
  version: 2
  renderer: NetworkManager
  wifis:
    wlp2s0b1:
      access-points:
        link-wifi:
          password: "..."
      dhcp4: true
```

WiFi 動作中 (brcmsmac driver、pci=nomsi で安定)。

## 起動時の検証チェックリスト

```bash
# cmdline 確認
cat /proc/cmdline
# → pcie_aspm=force pcie_aspm.policy=performance libata.force=nolpm pci=nomsi

# SATA LPM = max_performance
cat /sys/class/scsi_host/host0/link_power_management_policy

# WiFi (brcmsmac) 動作中
lsmod | grep brcmsmac
nmcli device status

# 重要 service active
systemctl is-active mbpfan kanata baconmac-mask-gpe17 baconmac-ac-unbind

# ADP1 消失 (AC unbind 効果)、BAT0 残る
ls /sys/class/power_supply/
# → BAT0 のみ (ADP1 消失が正常)

# zram active
zramctl
swapon --show

# Sleep 全 mask
for t in sleep suspend hibernate hybrid-sleep; do
  echo "$t: $(systemctl is-enabled $t.target)"
done

# fcitx5 + kanata 動作中
pgrep -af fcitx5
pgrep -af kanata
```

## ハング再発時の最初に見るべきもの

1. **journal 末尾**: `journalctl -k -b -1 | tail -50`
2. **SATA cycle**: `journalctl -k -b -1 | grep -E 'sda.*Stop|sda.*Start'`
3. **brcmsmac error**: `dmesg | grep -iE 'brcmsmac.*err|wlp2s0b1.*err'`
4. **温度**: `cat /sys/class/hwmon/hwmon1/temp1_input` (coretemp Package、idle で 50°C 超えてたら mbpfan 確認)
5. **CPU C-state**: `cat /sys/devices/system/cpu/cpu0/cpuidle/state*/usage` (C7 が使われてるか)

## 残課題・将来検討

- **充電不安定 (HW)**: 純正 Apple 60W/85W MagSafe1 アダプタ交換が根治
  - SW で完全諦め確認済 (`/sys/class/power_supply/ADP1/online` は read-only、USB PD ネゴは HW レベル、Linux 介入不可)
- ~~**CPU stuck 2.6GHz**: kanata/fcitx5 由来、機能維持のため受容 (deep C-state で熱は緩和されている)~~ → **2026-05-23 解消確認**。powertop 15分計測 (idle、AC 接続) で 800MHz 76.9% / 平均 920-1000MHz / CPU C7 98% を観測。schedutil + 現行 kernel で問題なし。「2.6GHz 張り付き」は過去観測されたが現状は再現しない
- **AutoPowerOn (AC で勝手に起動)**: SMC 仕様、Linux applesmc は read のみ、macOS で `nvram AutoBoot=%00` 必要
- **トラックパッド3本指/4本指ジェスチャ**: 2026-05-12 に **fusuma** で対応 (詳細は後段 UX 改善セクション)

## UX 改善 (2026-05-07)

| 課題 | 原因 | 対策 |
|---|---|---|
| Shift→IME モード切替 | mozc デフォルト `shift_key_mode_switch=ASCII_INPUT_MODE` | `mozc_tool --mode=config_dialog` で「シフトキーでの入力切替」を OFF |
| Win 風タイル不能 | `tile_*_key` が `<Super>KP_*` (テンキー) のみ | xfconf で `<Super>Left/Right/Up/Down` に追加バインド (`xfwm4/custom/`) |
| バッテリーアイコン消失 | `elementary-xfce-dark` 実体が snap 内のみ、XDG パス外 | `apt install elementary-xfce-icon-theme` (deb版を `/usr/share/icons/elementary-xfce/` に展開) |
| Cmd+C/V でコピペ不能 (ターミナル) | xfce4-terminal は Ctrl+Shift+C/V がデフォルト、kanata の Cmd-tap も chord では Super 発火しない | kanata の `defsrc` に `c v` 追加、`modlayer` に `M-c M-v` 追加。`~/.config/xfce4/terminal/accels.scm` で Copy/Paste を `<Super>c`/`<Super>v` に再バインド |
| ファン追従が遅い体感 | mbpfan デフォルト閾値 `low=63 / high=66 / max=86` が控えめで、CPU 60°C 付近で最小回転 (2000 RPM) 固定 | `/etc/mbpfan.conf` を `low=55 / high=60 / max=80` に引き下げ。65°C で 2300 RPM 程度まで自動上昇することを実測確認 |

## 蓋閉じ時 LED 全消灯 (2026-05-08 追加)

蓋を閉じてもディスプレイ・背面リンゴロゴ・キーボードバックライトが点いたままだった (suspend 禁止運用の副作用)。最終的な構成:

- **GRUB cmdline**: `acpi_backlight=native` 追加で `intel_backlight` (i915 PWM、`type=raw`、`max_brightness=1808`) を露出
- **`/usr/local/bin/baconmac-lid-poll.sh`**: `/proc/acpi/button/lid/LID0/state` を 1秒間隔で polling
  - 蓋閉じ → `intel_backlight/brightness` を退避して `1` 書込 + `smc::kbd_backlight/brightness=0`
  - 蓋開け → 退避値で復元
- **`/etc/systemd/system/baconmac-lid-poll.service`**: 上記スクリプトを root で常駐 (`Restart=on-failure`、`Nice=10`)
- 既存の `HandleLidSwitch=ignore` は維持 (suspend 禁止)、xfce4-power-manager の lid-action は 0 のまま (lock 抑制 = 蓋開け即復帰)

**なぜ polling か**: SW_LID input event がカーネル warning (`lid device is not compliant to SW_LID`) で時々死ぬのと、`/proc/acpi/button/lid/LID0/state` (ACPI control method) は常に正しいため。

**なぜ `brightness=1` か**: `brightness=0` は acpi_video0 で復帰不能 (Apple ファーム既知バグ)。intel_backlight でも安全側に倒して最小非ゼロ。視覚的には完全 OFF と区別不能。

## F1/F2 で画面明るさ調整 (2026-05-11 追加)

Apple MBA 2011 内蔵キーボードの F1/F2 (plain 押し) は **`KEY_BRIGHTNESSDOWN/UP` を正しく emit している** ことを evtest で確認 (`hid_apple.fnmode=3` auto = MBA 2011 では fkeysfirst 相当)。Fn+F1/F2 は素の F1/F2。

しかし `xfce4-power-manager` で `handle-brightness-keys=true` でも明るさは変わらなかった。原因:

1. **xrandr Backlight プロパティが未公開**: `acpi_backlight=native` で `intel_backlight` (type=raw) のみになると、Xorg/intel ドライバは Backlight RANDR プロパティを公開しない (xrandr --verbose で `Backlight:` 行が出ない、`Brightness: 1.0` (gamma) だけ)。xfce4-power-manager の RANDR 経由制御は使えない
2. **sysfs に user 書込不可**: `/sys/class/backlight/intel_backlight/brightness` は素で `0644 root:root`
3. **logind D-Bus 経由は動く**: `busctl call org.freedesktop.login1 /org/freedesktop/login1/session/auto org.freedesktop.login1.Session SetBrightness ssu backlight intel_backlight <値>` で書ける (確認済)。だが xfce4-power-manager はこの経路を使わない

**結論的構成 (2026-05-11):**

- `brightnessctl` (apt) 導入。`/lib/udev/rules.d/` の `brightness-udev` rule が `/sys/class/backlight/*/brightness` を `chgrp video; chmod g+w` する
- `usermod -aG video bacon` で書込権限獲得 (要再ログイン)
- `xfce4-power-manager` の `handle-brightness-keys` を `false` に (key grab を譲る)
- `xfce4-power-manager` の `brightness-switch-restore-on-exit` を `0` に (起動時に低輝度で開始する問題の根治)
- `xfce4-keyboard-shortcuts` の `/commands/custom/XF86MonBrightnessUp` = `brightnessctl set +5%`、`/commands/custom/XF86MonBrightnessDown` = `brightnessctl set 5%-`

**リンゴロゴについて**: MBA 2011 では LCD パネル背面の「Apple 切抜き」を LCD バックライトの漏れ光で照らす構造。独立 LED ではなく `intel_backlight` の輝度に完全比例する (画面が暗ければロゴも暗い)。MacBookPro Retina 2012+ で LED 直接駆動に変更されたが、2015 年頃に完全廃止。

**設定コマンド (再現用):**
```bash
# 画面輝度
sudo apt install -y brightnessctl xfce4-power-manager-plugins
sudo usermod -aG video bacon
sudo udevadm trigger --action=add --subsystem-match=backlight
DISPLAY=:0 xfconf-query -c xfce4-power-manager -p /xfce4-power-manager/handle-brightness-keys -s false
DISPLAY=:0 xfconf-query -c xfce4-power-manager -p /xfce4-power-manager/brightness-switch-restore-on-exit -s 0
# 再ログイン前は `sg video -c '...'` ラップで brightnessctl を呼ぶ必要あり (現セッションには video group が無いため)。再ログイン後は直接呼べる
DISPLAY=:0 xfconf-query -c xfce4-keyboard-shortcuts -p '/commands/custom/XF86MonBrightnessUp'   -n -t string -s "sg video -c 'brightnessctl set +5%'"
DISPLAY=:0 xfconf-query -c xfce4-keyboard-shortcuts -p '/commands/custom/XF86MonBrightnessDown' -n -t string -s "sg video -c 'brightnessctl set 5%-'"

# キーボード LED (UPower 経由、input group 不要)
sudo tee /usr/local/bin/baconmac-kbd-backlight >/dev/null <<'SH'
#!/bin/sh
STEP=26
cur=$(cat /sys/class/leds/smc::kbd_backlight/brightness 2>/dev/null) || exit 1
case "$1" in
  up)   new=$((cur + STEP)); [ "$new" -gt 255 ] && new=255 ;;
  down) new=$((cur - STEP)); [ "$new" -lt 0 ]   && new=0 ;;
  *) exit 1 ;;
esac
busctl call org.freedesktop.UPower /org/freedesktop/UPower/KbdBacklight \
  org.freedesktop.UPower.KbdBacklight SetBrightness i "$new" >/dev/null
SH
sudo chmod 0755 /usr/local/bin/baconmac-kbd-backlight
DISPLAY=:0 xfconf-query -c xfce4-keyboard-shortcuts -p '/commands/custom/XF86KbdBrightnessUp'   -n -t string -s '/usr/local/bin/baconmac-kbd-backlight up'
DISPLAY=:0 xfconf-query -c xfce4-keyboard-shortcuts -p '/commands/custom/XF86KbdBrightnessDown' -n -t string -s '/usr/local/bin/baconmac-kbd-backlight down'

# Panel に power-manager-plugin 追加 (battery + 輝度スライダー)
DISPLAY=:0 xfconf-query -c xfce4-panel -p /plugins/plugin-12 -n -t string -s 'power-manager-plugin'
DISPLAY=:0 xfconf-query -c xfce4-panel -p /panels/panel-1/plugin-ids \
  -t int -t int -t int -t int -t int -t int -t int -t int -t int -t int -t int -t int \
  -s 1 -s 2 -s 3 -s 4 -s 5 -s 6 -s 12 -s 7 -s 11 -s 8 -s 9 -s 10 --force-array
DISPLAY=:0 xfce4-panel --restart
```

## ターミナル / スナップ / パネル / ジェスチャ刷新 (2026-05-12 追加)

| 課題 | 原因 | 対策 |
|---|---|---|
| ターミナル Ctrl+C/V でコピペ不能 | xfce4-terminal は「選択中のみ Ctrl+C コピー、それ以外は SIGINT」のスマート動作未対応 (Bugzilla #11975 open)。kanata 経由で Cmd+C/V → Super+C/V に変換し accels.scm にバインドする macOS 風 workaround を使っていた | **GNOME Terminal に乗り換え** (libvte 系で HD3000 でも軽量、スマート Ctrl+C 対応)。`apt install gnome-terminal dconf-cli`、`update-alternatives --set x-terminal-emulator /usr/bin/gnome-terminal.wrapper`、`~/.config/xfce4/helpers.rc` に `TerminalEmulator=gnome-terminal`、xfconf `xfce4-keyboard-shortcuts /commands/custom/<Primary><Alt>t = gnome-terminal`。kanata の defsrc / base / modlayer から `c v` 関連を削除し v7 へ |
| panel-1/panel-2 が編集不可 | `/panels/panel-N/position-locked=true` でロックされていた | `xfconf-query -c xfce4-panel -p /panels/panel-1/position-locked -s false`、`panel-2` も同様。`xfce4-panel --restart` |
| 画面端ドラッグでスナップしない | `tile_on_move=true` 設定済だが起動以来 xfwm4 を reload していなかった | `xfwm4 --replace` で WM 再ロード |
| 3本指/4本指ジェスチャ不可 | gesture daemon 未導入 | **fusuma** 3.12.0 を gem install。`input` group には bacon を入れず、**touchpad のみ ACL 付与する udev rule** `/etc/udev/rules.d/71-fusuma-touchpad.rules` で `SUBSYSTEM=="input", ENV{ID_INPUT_TOUCHPAD}=="1", TAG+="uaccess"` を追加。`/dev/input/event7` (bcm5974) のみ active session user に ACL が付き、keyboard event4 は root:input 維持。`~/.config/fusuma/config.yml` で 3本指 swipe = ブラウザ前後/最大化、4本指 swipe = ワークスペース切替、2本指 pinch = ズーム。`~/.config/autostart/fusuma.desktop` で自動起動 |

### 重要な罠

- **udev rule 番号**: `40-fusuma-touchpad.rules` だと `60-input-id.rules` (`ID_INPUT_TOUCHPAD` を設定する) よりも先に走ってしまい match 失敗。`71-fusuma-touchpad.rules` で確実に後で走らせる必要あり。`73-seat-late.rules` の `RUN{builtin}+="uaccess"` が TAG=="uaccess" を実 ACL に変換するので 73 より前であること
- **input group の罠**: `usermod -aG input bacon` は楽だがキーボード event4 も読めるようになりキーロガリスク。touchpad だけに ACL を絞る上記方式が canonical
- **fusuma の Permission denied ログ**: event0/1/2/3/4/5/6/8-13 で出るが **event7 (touchpad) は読めている** ので機能上は問題なし。気になるなら filter で抑制可能だが触らない方が安全
- **kanata restart 時の入力フリーズ**: `systemctl restart kanata` の瞬間 1-2 秒キーが効かない。リモート作業中は注意

## Windows 風 UX (タイル / ジェスチャ / Dock / クリップボード履歴) (2026-05-13 追加)

前日 (2026-05-12) の UX 改善で動かなかった項目を順に潰し、Windows と macOS の良い所取りで仕上げた。

### 1. tile_on_move が効かなかった → `wrap_windows=false`

XFCE 4.20 で `tile_on_move` と `wrap_windows` (ドラッグ時にワークスペース wrap) が排他になる "by design な変更" がある。`tile_on_move=true` だけでは効かず、`wrap_windows=true` を切る必要があった。

```bash
xfconf-query -c xfwm4 -p /general/wrap_windows -s false
```

### 2. Cmd+Arrow で Windows 風スプリット

kanata v8 で arrow を modlayer 経由で Super+Arrow に変換。xfwm4 側の Super+矢印は既に半分タイル bind 済だったが、`<Super>Up/Down` を tile_up/down_key (上/下半分) から **maximize_window_key / hide_window_key** に書換えて Windows 互換に。

```bash
xfconf-query -c xfce4-keyboard-shortcuts -p '/xfwm4/custom/<Super>Up'   -r
xfconf-query -c xfce4-keyboard-shortcuts -p '/xfwm4/custom/<Super>Up'   -n -t string -s 'maximize_window_key'
xfconf-query -c xfce4-keyboard-shortcuts -p '/xfwm4/custom/<Super>Down' -r
xfconf-query -c xfce4-keyboard-shortcuts -p '/xfwm4/custom/<Super>Down' -n -t string -s 'hide_window_key'
xfwm4 --replace &      # ランタイムへの取り込みを強制
```

罠: xfwm4 4.20 は xfconf からの keybind 上書きをランタイムで再ロードしないことがある (Launchpad bug #1292290)。`-r` 削除→`-n -s` 再作成 + `xfwm4 --replace` の二段が canonical。

最終キー対応 (Cmd ホールド前提):
| キー | アクション |
|---|---|
| Cmd+← | 左半分タイル (tile_left_key) |
| Cmd+→ | 右半分タイル (tile_right_key) |
| Cmd+↑ | 最大化トグル (maximize_window_key) |
| Cmd+↓ | 最小化 (hide_window_key) |

### 3. Docklike taskbar + panel-2 削除

panel-2 (中央下の launcher 列) は削除、panel-1 の `plugin-2` を tasklist → **docklike** に置換。Windows 11 風のピン留め + ウィンドウグループ化 + クリックで起動/切替。

```bash
sudo apt install xfce4-docklike-plugin
xfconf-query -c xfce4-panel -p /plugins/plugin-2 -s docklike
# panel-2 を panels リストから外し、panel-2 関連プロパティ全削除
xfconf-query -c xfce4-panel -p /panels -t int -s 1 --force-array
xfconf-query -c xfce4-panel -p /panels/panel-2 -r -R
xfce4-panel --restart
```

ピン留めは「**起動中アプリのアイコンを右クリック → "Pin to Dock"**」。空きドック領域ではメニュー出ない。

### 4. トラックパッド: 右下角クリック + タップでクリック

Apple MBA トラックパッドは libinput デフォルトで **clickfinger 方式** (1本指物理クリック=左、2本指物理クリック=右)、かつ **Tap-to-click 無効**。ユーザー直感に合わせて以下に変更:
- **物理クリック**: `button-areas` (右下角押し = 右クリック、Windows ノート風)
- **タップ**: 有効化 (1本指タップ=左、2本指タップ=右)

`/etc/X11/xorg.conf.d/40-libinput-touchpad.conf` で永続化:

```
Section "InputClass"
    Identifier "Apple bcm5974 trackpad"
    MatchDriver "libinput"
    MatchIsTouchpad "on"
    MatchProduct "bcm5974"
    Option "ClickMethod" "button-areas"
    Option "Tapping" "on"
    Option "TappingButtonMap" "lrm"
    Option "TappingDrag" "on"
    Option "NaturalScrolling" "true"
EndSection
```

ライブ反映は `xinput set-prop` で。

### 5. 3本指/4本指ジェスチャ (fusuma) + skippy-xd

`fusuma` 3.12.0 (Ruby gem) でジェスチャ daemon。`input` group を避け **touchpad のみ uaccess を付与する udev rule** で安全運用。設定 `~/.config/fusuma/config.yml`:

| 操作 | コマンド |
|---|---|
| 3本指 ← / → | xdotool key alt+Right / alt+Left (ブラウザ前後) |
| 3本指 ↑ / ↓ | 最大化 / 最大化解除 (wmctrl) |
| 4本指 ← / → | ワークスペース切替 (wmctrl -s) |
| 4本指 ↑ | **skippy-xd --toggle-window-picker** (Mission Control 風) |
| 4本指 ↓ | デスクトップ表示 (xdotool key super+d) |
| 2本指 pinch | Ctrl+± (ズーム) |

skippy-xd は apt に無いため github.com/richardgv/skippy-xd をソースビルド。依存は apt 内 (`libimlib2-dev libxft-dev libxinerama-dev libxcomposite-dev libxdamage-dev libxfixes-dev libxmu-dev`)。`make install` で /usr/bin/skippy-xd 配置、autostart で `skippy-xd --start-daemon` 常駐。

### 6. クリップボード履歴 (xfce4-clipman) — panel plugin は使わず daemon 直起動

**重要な罠**: `xfce4-clipman-plugin` の **panel plugin (libclipman.so) は libdocklike と同居すると load 失敗** する模様。「プラグインのロード失敗」「プラグインを再起動」ダイアログが大量に出続けるため、panel-1 への組込は**諦め**、standalone daemon `xfce4-clipman` を autostart 経由で起動する方式に切替。daemon は D-Bus に `org.xfce.clipman` を出すので、`xfce4-popup-clipman` から呼び出せる。

```bash
sudo apt install xfce4-clipman-plugin     # 実体は daemon バイナリ + plugin lib 同梱、daemon だけ使う
# autostart
cat > ~/.config/autostart/xfce4-clipman.desktop <<EOF
[Desktop Entry]
Type=Application
Name=Clipman
Exec=xfce4-clipman
X-GNOME-Autostart-enabled=true
EOF
```

settings (xfconf チャンネル `xfce4-panel`、base `/plugins/clipman`):

| キー | 値 | 効果 |
|---|---|---|
| `/tweaks/popup-at-pointer` | true | popup を**マウスカーソル位置に**表示 (デフォルトは固定位置) |
| `/tweaks/paste-on-activate` | true | 履歴選択時に **Ctrl+V を自動送信**して即貼り付け |
| `/settings/add-primary-clipboard` | true | PRIMARY (選択) と CLIPBOARD (Ctrl+C) を同期して履歴に追加 |

XFCE keyboard shortcut で `<Primary><Super>v` (Ctrl+Super+V) → `xfce4-popup-clipman` を bind。kanata v9 で Cmd+V を Super+V に変換しておくと、ユーザーの **Cmd+Ctrl+V** 押下が物理 Ctrl + kanata 発射の Super+V = **Ctrl+Super+V** として成立 → popup 起動。

### 7. その他のキーバインド更新

| シ ortcut | 動作 |
|---|---|
| `<Primary><Alt>t` | gnome-terminal (元は exo-open --launch TerminalEmulator) |
| `<Super>Up` | maximize_window_key (元 tile_up_key) |
| `<Super>Down` | hide_window_key (元 tile_down_key) |
| `<Super>Left/Right` | tile_left/right_key (現状維持) |
| `<Primary><Super>v` | xfce4-popup-clipman |

### 重要な罠 (今回追加分)

- **xfwm4 4.20 keybind ランタイム反映**: `xfconf-query -s` だけでは取りこぼす場合あり。`-r` で削除してから `-n -t string -s` で再作成 → `xfwm4 --replace` で完全反映
- **xfwm4 4.20 tile_on_move 排他バグ**: `wrap_windows=true` だと tile_on_move が無効化される
- **docklike "Pin to Dock"**: 起動中アプリのアイコン**だけ**で出るメニュー。空き領域では出ない
- **トラックパッド右クリック誤解**: libinput clickfinger は「1本指=左、2本指=右」のため、ユーザーが「右側を押した」と思った操作は実質 1本指=左クリックになる。`button-areas` か Tap で 2-finger right に切替が必要
- **clipman panel plugin load 失敗**: 原因未特定だが daemon 単独運用で完全回避可能。daemon でも D-Bus 経由で `xfce4-popup-clipman` から呼べる
- **clipman の popup 位置**: デフォルトは panel icon 付近を狙うため、panel plugin 無しで使うと "固定された場所" になる。`/plugins/clipman/tweaks/popup-at-pointer=true` で解決
- **fusuma 4本指 swipe が log に出ない場合**: libinput 側で 4-finger ジェスチャを認識する必要あり。bcm5974 + libinput 1.31 は対応済だが、トラックパッド面積が小さい MBA 2011 では 4本指の認識が時々滑る (実害は小さい)

### コマンドまとめ (今回追加分の再現用)

```bash
# 1. wrap_windows
xfconf-query -c xfwm4 -p /general/wrap_windows -s false

# 2. xfwm4 keybind workaround
for k in Up Down; do
  xfconf-query -c xfce4-keyboard-shortcuts -p "/xfwm4/custom/<Super>$k" -r
done
xfconf-query -c xfce4-keyboard-shortcuts -p '/xfwm4/custom/<Super>Up'   -n -t string -s 'maximize_window_key'
xfconf-query -c xfce4-keyboard-shortcuts -p '/xfwm4/custom/<Super>Down' -n -t string -s 'hide_window_key'
xfwm4 --replace &

# 3. docklike + panel-2 削除
sudo apt install -y xfce4-docklike-plugin
xfconf-query -c xfce4-panel -p /plugins/plugin-2 -s docklike
xfconf-query -c xfce4-panel -p /panels -t int -s 1 --force-array
xfconf-query -c xfce4-panel -p /panels/panel-2 -r -R
xfce4-panel --restart

# 4. touchpad
sudo tee /etc/X11/xorg.conf.d/40-libinput-touchpad.conf >/dev/null <<'EOF'
Section "InputClass"
    Identifier "Apple bcm5974 trackpad"
    MatchDriver "libinput"
    MatchIsTouchpad "on"
    MatchProduct "bcm5974"
    Option "ClickMethod" "button-areas"
    Option "Tapping" "on"
    Option "TappingButtonMap" "lrm"
    Option "TappingDrag" "on"
    Option "NaturalScrolling" "true"
EndSection
EOF
# live 反映
bcm_id=$(xinput list | grep -i bcm5974 | grep -oE 'id=[0-9]+' | head -1 | cut -d= -f2)
xinput set-prop $bcm_id 'libinput Click Method Enabled' 1 0
xinput set-prop $bcm_id 'libinput Tapping Enabled' 1

# 5. fusuma touchpad uaccess (input group 回避)
sudo tee /etc/udev/rules.d/71-fusuma-touchpad.rules >/dev/null <<'EOF'
SUBSYSTEM=="input", ENV{ID_INPUT_TOUCHPAD}=="1", TAG+="uaccess"
EOF
sudo udevadm control --reload-rules
sudo udevadm trigger --action=add --subsystem-match=input

# 6. skippy-xd build
sudo apt install -y git libimlib2-dev libfontconfig1-dev libfreetype6-dev libx11-dev libxext-dev libxft-dev libxrender-dev zlib1g-dev libxinerama-dev libxcomposite-dev libxdamage-dev libxfixes-dev libxmu-dev libconfig-dev pkg-config
git clone --depth 1 https://github.com/richardgv/skippy-xd.git ~/src/skippy-xd
cd ~/src/skippy-xd && make && sudo make install
cp skippy-xd.sample.rc ~/.config/skippy-xd/skippy-xd.rc

# 7. clipman daemon + settings
sudo apt install -y xfce4-clipman-plugin
xfconf-query -c xfce4-panel -p /plugins/clipman/tweaks/popup-at-pointer -n -t bool -s true
xfconf-query -c xfce4-panel -p /plugins/clipman/tweaks/paste-on-activate -n -t bool -s true
xfconf-query -c xfce4-panel -p /plugins/clipman/settings/add-primary-clipboard -n -t bool -s true
xfconf-query -c xfce4-keyboard-shortcuts -p '/commands/custom/<Primary><Super>v' -n -t string -s 'xfce4-popup-clipman'

# 8. kanata v9 (Arrow + V を modlayer に追加)
# /etc/kanata/baconmac.kbd: 上記 v9 内容で上書き
sudo systemctl restart kanata
```

## 切り分け履歴 (重要な学び)

1. 「ランダムハング」で当初 ASPM/C-state を疑い対策 → 部分的解決
2. SATA cycle 観測 → SSD DIPM 問題と判明、libata.force=nolpm で完全停止
3. AC flap で GPE17 storm → AC driver unbind systemd で完全遮断
4. それでも hang 残る → idle test で再現確認
5. brcmsmac unload + 30 分 idle 安定確認 → 一時的に「真の主因」と誤認
6. 後に **PCI MSI が真の問題**と判明 (Arch Wiki MacBookAir4,1 既知 workaround)
7. `pci=nomsi` 追加 + brcmsmac 復活 → **30 分 idle hang ゼロ**で WiFi 完全復活
8. fcitx5 hotkey config syntax 罠で IME 切替動かず → `[Hotkey/TriggerKeys]` 別 section 必須
9. kanata の chord は overlap で発火しない → layer-while-held 方式に書換で解決
10. 「ShiftキーがIME切替」は fcitx5/kanata 由来ではなく **mozc の `shift_key_mode_switch` デフォルト**だった (registry.db 12byte = 完全デフォルト適用)
11. `IconThemeName=elementary-xfce-dark` が反映されない → 実体は snap 内 (`/snap/gtk-common-themes/...`) で XDG 検索パスに無く、`/usr/share/icons/` に置く必要 (`elementary-xfce-icon-theme` deb)
12. xfwm4 の tile アクションは標準実装済 (`tile_on_move=true` で画面端ドラッグ tiling、`tile_*_key` で半分タイル) だが MBA キーボードに無いテンキー (`<Super>KP_*`) にしかバインドされてなかった
13. mbpfan のデフォルト閾値 (low=63/high=66/max=86) は MBA Mid 2011 の体感では遅すぎる。i7-2677M は軽負荷でも 60°C 前後を維持しがちで「ファンが回ってない」印象になる。実用には low=55/high=60/max=80 程度に下げないと追従感が出ない
14. 蓋閉じ LED OFF: 当初 `acpi_video0/bl_power=4` を acpid handler で叩く設計 → SW_LID input event がカーネル警告とともに死ぬ事象を発見、polling 方式に切替。Apple ACPI _BCM/_PSx は no-op で sysfs 値だけ動き HW は不変 → `acpi_backlight=native` で `intel_backlight` 露出が canonical 解決。途中 `brightness=0` を試して復帰不能で再起動した経緯あり (Apple ファーム既知バグ、`brightness=1` を退避値に採用)。`xset dpms force off` も blank only で実バックライトは切れない既知問題 (ArchWiki / Linux Mint bug 1636857)
15. F1/F2 で明るさ動かない: 当初「キー emit されてないのか」「fnmode の問題か」「kanata が食ってるか」を疑った → evtest で kanata 仮想出力 (event13) を観察し、plain F1/F2 が `KEY_BRIGHTNESSDOWN/UP` を正しく emit していることを確認。問題は **書き込み側** だった: `intel_backlight` (type=raw) は xrandr Backlight プロパティ未公開 + user 書込不可で、xfce4-power-manager は key を受けても無音失敗。`brightnessctl` + `video` group + xfce keybinding に直接バインドが最短路。logind D-Bus (`SetBrightness ssu backlight intel_backlight N`) も書込手段として動作することを確認 (代替路として記録)。リンゴロゴが暗かったのは LCD バックライトと一体構造 + `brightness-switch-restore-on-exit=1` で前回低輝度値が起動毎に復元されていたため
16. F5/F6 でキーボード LED 動かない: 同じ evtest で plain F5/F6 が `KEY_KBDILLUMDOWN/UP` を emit していることを確認。書込先 `/sys/class/leds/smc::kbd_backlight/brightness` は brightnessctl 同梱 udev で `0664 root:input` になるが、`input` group はキーボード入力読取権限 (キーロガ系) も付くため避けたい。代わりに **UPower の `org.freedesktop.UPower.KbdBacklight.SetBrightness` (D-Bus)** が polkit でアクティブセッションに書込許可されており group 不要で動く。`/usr/local/bin/baconmac-kbd-backlight` でラップして `XF86KbdBrightnessUp/Down` にバインド
17. **xfwm4 4.20 で tile_on_move と wrap_windows が排他になった**: 設定上 `tile_on_move=true` でも `wrap_windows=true` のままだとドラッグ tile が無効化される。デフォルトオフだが過去 GUI で触っていたため true になっていた。"by design" だが事前情報なしには絶対に踏む
18. **xfwm4 4.20 keybind が xfconf 変更で即反映されない**: `xfconf-query -p ... -s 'new_action'` だけだと runtime に取り込まれない。Launchpad bug #1292290 のとおり `-r` で一度削除 → `-n -t string -s 'new_action'` で再作成 → `xfwm4 --replace` の三段でようやく確実に反映。`tile_down_key` から `hide_window_key` への変更が「中途半端な位置に最小化」現象を起こしたのはこのため (古い tile が裏でまだ生きていた)
19. **トラックパッド「右側押し = 左クリック」の謎**: libinput デフォルトの **clickfinger** 方式は「1本指物理クリック=左、2本指物理クリック=右」。ユーザーが「右側を押した」つもりでも 1本指なら左扱い。`ClickMethod=button-areas` で右下角押しを右クリックに変えるか、`Tapping=on` で 2本指タップを右クリックに割り当てる必要。`Tap-to-click: disabled` がデフォルトなのは clickpad 系を考慮した安全側設定
20. **xfce4-clipman の panel plugin が同居環境で load 失敗**: docklike + power-manager-plugin + pulseaudio などの外部 plugin が並んだ panel-1 に clipman を追加すると `libclipman.so` のロードに失敗、「プラグインのロード失敗」「プラグインを再起動」ダイアログが連続で出る。回避: **standalone daemon `xfce4-clipman` を autostart 起動**。daemon でも D-Bus `org.xfce.clipman` を出すので `xfce4-popup-clipman` から呼べる。panel icon は失うが popup ベース運用なら問題なし
21. **clipman popup 位置の固定問題**: panel plugin 無しで daemon 単独運用すると、popup は固定座標に出る。`/plugins/clipman/tweaks/popup-at-pointer=true` (hidden setting) でマウスカーソル位置追従に。**`/plugins/clipman/tweaks/paste-on-activate=true`** で選択時の Ctrl+V 自動送信も合わせると Windows Win+V 完全互換 (選んだだけで貼り付くまで完結)
22. **fusuma を input group 無しで動かす canonical な方法**: `usermod -aG input bacon` はキーボード event4 も読めるためキーロガ系の権限拡大。代わりに `/etc/udev/rules.d/71-fusuma-touchpad.rules` に `SUBSYSTEM=="input", ENV{ID_INPUT_TOUCHPAD}=="1", TAG+="uaccess"` を置き、`73-seat-late.rules` の uaccess builtin がアクティブセッションユーザーに touchpad event7 だけ ACL を付与する。rule ファイル名は `60-input-id.rules` 後・`73-seat-late.rules` 前 (= 70 番台) 必須
23. **kanata tap-hold の chord 発火**: `(tap-hold 200 200 lmet (layer-while-held modlayer))` で Cmd を 200ms 以上保持すると modlayer 入り。素早く Cmd→矢印すると tap 判定→Super_L 単発+矢印単発に分解されることがあり Super+矢印 として届かない。実用上は意識的に「Cmd 押す→少し待つ→矢印」で確実。早押し対策が必要なら `tap-hold-press` 変種 + 数値を 150 等に下げる
24. **systray に WiFi アイコンが2個出る (nm-applet 二重表示)**: nm-applet は `libayatana-appindicator3` にリンクされており、SNI host (xfce4-panel の systray plugin) が見えれば SNI mode で動く。が、**autostart 経由で xfce4-panel より先に起動**すると SNI host が見つからず `GtkStatusIcon` (XEmbed legacy) に fallback してアイコンを embed → 後から SNI host が立ち上がっても fallback icon が掃除されず、SNI mode の新アイコンと並んで2個表示される。`xfconf-query -c xfce4-panel -p /plugins/plugin-6/known-legacy-items` に `ネットワーク` (locale=ja) と `network` (locale=C) の両方が残るのが症状指標。検証: `pkill -x nm-applet && sleep 2 && nm-applet &` で panel 起動後に再起動すると SNI 1個だけになる。**恒久対応**: `~/.config/autostart/nm-applet.desktop` の `Exec=nm-applet` を `Exec=sh -c 'sleep 5 && exec nm-applet'` に変更し SNI host より後に起動させる (2026-05-23 適用)

詳細な切り分け過程は Windows 側 `C:\Users\bacon\OneDrive\デスクトップ\baconmac-トラブル原因と解決策.md`。
