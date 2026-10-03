# Hyper-V Lab: AD + DNS + DHCP + NAT + SQL Server + SSMS + AD Group Authentication

**目次**
- [概要](#概要)
- [アーキテクチャ構成](#アーキテクチャ構成)
- [構築手順](#構築手順)
- [トラブルシューティング](#トラブルシューティング)
- [学びのまとめ](#学びのまとめ)

---

## 概要

このプロジェクトは、Hyper-V上で以下のインフラを一通り構築し、**ADグループ認証をSQL Serverに適用**する実践的な学習環境です。

**最終構成**
| コンポーネント | IP | 役割 |
|--------------|-----|------|
| DC01 (Internal NIC) | 192.168.50.1 | AD Server + DNS + DHCP + RRAS/NAT |
| DC01 (External NIC) | - | Internet接続 (DHCP) |
| SQL01 | 192.168.50.20 | SQL Server Express (Static) |
| DEV01 | 192.168.50.102 | Dev Workstation (DHCP) |

---

## アーキテクチャ構成

```
                   Internet
                       │
              Home / Office LAN
                       │
                ┌──────▼──────┐
                │    DC01     │
                │             │
                │ AD DS       │
                │ DNS         │
                │ DHCP        │
                │ RRAS/NAT    │
                └──────┬──────┘
                       │
                Lab-Internal
               192.168.50.0/24
                       │
            ┌─────────┴─────────┐
            │                   │
        ┌───▼───┐           ┌───▼───┐
        │ SQL01 │           │ DEV01 │
        │ 50.20 │           │ 50.102│
        └───────┘           └───────┘
             ▲                   │
             │                   │
             │              DEV\developer01
             │                   │
             └────────┬──────────┘
                    │
              SQLDevelopers
                 AD Group
```

---

## 構築手順

### 1. Hyper-Vネットワーク構築

**Virtual Switch構成**

| Switch名 | 種類 | 用途 |
|----------|------|------|
| Lab-External | External | DC01 External NIC (物理Wi-Fiに接続) |
| Lab-Internal | Internal | Lab内部ネットワーク (192.168.50.0/24) |

**DC01のNIC構成**

- **Internal NIC**: 192.168.50.1 (Static), Gatewayは設定なし
- **External NIC**: DHCP (Internet接続用)

---

### 2. DC01にAD DS構築

```powershell
# Domain Name: DEV.LOCAL
# Forest type: New Forest
```

**Promotion時の注意点**:
- 警告: "Single NIC is not configured with a static IP address"
- これは問題なし (External NICは意図的にDHCP)

---

### 3. DNS構築

**確認コマンド**:
```batch
nslookup dev.local
ping dc01.dev.local
```

---

### 4. DHCP構築

**Scope設定**:
- IP Range: 192.168.50.100 - 192.168.50.200
- Subnet: 255.255.255.0
- Router: 192.168.50.1
- DNS: 192.168.50.1
- Domain: DEV.LOCAL

**結果**: DEV01は自動的に 192.168.50.102 を取得
- Gateway: 192.168.50.1
- DNS: 192.168.50.1
- DNS Suffix: DEV.LOCAL

---

### 5. RRAS / NAT構成

DC01をGatewayとして設定:

```powershell
# Remote Access → Routing → RRAS
# External Interface: Ethernet 2 (Internet側)
```

**動作確認**:
```batch
ping 192.168.50.1      # 成功
ping 8.8.8.8           # 成功
nslookup google.com    # 成功
```

---

### 6. SQL01構築 & 7. Domain Join

**Static IP設定**:
- IP: 192.168.50.20
- Gateway: 192.168.50.1
- DNS: 192.168.50.1

**Domain Join確認**:
```batch
whoami                 # → dev\administrator
systeminfo | findstr /B /C:"Domain"  # Domain: DEV.LOCAL
```

---

### 8. SQL Server Expressインストール

- Instance: SQLEXPRESS
- Authentication: Windows Authentication

**Service確認**:
```batch
sc query MSSQL$SQLEXPRESS  # State: RUNNING
```

---

### 9. SQL Server TCP/IP設定

デフォルトでDynamic Port (例: 49732) → 固定Port 1433に変更

```powershell
# SQL Server Configuration Manager
# TCP Dynamic Ports: (空欄)
# TCP Port: 1433
```

SQL Server再起動後、`SQL01,1433`で接続可能

---

### 10. SQL Server Firewall設定

```powershell
# Windows Firewall
# ルール: 受信 TCP 1433
# 許可先: 192.168.50.0/24 (Lab Network)
```

---

### 11. DEV01構築 & 12. Domain Join

DHCPで自動取得:
- IP: 192.168.50.102
- ログインユーザー: DEV\developer01

---

### 13. AD User/Group作成

**AD User**:
- Full Name: Developer User
- Logon name: developer01

**AD Group**:
- Group名: SQLDevelopers
- メンバー: developer01

---

### 14. SQL ServerとAD Group連携 (核心工程)

```sql
-- SQL Server Management Studio (SSMS)
-- DC01で作成したSQLDevelopersグループをLoginとして登録
-- Security → Logins → DEV\SQLDevelopers
```

**完成したアイデンティティフロー**:
```
AD
├── SQLDevelopers
│       └── developer01
│
└── (DC01)
         ↓
SQL Server
└── Security
    └── Logins
        └── DEV\SQLDevelopers
```

---

## トラブルシューティング

### Troubleshooting ① Gen2 VMが起動しなかった

**問題**: Generation 2でISOからBootエラー
**解決**: Generation 1に変更
**学び**: Gen1/Gen2は「正しい」ではなく、OS・ISO・UEFIの互換性を確認

### Troubleshooting ② DHCP / Network設定

**重要ポイント**: Internal NICにはGatewayを設定しない
- Internal NICのGateway = 空 (DC01のExternal NIC側がInternetゲートウェイ)
- 内部端末から見るDC01 = 192.168.50.1

### Troubleshooting ③ RRAS / NAT

Interface設定がうまく表示されなかった → External = Ethernet 2 としてInternet側設定

### Troubleshooting ④ SQL Server Dynamic Port

```batch
# 最初のポート: 49732 (Dynamic)
# 変更後: 1433 (Static)
# 確認: telnet 192.168.50.20 1433
```

### Troubleshooting ⑤ SQL Server Firewall

```powershell
Test-NetConnection 192.168.50.20 -Port 1433
# TcpTestSucceeded : True  (ネットワーク到達可能)
```

### Troubleshooting ⑥ SQL Server Login作成権限

**問題**: developer01でSSMSからLogin作成エラー
**原因**: 通常ユーザーはSQL Server権限なし
**解決**: DEV\Administratorで接続 → Security → Loginsへ作成

**学び**: Windows Administrator権限 ≠ SQL Server権限

### Troubleshooting ⑦ SSMSのCertificateエラー

`Trust server certificate`を有効化 → 接続成功
(ローカル学習環境なので許容)

### Troubleshooting ⑧ SQL Server Browser

**状態**: Stopped
**結論**: 固定ポート (1433) を使用するため不要

### Troubleshooting ⑨ Hyper-V VM画面が表示されない

```powershell
# vmconnect.exe が固まっている可能性
# Task Managerで確認
# 必要時は vmms サービス再起動
```

---

## 学びのまとめ

### 本番レベルの学び

本Labは「Windows Server操作」だけでなく、**複数コンポーネントの連携**を体験できました。

| コンポーネント | 役割 |
|--------------|------|
| AD | Identity/Group Management |
| DNS | Name Resolution |
| DHCP | IP Configuration (自動化) |
| RRAS | Routing/NAT (Internet接続) |
| Firewall | Traffic Control |
| SQL Server | Database Service |
| SSMS | Management Client |
| AD Group | SQL Server Login (集中管理) |

### 重要ポイント

1. **「ADにユーザーを作った」だけではSQL Serverには入れない**
2. **AD GroupをSQL Server Loginとして登録**することで、ADのグループメンバーシップをDBの認証・権限管理につなげられる

### 完成フロー

```
developer01 → AD認証 → DEV\SQLDevelopers → SQL Server Login → SQL01
```

---

## 参考コマンド集

```batch
REM AD確認
whoami /groups

REM Network確認
ping 192.168.50.1
Test-NetConnection 192.168.50.20 -Port 1433

REM DNS確認
nslookup dc01.dev.local
nslookup google.com

REM SQL確認
sc query MSSQL$SQLEXPRESS
```

---

**作成日**: 2026年10月  
**環境**: Windows Server 2025, SQL Server Express 2025, Hyper-V on Windows 10