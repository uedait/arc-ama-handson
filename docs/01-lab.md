# Step0：検証用ラボの構築

[トップ](../README.md)

この章は、Step1 と Step2 で使用する Azure VM、オンプレミス Windows Server、Proxy、DNS とネットワークがそろっている場合には省略できます。
オンプレミス環境がない場合など、Azure 上に疑似的なオンプレミス環境を用意してハンズオンを進めたい場合にご利用ください。

## 概要
スクリプトで、3つの VNet、Azure Firewall Standard、Azure Bastion Standard、DNS Private Resolver、3台の Azure VM、ディスク、計5個の Public IP（Firewall・Bastion・各 VM）などを自動作成します。具体的な構成は「1. スクリプトで自動構築される内容」を参照してください。
3台のうち1台は、Step1 でログ収集の対象にする Azure VM です。残り2台でオンプレミス環境を模します。1台は Windows DNS サーバー、もう1台は Nested Hyper-V ホスト（L1）です。L1 内に Windows Server（L2）を作成し、Step2 で L2 を Azure Arc に登録して Azure Monitor でログを収集します。

**実行すると費用が発生します。**
2026年11月4日の[料金計算ツール](https://azure.microsoft.com/ja-jp/pricing/calculator/)で、東日本、日本円で試算した1日（24時間）あたりの参考費用は以下のとおりです。

| 課金対象 | 3台の VM を起動中 | 3台とも割り当て解除後 |
|---|---:|---:|
| Windows D2s v5 ×2台 | 1,641円 | 0円 |
| Windows D8s v5 ×1台 | 3,281円 | 0円 |
| Firewall Standard ×1 | 4,747円 | 4,747円 |
| Bastion Standard、基本の2インスタンス | 1,101円 | 1,101円 |
| DNS Private Resolver、Inbound ×1 | 936円 | 936円 |
| Standard SSD LRS、E10 ×3枚 | 150円 | 150円 |
| Standard 静的 Public IPv4 ×5個 | 95円 | 95円 |
| 合計 | 11,951円 | 7,029円 |

VM の割り当て解除後も Bastion、Firewall、Resolver、Public IP、ディスク等の課金は続きます。参考費用は実施時の料金と利用期間に合わせて確認してください。
なお、既定では、VM サイズ `Standard_D8s_v5` を使用しています。
スクリプト実行前に、リージョンの提供状況、Gen2・Nested Virtualization 対応、容量とクォータも別途確認してください。

## 1. スクリプトで自動構築される内容

| ネットワーク | 対象 | 設定 |
|---|---|---|
| spoke1 `10.10.0.0/16` | workload VM | Windows Server 2022、`10.10.1.4`、既定 `Standard_D2s_v5` |
| hub `10.20.0.0/16` | Azure Firewall Standard と Policy | AzureFirewallSubnet `10.20.0.0/26`、専用 Public IP |
| 同上 | DNS Private Resolver Inbound | subnet `10.20.2.0/28`、IP `10.20.2.4` |
| 同上 | Bastion Standard | AzureBastionSubnet `10.20.3.0/26`、2インスタンス、専用 Public IP、ネイティブクライアント対応 |
| onpre `10.30.0.0/16` | Windows DNS VM | `10.30.1.5`、既定 `Standard_D2s_v5`、既定フォワーダー `168.63.129.16` |
| 同上 | L1 Hyper-V ホスト | `10.30.1.4`、既定 `Standard_D8s_v5`、Standard セキュリティ |
| L1 内部 | L2 Windows Server | 公式 ISO の WIM から Gen2 親・差分 VHDX、2 vCPU・4 GB、無人 specialize / OOBE |
| L2 のネットワーク | Internal vSwitch と WinNAT | IP `192.168.100.10/24`、GW `192.168.100.1`、DNS `10.30.1.5` |

3台の Azure VM には、それぞれ専用の Standard 静的 PIP と NIC NSG を関連付けます。RDP は Bastion subnet から許可し、それ以外からの RDP と Internet からの着信を遮断します。構築中の外向き通信には各 VM の PIP を使用します。

Day1 の Step1 では LAW、DCR、Azure VM の AMA、ログ検索アラートとメール通知を設定します。
Day2 の Step2 では Explicit Proxy と通信許可、Arc 登録、L2 の AMA を設定し、Day1 の LAW とアラート規則でログ収集と通知を試します。

Firewall の Explicit Proxy と通信許可は Step2 で設定します。Step0 の完了時点では、各 VM の Internet 向け送信が制限された状態になります。

## 2. 一括デプロイを実行する

### 管理端末に資料を取得する

Windows の管理端末に PowerShell 7、Git、Azure CLI、GitHub CLI と Azure CLI から使用できる Bicep を用意してください。
管理端末から Azure、`github.com`、GitHub API と `codeload.github.com` へ HTTPS 接続できる必要があります。
また、Azure のデプロイに必要な権限を用意してください。

PowerShell 7 を開き、資料を保存するフォルダーへ移動してから、次を実行します。

```powershell
git clone https://github.com/uedait/arc-ama-handson.git
Set-Location .\arc-ama-handson
```

Git を使用しない場合は、[GitHub のリポジトリ](https://github.com/uedait/arc-ama-handson)で「Code → Download ZIP」を選び、ZIP を展開してください。Private リポジトリで実施する場合は、資料取得にも閲覧権限のある GitHub アカウントが必要です。

すでに資料を取得している場合は、取得済みの `arc-ama-handson` フォルダーへ移動してください。以降の管理端末用コマンドは、`README.md` と `infra` フォルダーがあるリポジトリのルートで実行します。

構築スクリプトは GitHub に公開済みのコミットから構築資材を取得します。ローカルの変更を検証環境へ適用する場合は、先にその変更を新リポジトリへ公開してください。

### 自動構築で使用する GitHub アカウントを認証する

管理端末で次を実行してください。GitHub CLI が認証済みの場合は省略できます。

```powershell
gh auth login --hostname github.com --web --git-protocol https
```

GitHub と Azure の認証は別です。GitHub のパスワードやトークンをスクリプトへ入力したり、ファイルに記載したりする必要はありません。

### 利用者のアカウントで実行する

対象の Azure テナントで対話ログインしてから、一括デプロイを実行します。
証明書で認証したサービスプリンシパルを使う場合は、次の節の実行例を使用してください。

```powershell
az login --tenant '<Tenant ID>'
.\infra\lab\Deploy-Lab.ps1
```

### 証明書で認証したサービスプリンシパルを使う場合

実施環境の認証手順で証明書による Azure CLI ログインと接続先の確認を済ませ、同じ PowerShell プロセスで次を実行します。

```powershell
$env:AZURE_CONFIG_DIR = '<事前認証した専用キャッシュの絶対パス>'
.\infra\lab\Deploy-Lab.ps1 `
    -TenantId '<Tenant ID>' `
    -SubscriptionId '<Subscription ID>' `
    -ClientId '<サービスプリンシパルのクライアント ID>'
```

### 構築内容を確認する

対話では次の値を確認します。

| 項目 | 既定値・方針 |
|---|---|
| サブスクリプション | 利用者ログインでは選択中の ID を既定として確認。サービスプリンシパルでは引数で指定した ID を照合 |
| RG | `rg-arclab-lab`。新規または空の専用 RG |
| 名前の接頭辞 | `arclab`。VM 名は `-workload`、`-dns`、`-hyperv` を付ける |
| Azure VM の管理者 | 既定ユーザー名 `labadmin`。3台に使うパスワードを非表示で入力 |
| L2 の管理者 | `Administrator` のパスワードを非表示で入力 |
| 実行承認 | 対象、コミット、課金対象、Nested 対応・クォータ・予算・期限を確認して `APPROVE` |

ISO の URL・SHA-256・利用条件の確認を追加のヒアリング項目には含めません。既定の公式配布物から取得し、SHA-256 は取得ファイルの識別とキャッシュ照合に使用します。公開元の既知ハッシュとの照合とは区別してください。

既知の値を引数で渡す場合は、次のように実行できます。パスワードは対話で入力してください。

```powershell
.\infra\lab\Deploy-Lab.ps1 `
    -SubscriptionId '<Subscription ID>' `
    -ResourceGroupName 'rg-arclab-lab' `
    -Prefix 'arclab' `
    -Location 'japaneast' `
    -SourceRef '<使用する完全なコミット SHA>'
```

実行中は段階名と状態を CLI に表示します。VM 内でスクリプトを手動実行する操作や、L1 の再起動ボタンの操作は、自動構築の通常手順には含めません。

## 3. 完了結果と Bastion での確認

1. Portal で対象 RG と各 VM の Private IP を確認します。
2. 各 VM の「接続 → Bastion」から `labadmin` と入力したパスワードで接続してください。
3. workload、DNS、L1 の3台でホスト名と構成を確認します。
4. L1 の Hyper-V Manager / VMConnect で L2 を開き、Administrator の入力したパスワードで接続します。
5. 3台の NIC NSG の `LabInternetOutbound` が `Deny` であることと、L2 の `C:\Handson` に `Install-ArcAgent.ps1` と `Write-HandsonEvent.ps1` が配置されていることを確認してください。

通常の VM 操作はブラウザーの Bastion 接続で行えます。

参考：[Managed Run Command](https://learn.microsoft.com/azure/virtual-machines/windows/run-command-managed)、[Bastion と VNet Peering](https://learn.microsoft.com/azure/bastion/vnet-peering)、[PowerShell Direct](https://learn.microsoft.com/virtualization/hyper-v-on-windows/user-guide/powershell-direct)。
