# なんちゃって Hydration ＝ Azure Local VM 登録ツール

バックアップツールなどで Azure Local に移行した Hyper-V クラスター VM を、**Azure Local VM（Azure Local VM enabled by Azure Arc）として Azure に登録する PowerShell ベースのツール**です。

> **検証環境専用です。** 公開プレビュー API（`createFromLocal`）を使い、あくまでも Hydration 検証用にサンプルで作ったものです。
> 処理を実行するにはクラスターの管理者権限と仮想マシン内の OS に対する管理権限が必要です。
> 正式な移行手順ではありませんし、動作確認は単一ラボのみで、本番利用は想定していません。
> テスト環境においても、実行前に VM のバックアップを取り、他の変更が入らない保守時間帯に実行してください。

## 0. 概要

### 本ツールが内部的に実施している 3 つの処理

`tools\Invoke-AzureLocalVmSetup.ps1` 実行時に、次の3つの処理を続けて実行します。

| 処理 | 内容 | 個別に実行する場合 |
| --- | --- | --- |
| **登録** | 既存の Hyper-V VM を、Azure 上の新しい親リソース（`Microsoft.HybridCompute/machines`）と VM インスタンスとして登録する。既存の Azure リソースは採用も上書きもせず拒否する。 | `tools\Register-AzureLocalVm.ps1` |
| **NIC 置換** | Azure API を使って NIC を追加する。元の Hyper-V NIC の IP 設定を退避し、Azure 管理 NIC に移す。通信確認後、確認を得て元の NIC だけを削除する。通信が短時間途切れるので注意。 | `tools\Replace-AzureLocalVmNic.ps1` |
| **ゲスト Arc 接続** | VM 内に構成エージェントを導入し、Azure から VM のゲストを管理できるようにする。 | `tools\Complete-AzureLocalVm.ps1` |

### 実際の作業手順

| 手順 | 内容 | 
| --- | --- |
| 1 | 条件の確認 |
| 2 | ツール(ファイル)のダウンロードと展開 |
| 3 | 設定ファイルの編集 |
| 4 | 処理の実行（登録 → NIC 置換 → ゲスト Arc 接続） |

実行後の確認と失敗時の対応は補足１、対応範囲と制限は補足２にあります。

## 1. 条件の確認

実行前に、次の条件を満たしているか確認してください。

| 項目 | 必要条件 |
| --- | --- |
| 管理端末 | Windows PowerShell 5.1 または PowerShell 7.4 以上、Az.Accounts 5 以上、Azure サインイン済み |
| 対象 VM | クラスター上で Running の第2世代 VM。PowerShell Direct に対応する 64 ビット Windows。**差分ディスクでない VM**（第7節） |
| 接続 | 管理端末 → VM 所有ノードへの WinRM。ノード → ゲストへの PowerShell Direct |
| 権限 | Azure リソースの読み取り／一覧／作成／更新、所有ノードの Hyper-V／FailoverClusters 操作、ゲスト管理者 |
| Azure | AzureCloud。対象リソースはすべて同一サブスクリプション。既存の Azure Local、custom location、logical network |
| ネットワーク | 元 NIC と同じ switch／VLAN／IPv4 prefix の、単一 subnet・Static の logical network。割り当て可能な pool 内 IPv4 |
| ゲストの通信 | DNS と必要な Azure エンドポイントへの外向き通信（NIC 置換の検査は `management.azure.com` への直接 TCP 443） |
| 保存先 | バックアップ／結果レポート用の既存フォルダー（例: `C:\ProtectedBackups`）。ACL は運用者が管理 |
| 事前準備 | VM のバックアップ取得、他の変更が入らない保守時間帯の確保（NIC 置換で短時間通信が途切れます） |

依存モジュールの導入、サインイン、TrustedHosts／実行ポリシーの変更はツールでは行いません。

## 2. ツールファイルのコピーと展開

配布 ZIP を `C:\Tools\AzureLocalVmToolkit` などへ展開します。`tools`、`examples`、`tests` を同じ構成で置く必要があり、スクリプト単体のコピーでは動きません。

```powershell
Expand-Archive -LiteralPath 'C:\Packages\AzureLocalVmToolkit-final.zip' `
    -DestinationPath 'C:\Tools\AzureLocalVmToolkit' -Force
