# TrueNAS 異地備份組網軟體選型

## 1. 結論

你的目標是讓多個站點的 TrueNAS 安全地互相到達，以執行 `ZFS snapshot replication`。組網軟體只負責提供私有加密連線，備份本身仍由 TrueNAS 的 `Periodic Snapshot Tasks` 與 `Remote Replication Tasks` 負責。

目前建議順序如下：


| 優先順序 | 適用情境                                               | 推薦方案                                                     |
| -------- | ------------------------------------------------------ | ------------------------------------------------------------ |
| 1        | 使用`TrueNAS SCALE 24.10+`、希望最快落地、多點擴充容易 | `Tailscale + TrueNAS ZFS Replication`                        |
| 2        | 必須完全自行掌控控制平面、偏好可 self-host 的開源方案  | `NetBird self-hosted + TrueNAS ZFS Replication`              |
| 3        | 已有可管理的企業級防火牆、只有兩個固定站點             | `Site-to-Site WireGuard/IPsec VPN + TrueNAS ZFS Replication` |
| 備選     | 已熟悉 ZeroTier 或需要其虛擬網路/Flow Rules 功能       | `ZeroTier + TrueNAS ZFS Replication`                         |

以目前資訊而言，我最先推薦 `Tailscale`。理由是 TrueNAS 官方文件直接提供其 App 安裝方式，而 Tailscale 也有 TrueNAS SCALE 連接多台 NAS 執行 ZFS replication 的專門文件。若你明確要求「控制服務不得由外部廠商管理」，則首選改為 `NetBird self-hosted`。

## 2. 為什麼不直接選 ZeroTier

ZeroTier 可以完成任務，並非不好。TrueNAS Apps Market 有 ZeroTier App，TrueNAS 官方遠端存取文件也提供安裝說明；它很適合需要進階虛擬網路與 Flow Rules 的人。

但在本案中，你主要需求是讓 TrueNAS 之間穩定地執行 replication，而不是建立複雜的 Layer 2 虛擬網路。`Tailscale` 的 TrueNAS/ZFS replication 文件較直接，首次導入時較容易維運與交接，因此列為一般情境首選。

## 3. 方案比較


| 方案                                | TrueNAS SCALE 導入                                         | 多點擴充               | 自行託管控制平面                     | 對本案評價               |
| ----------------------------------- | ---------------------------------------------------------- | ---------------------- | ------------------------------------ | ------------------------ |
| Tailscale                           | TrueNAS 官方文件與 App 可用                                | 容易                   | 可另評估 Headscale，但會增加維運工作 | 最推薦快速落地           |
| NetBird                             | NetBird 文件提供 TrueNAS 安裝方向；需依 SCALE 版本驗證 App | 容易                   | 支援 self-host                       | 最推薦開源自架取向       |
| ZeroTier                            | TrueNAS 官方文件與 Community App 可用                      | 容易                   | 可評估自架，管理方式需另規劃         | 可用備選                 |
| WireGuard / WG Easy                 | TrueNAS 官方文件列為可用選項                               | 站點增加時手動設定變多 | 自行管理                             | 固定兩站且懂網路時很適合 |
| 防火牆 Site-to-Site IPsec/WireGuard | 不需在 NAS 上安裝 overlay App                              | 依防火牆能力           | 自行管理                             | 企業環境兩站首選之一     |

## 4. 推薦方案 A：Tailscale 快速落地

### 適用條件

- 兩台設備是 `TrueNAS SCALE 24.10+`。
- 可接受使用 Tailscale 管理平台管理節點加入與 ACL。
- 希望日後容易加入第三個備份站或管理電腦。

### 架構

```text
Tailscale Tailnet：backup-network

TrueNAS-A (Tailscale IP) ---- SSH / ZFS Replication ---- TrueNAS-B (Tailscale IP)
        |                                                  |
        +-- 不公告站點 A 的一般 LAN                         +-- 不公告站點 B 的一般 LAN
```

### 安全設定原則


