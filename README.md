# MacbookA1370Repair

Apple **MacBook Air (Mid 2011, MacBookAir4,1 / A1370)** を Linux (Ubuntu + XFCE) で安定稼働させるためのトラブル史 + 現行構成記録。

## 機体仕様

| 項目 | 内容 |
|---|---|
| 機種 | MacBook Air (Mid 2011) |
| Model Identifier | MacBookAir4,1 |
| EMC | A1370 |
| CPU | Intel Core i7-2677M (Sandy Bridge) |
| GPU | Intel HD Graphics 3000 |
| WiFi | Broadcom BCM43224 |
| RAM | 3.7 GiB |
| Storage | SanDisk SD9TN8W M.2 SSD (アダプタ経由) |
| OS | Ubuntu 26.04 LTS Server + XFCE 4.20 (X11) |
| Kernel | 7.0.0-14-generic |

## ドキュメント

- **[baconmac-setup.md](./baconmac-setup.md)** — 全ての対策・現行構成・切り分け履歴を集約したマスタードキュメント

## 対策済の主要トピック

- ランダムハング (ASPM / SATA DIPM / ACPI GPE17 / brcmsmac MSI) を GRUB cmdline で完全停止
- 蓋閉じ時の LCD / リンゴロゴ / KBD LED の完全消灯 (`acpi_backlight=native` + polling daemon)
- F1/F2 輝度 + F5/F6 KBD LED の権限処理 (`input` group 不要の UPower D-Bus 経路)
- ファン制御 (`mbpfan` 閾値調整)
- TLP は完全停止 (USB Ethernet 切断 / SSD hang の元)
- XFCE UX 整備: macOS / Windows 風キーバインド、トラックパッドジェスチャ (fusuma)、Mission Control 風 (skippy-xd)、クリップボード履歴 (xfce4-clipman)、Docklike taskbar

## ライセンス / 用途

個人記録。同じ A1370 を Linux 化する人の参考用に公開。