```

更新時も ZIP 内のファイルを一式上書きし、**新しい PowerShell ウィンドウ**で作業してください。ZIP には設定ファイルやバックアップは含まれないため、それらは上書きされません。

## 3. 設定ファイルの編集

テンプレートファイル `examples\registration.example.json` を `registration.lab.json` にコピーして編集します。例の値は架空です。**パスワード、トークン、SAS は書かないでください**（未定義の項目は拒否されます）。

```powershell
Set-Location C:\Tools\AzureLocalVmToolkit
Copy-Item .\examples\registration.example.json .\registration.lab.json
notepad.exe .\registration.lab.json
```

| 項目 | 内容 |
| --- | --- |
| `TenantId`／`SubscriptionId` | 対象の Azure 環境 |
| `ResourceGroup`／`MachineName`／`Location` | **新規**に作る Azure 親リソースの RG／名前／リージョン（例: `japaneast`） |
| `ClusterId`／`CustomLocationId` | 既存 Azure Local と custom location のリソース ID |
| `LogicalNetworkId` | NIC 置換で使う、既存 logical network の完全なリソース ID |
| `LocalVmName` | Hyper-V 上の VM 名。大文字小文字は区別しません。クラスター内で一意であることが前提 |
| `ExpectedVmId`（任意） | `Get-VM` の `Id`。省略すると、所有ノードで名前が一致した VM の UUID を自動採用します。指定すると照合し、不一致なら停止します。VM を作り直した場合は更新するか行ごと削除してください。 |
| `OsType` | `Windows` |
| `StackHciApiVersion`（任意） | 既定は `2026-04-01-preview`。`2026-07-01-preview` も選べますが、リージョンに無いと `NoRegisteredProviderFound` で停止します。自動の切り替えはしません。 |
| `NetworkInterfaceIds`（任意） | 既存の Azure NIC を参照する場合の ID 一覧。この統合手順では使わないため、`[]` のままにします（元の Hyper-V NIC の ID は書きません）。 |
| `DataDiskIds`（任意） | 対象 VM に対応する既存 Azure ディスクの ID 一覧。不要なら `[]` のままにします。 |
| `LocalHost`（任意） | 通常は書きません。VM の所有ノードはフェールオーバークラスターから自動取得します。 |

`_comment_` で始まる項目（`_comment_ExpectedVmId`、`_comment_NetworkInterfaceIds`、`_comment_DataDiskIds`）は説明用で、ツールは無視します。

## 4. 処理の実行

### 進め方

どの処理でも、次の順で進めます。

1. **プラン表示**: 引数だけ指定して実行。接続も書き込みもせず、対象と影響を表示します。
2. **`-WhatIf`**: `-Execute -EnableUnsupportedLabPreview -WhatIf`。認証・読み取り・書き込みはしません。
3. **実行**: `-Execute -EnableUnsupportedLabPreview -Confirm`。対象と影響を表示して確認を求めます。

書き込みには `-Execute` と `-EnableUnsupportedLabPreview` の**両方**が必要です。`-Confirm:$false` は確認を省略するため、通常は使わないでください。ツールは VM の停止／起動／再起動、ディスクコピー、Azure リソースの削除、自動ロールバックを行いません。

### 実行例（登録 → NIC 置換 → ゲスト Arc 接続）

次を作業用 `.ps1` に保存して実行すると、対話入力と貼り付けが混ざりません（5.1 で日本語を含めるなら UTF-8 BOM 付き）。資格情報はコンソールで入力し、ファイルには書きません。ホスト用は所有ノードに接続するアカウント、ゲスト用は VM 内の Windows 管理者です。

```powershell
Set-Location C:\Tools\AzureLocalVmToolkit
$config = Get-Content -LiteralPath .\registration.lab.json -Raw | ConvertFrom-Json
$session = $null; $hostCredential = $null; $guestCredential = $null
function Read-ConsoleCredential([string]$Purpose) {
    $name = Read-Host -Prompt "$Purpose - user name (DOMAIN\user or user)" -ErrorAction Stop
    if ([string]::IsNullOrWhiteSpace($name)) { throw 'User name was not entered.' }
    $password = Read-Host -Prompt "$Purpose - password" -AsSecureString -ErrorAction Stop
    if ($null -eq $password -or $password.Length -eq 0) { throw 'Password was not entered.' }
    [pscredential]::new($name, $password)
}
try {
    Connect-AzAccount -Tenant $config.TenantId -Subscription $config.SubscriptionId -ErrorAction Stop
    Set-AzContext -Tenant $config.TenantId -Subscription $config.SubscriptionId -ErrorAction Stop

    # クラスターに問い合わせ、VM の現在の所有ノードへ接続する
    $hostCredential = Read-ConsoleCredential 'Cluster / VM owner node'
    Import-Module .\tools\AzureLocalVmRegistration.psm1 -Force
    $configTable = @{}
    foreach ($p in $config.PSObject.Properties) { $configTable[$p.Name] = $p.Value }
    $session = Connect-AzureLocalVmOwnerNode -Configuration $configTable -Credential $hostCredential
    "Owner node: $($session.ComputerName)"

    $guestCredential = Read-ConsoleCredential 'Windows guest'
    $stamp = Get-Date -Format 'yyyyMMdd-HHmmss-fff'
    $run = @{
        ConfigurationPath             = '.\registration.lab.json'
        ReplaceExistingNetworkAdapter = $true
        NetworkInterfaceName          = "$($config.MachineName)-nic0"
        BackupPath                    = "C:\ProtectedBackups\$($config.MachineName)-$stamp.nic.json"
        ReportPath                    = "C:\ProtectedBackups\$($config.MachineName)-$stamp.setup.json"
        ConvertDhcpToStatic           = $true                    # 元 NIC が DHCP のとき、静的 IP 化を承認
        # PrivateIPAddress            = '192.0.2.30'             # IP を変える場合だけ、pool 内の未使用 IP
        TimeoutSeconds                = 900
    }

    # 1) プラン表示  2) WhatIf  3) 実行
    .\tools\Invoke-AzureLocalVmSetup.ps1 @run
    .\tools\Invoke-AzureLocalVmSetup.ps1 @run `
        -Execute -EnableUnsupportedLabPreview -AcceptNetworkInterruption -WhatIf
    .\tools\Invoke-AzureLocalVmSetup.ps1 @run `
        -OwnerNodeSession $session -GuestCredential $guestCredential `
        -Execute -EnableUnsupportedLabPreview -AcceptNetworkInterruption -Confirm -ErrorAction Stop
} finally {
    if ($null -ne $session) { Remove-PSSession -Session $session }
    if ($null -ne $hostCredential) { $hostCredential.Password.Dispose() }
    if ($null -ne $guestCredential) { $guestCredential.Password.Dispose() }
}
```

- **`BackupPath`／`ReportPath`** は既存フォルダー内の**新しいファイル名**にします。既存ファイルは上書きしません。
- **元 NIC（`SourceNetworkAdapterId`）**: VM の Hyper-V NIC が1枚なら自動選択します。0枚／複数枚なら書き込み前に停止します。複数枚の場合だけ、`Get-VMNetworkAdapter` の `Id`（`Microsoft:<VM UUID>\<NIC UUID>`）を指定します。
- **IP**: 元の IP を使うなら `PrivateIPAddress` は省略します。元の IP が logical network の pool 外なら停止するため、変更を承認する場合だけ pool 内の未使用 IP を指定します。
- **内部の処理順**: 事前検査（Azure、所有ノード、ゲスト、元 NIC／IP／pool）→ 登録 → NIC 置換（バックアップ作成後に Azure 管理 NIC を接続し、IP を移して通信確認、確認後に元 NIC を削除）→ ゲスト Arc 接続（構成エージェント導入と Arc 接続）。

### (参考): 処理を個別に実行する／NIC 置換を省く

通常は不要です。

```powershell
# 登録のみ
.\tools\Register-AzureLocalVm.ps1 -ConfigurationPath .\registration.lab.json
# NIC 置換のみ（登録後）
.\tools\Replace-AzureLocalVmNic.ps1 -ConfigurationPath .\registration.lab.json
# ゲスト Arc 接続のみ（登録後）
.\tools\Complete-AzureLocalVm.ps1 -ConfigurationPath .\registration.lab.json -EnableGuestManagement
```

流れは同じ（プラン表示 → `-WhatIf` → `-Execute`）で、引数は `Get-Help .\tools\<スクリプト名>.ps1 -Full` で確認できます。
NIC 置換を省いて登録 → ゲスト Arc 接続だけ行う場合は、`$run` から `ReplaceExistingNetworkAdapter`、`NetworkInterfaceName`、`BackupPath`、`ConvertDhcpToStatic` と `-AcceptNetworkInterruption` を外します。既存 NIC と IP／DNS は変更されませんが、この構成の実機動作は未検証です。

## 補足１. 結果の確認と失敗時の対応

### 成功の目安

| 確認項目 | 期待値 |
| --- | --- |
| 全体 | `Status: PreviewSetupSucceeded` |
| 登録 | `Registration.Status: PreviewProvisioningSucceeded`。Hyper-V の UUID と一致 |
| NIC 置換 | `NicReplacement.Status: PreviewNicReplacementSucceeded`、`OriginalRemoved: true` |
| ゲスト Arc 接続 | `Completion.GuestArcConnected: true` |

実体（Azure／Hyper-V／ゲストの MAC と IP）も目視で確認してください。`ReportPath` には各段階の結果が保存されます。

### 失敗したとき

**登録・NIC 作成・置換を、確認せずに再実行しないでください。** 失敗時は、終了エラーとレポートに「どの段階まで進んだか」と手動対処が出ます。ツールは自動でリトライ、ロールバック、Azure リソースの削除をしません。

| 状況 | 対応 |
| --- | --- |
| 登録が失敗 | Azure 上の親リソース／VM インスタンスの状態を確認。残った失敗リソースは手動で削除してから再実行（既存リソースは拒否される）。VM を作り直した場合は `ExpectedVmId` も更新 |
| `NoRegisteredProviderFound` | `StackHciApiVersion` を、リージョンで提供される版（`2026-04-01-preview` など）に変更 |
| `no potential OS disks`（OS ディスクが見つからない） | 差分ディスクの VM で発生した事例あり。差分ディスクでない VM で再試行（第7節） |
| NIC 置換: NIC の作成がタイムアウト | Azure 側の実体を確認。未接続／失敗の NIC は自動では採用しません |
| NIC 置換: 元 NIC の削除を拒否 | 新 NIC へは切り替え済みで、元 NIC は切断されたまま残る。統合処理はここで止まり、ゲスト Arc 接続には進まない |
| NIC 置換は成功、ゲスト Arc 接続だけ失敗 | NIC 置換を繰り返さず、下の再開コマンドを使う |

**ゲスト Arc 接続だけを再開する**（`ReplaceExistingNetworkAdapter` と NIC 関連の引数は付けません。`ReportPath` は新しい名前）:

```powershell
.\tools\Invoke-AzureLocalVmSetup.ps1 `
    -ConfigurationPath .\registration.lab.json -RegistrationAlreadyComplete `
    -OwnerNodeSession $session -GuestCredential $guestCredential `
    -ReportPath 'C:\ProtectedBackups\guest-resume-001.json' `
    -Execute -EnableUnsupportedLabPreview -Confirm -ErrorAction Stop
```

`-RegistrationAlreadyComplete` は登録だけを省略します。既存リソースを上書きする指定ではありません。

元 NIC が残っていれば、保存したバックアップの設定を手動で復元できます。削除後は同じ名前や MAC で作り直しても元の GUID には戻らないため、VM のバックアップから復旧してください。

## 補足２. 対応範囲と制限

**対象外**: 移行、ディスクコピー、一般化、VM の停止／起動、Azure リソースの削除、自動ロールバック、Azure パブリッククラウドへの移行。

**対応しない VM／構成（書き込み前に拒否）**:

- 複数の IP、追加ルート、IPv6、LBFO チーミング、trunk VLAN、レガシー NIC、元 NIC の ACL／spoofing／ミラーリング等
- 既存の Azure 親リソース／VM インスタンス（採用も更新もしない）
- Running でない VM、第2世代以外
- 異なるサブスクリプションのリソース、AzureCloud 以外

**差分ディスクの VM では失敗した事例があります。** 差分ディスクの VM では登録が `no potential OS disks` で失敗し、差分ディスクでない VM に作り直すと成功しました。ツールはこの構成を事前には検出しないため、登録前に自分で確認してください。

```powershell
Get-VMHardDiskDrive -VMName <VM名> | ForEach-Object { Get-VHD -Path $_.Path } | Select-Object Path, VhdType, ParentPath
```

`VhdType` が `Differencing` の場合は使わないでください。原因を差分ディスクと断定できる材料は、作り直して成功したという結果のみです。

**制限**:

- 公開プレビュー API を使うため、製品サポートはなく、サーバー側の動作は変わる可能性があります。
- NIC 置換では元の MAC は引き継がれません。Azure が MAC を割り当てるため、NIC の ID とゲストの InterfaceGuid が変わります。DHCP の予約は移行されません。
- 通信が短時間途切れます。アプリが NIC GUID／IP に固定バインドしている場合は別途確認が必要です。
- 必要な RBAC の最小セットは未確認です。

## 補足３. 検証状況とテスト

- **オフライン検証**: モックによるテスト計 215 件が、Windows PowerShell 5.1 と PowerShell 7 で通っています。
- **実環境**: ラボの Windows VM（差分ディスクでない通常の VM）で、登録・NIC 置換・ゲスト Arc 接続（`2026-04-01-preview`）の成功を確認しています。停止／起動／再起動は行っていません。
- **未検証**: `2026-07-01-preview`、他の Azure Local バージョン、NIC 置換なし（登録 → ゲスト Arc 接続）の実機、Windows Server 以外の OS、停止／起動などのライフサイクル操作。所有ノードの自動取得、VM 名からの UUID 自動取得、VM 名の大文字小文字の無視は、オフラインのモックでの検証です。

```powershell
pwsh -NoProfile -File .\tests\AzureLocalVmRegistration.Tests.ps1
pwsh -NoProfile -File .\tests\AzureLocalVmCompletion.Tests.ps1
pwsh -NoProfile -File .\tests\AzureLocalVmNicReplacement.Tests.ps1
pwsh -NoProfile -File .\tests\AzureLocalVmSetup.Tests.ps1
```