| 項目                | 建議                                                    |
| ------------------- | ------------------------------------------------------- |
| 節點加入            | 只將 TrueNAS 備份節點及必要管理端加入                   |
| Subnet routes       | 備份用途不開啟，不讓整個辦公室 LAN 跨站可達             |
| Exit node           | 不啟用                                                  |
| ACL                 | 只允許`TrueNAS-A <-> TrueNAS-B` 的 replication SSH 連線 |
| TrueNAS replication | 以 Tailscale 私有 IP 作為遠端 SSH Connection 位址       |
| MFA                 | Tailscale 管理帳號啟用 MFA                              |

### 優缺點


| 優點                                  | 缺點                          |
| ------------------------------------- | ----------------------------- |
| 對 NAT/動態 IP 友善，新增站點快       | 控制平面預設使用外部服務      |
| TrueNAS 與 Tailscale 都有對應整合文件 | 必須妥善管理 ACL 與登入帳號   |
| 不必讓整段 LAN 互通                   | NAS 上執行 VPN App 需維持更新 |

## 5. 推薦方案 B：NetBird Self-Hosted

### 適用條件

- 希望採開源且自行掌控管理服務。
- 可以維運一台獨立的小型 Linux VM 或雲端 VM 作為 NetBird 管理/連線協調服務。
- 願意在實際 TrueNAS SCALE 版本上先驗證 App 與 replication 流程。

### 架構

```text
獨立管理 VM：NetBird Self-Hosted Management / Signal / Relay
                    |
TrueNAS-A Peer -----+----- TrueNAS-B Peer
       ZFS replication only
```

NetBird 使用 WireGuard 建立 peer-to-peer 加密通道，支援 access control 與自行託管。若資料治理規定不希望節點註冊資訊由外部平台管理，此方案比託管 overlay service 更符合需求。

### 代價

你必須自行負責：

- 管理服務更新與安全修補。
- 控制服務備份與故障復原。
- 身分驗證、MFA 與憑證生命週期。
- 節點連線與 ACL 問題排查。

因此，這是「控制力較高」而不是「最省工」的選項。

## 6. 何時改用防火牆 Site-to-Site VPN

如果只有兩個固定站點，而且兩邊已有可穩定建立 IPsec/WireGuard 的防火牆，將 VPN 建在防火牆而非 NAS 通常更乾淨：

- NAS 專注於儲存與 replication，不需額外執行 overlay App。
- 網路政策集中在防火牆管理。
- 企業 IT 較容易監控與稽核。

此方案的限制是新增第三、第四站時，VPN 與路由/規則管理可能較繁複；若各站 Internet 沒有固定 IP 或處於複雜 NAT 後方，也會比 overlay network 麻煩。

## 7. 多站點擴充方式

未來如果有 A、B、C、D 多個站點，不建議所有站點相互全面複寫。建議設計一個備份中心：

```text
TrueNAS-A ----\
TrueNAS-B -----+---- Overlay Network ---- 中央異地 Backup TrueNAS
TrueNAS-C ----/
```


| 設計原則 | 說明                                                        |
| -------- | ----------------------------------------------------------- |
| 網路     | 每個站點 TrueNAS 只可達中央備份 TrueNAS 的 replication 服務 |
| 備份     | 各站以 ZFS replication Push 到中央備份 Dataset              |
| 權限     | 每站使用不同目的 Dataset 與不同授權                         |
| 保護     | 中央備份另建立離線或不可變副本                              |
| 重要站點 | 若需快速接手服務，再額外規劃特定站點的雙向災難復原          |

## 8. 下一步確認項目


| 項目                                         | 目的                               |
| -------------------------------------------- | ---------------------------------- |
| 兩台 TrueNAS 是`SCALE` 或 `CORE`，以及版本號 | 判斷 App 與設定畫面是否可用        |
| 是否接受雲端管理控制平面                     | 決定選 Tailscale 或自架 NetBird    |
| 是否已有支援 IPsec/WireGuard 的防火牆        | 判斷防火牆 Site-to-Site 是否更簡潔 |
| 未來會不會增加第三個以上站點                 | 決定兩站互備或中央備份架構         |
| 重要資料量、每日變更量與上傳頻寬             | 決定首次複寫方式與排程             |
