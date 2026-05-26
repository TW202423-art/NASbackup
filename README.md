# NASbackup

使用 `Tailscale` 與 `TrueNAS ZFS Replication` 建置雙站點異地備份的規劃專案。

## 專案簡介

本專案規劃兩個彼此隔離站點的 TrueNAS 資料備份架構。兩端 NAS 不透過公開 Internet 直接提供管理或備份服務，而是加入專用的 Tailscale 私有網路，再利用 TrueNAS 原生的 ZFS snapshot replication 傳送異地備份。

方案目標：

- 讓站點 A 與站點 B 的重要資料各自保有異地副本。
- 保留原本內部網路隔離，不將兩側整個 LAN 互相打通。
- 使用 ZFS snapshot 與 replication 保留可復原的時間點版本。
- 支援未來增加第三個備份站點或中央備份 NAS。
- 搭配離線或不可變副本，降低勒索軟體與誤刪風險。

## 採用方案

```text
                      Tailscale 專用私有網路
              僅允許 TrueNAS 間 SSH Replication

站點 A                                                站點 B
+------------------------+                           +------------------------+
| TrueNAS-A              |                           | TrueNAS-B              |
| 正式 Dataset            |                           | 正式 Dataset            |
| Periodic Snapshot      |                           | Periodic Snapshot      |
|                        |                           |                        |
| remote-backup-b       |<==== ZFS Replication ====>| remote-backup-a       |
+------------------------+                           +------------------------+

A 正式資料的 snapshot -> 複寫至 B 的 remote-backup-a
B 正式資料的 snapshot -> 複寫至 A 的 remote-backup-b
```

## 為什麼使用 Tailscale

`Tailscale` 用於提供兩台 TrueNAS 之間的加密私有連線；備份本身仍由 TrueNAS 負責。


| 項目         | 說明                                                         |
| ------------ | ------------------------------------------------------------ |
| TrueNAS 支援 | TrueNAS SCALE 可由 Apps 安裝 Tailscale，官方文件提供操作方向 |
| 網路需求     | 可因應動態 IP、NAT 或未來增加站點                            |
| 安全範圍     | 只讓指定 NAS 節點互通，不需要公告兩邊完整 LAN                |
| 備份搭配     | TrueNAS replication 可使用 Tailscale IP 建立 SSH Connection  |
| 管理彈性     | 可使用 ACL 僅允許備份所需的連線                              |

## 備份設計原則

組網與備份是兩件不同的事情：

```text
Tailscale                    = 加密連線與節點存取控制
TrueNAS Periodic Snapshot    = 建立資料時間點版本
TrueNAS Remote Replication   = 將 snapshot 複寫到異地 NAS
離線/不可變副本              = 防止線上備份同時遭破壞
```

不採用一般 SMB 雙向同步作為主要備份，因為誤刪或勒索加密結果可能被同步至另一端。

## 安全設定要求


| 設定項目         | 本專案原則                                                     |
| ---------------- | -------------------------------------------------------------- |
| Tailscale 節點   | 僅加入兩台 TrueNAS 與必要的 IT 管理端                          |
| Tailscale ACL    | 僅允許 TrueNAS-A 與 TrueNAS-B 之間的 replication SSH 連線      |
| Subnet Routes    | 備份用途不啟用，不將站點 LAN 路由到另一站                      |
| Exit Node        | 不啟用                                                         |
| TrueNAS 管理介面 | 不直接暴露於公開 Internet                                      |
| 備份目的 Dataset | 不提供一般使用者作為工作資料夾存取                             |
| 帳號與憑證       | 不將密碼、Auth Key、SSH Private Key 或 Recovery Key 提交至 Git |
| 額外副本         | 關鍵資料另保留離線或不可變備份                                 |

## 導入階段


| 階段              | 作業項目                                                           | 完成條件                              |
| ----------------- | ------------------------------------------------------------------ | ------------------------------------- |
| 1. 設備確認       | 確認兩端為`TrueNAS SCALE 24.10+`、版本、Pool、容量與需備份 Dataset | 完成設備與資料清單                    |
| 2. Tailscale 組網 | 於兩台 TrueNAS 安裝 Tailscale App，加入專用 tailnet                | 兩台 NAS 可使用 Tailscale IP 互相到達 |
| 3. 存取限制       | 建立 ACL，只允許 NAS replication 所需連線，不公告站點 LAN          | 一般網路無法跨站存取                  |
| 4. Snapshot       | 建立 TrueNAS`Periodic Snapshot Tasks`                              | 可看見排程產生的 snapshot             |
| 5. Replication    | 建立 A 到 B 與 B 到 A 的`Remote Replication Tasks`                 | 首次完整複寫成功                      |
| 6. 驗證復原       | 執行單檔還原與一份 Dataset 測試還原                                | 確認備份可讀、可復原                  |
| 7. 強化保護       | 對關鍵資料加入離線或不可變副本                                     | 有獨立於兩台線上 NAS 的副本           |

## 初始排程建議


| 資料類型     | 本機 Snapshot   | 異地 Replication | 額外副本             |
| ------------ | --------------- | ---------------- | -------------------- |
| 關鍵資料     | 每 1 至 4 小時  | 每日一次以上     | 每週離線或不可變副本 |
| 重要共享資料 | 每 4 小時或每日 | 每日離峰時段     | 每月副本             |
| 一般歸檔資料 | 每日或每週      | 每週             | 依容量需求安排       |

實際排程需依資料量、每日變更量、兩端上傳頻寬以及可容忍的資料損失時間調整。

## 專案文件

- [TrueNAS 異地備份組網軟體選型](NETWORK_SOFTWARE_SELECTION.md)：說明為何採用 Tailscale，以及 NetBird、ZeroTier、Site-to-Site VPN 等替代選項。

## 建置前待確認資料


| 項目                    | TrueNAS-A | TrueNAS-B |
| ----------------------- | --------- | --------- |
| TrueNAS SCALE 版本      | 待填      | 待填      |
| Pool 名稱及可用容量     | 待填      | 待填      |
| 需備份 Dataset 與資料量 | 待填      | 待填      |
| 每日異動資料量          | 待填      | 待填      |
| Internet 上傳頻寬       | 待填      | 待填      |
| 是否已有離線/不可變副本 | 待填      | 待填      |
