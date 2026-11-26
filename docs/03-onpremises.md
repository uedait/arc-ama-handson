# Day2／Step2：オンプレミス VM のログ収集とメール通知

[README](../README.md) | [Step0：検証環境の構築](01-lab.md) | [Day1／Step1：Azure VM](02-azure.md)

オンプレミスの Windows Server を Azure Arc に登録し、Azure Monitor Agent（AMA）で Windows イベントを収集します。Connected Machine Agent（Arc Agent）と AMA は、明示 Proxy を介して Azure の公開エンドポイントへ接続します。

収集先には Day1 の Log Analytics ワークスペース（LAW）を使用します。同じ Data Collection Rule（DCR）にオンプレミス VM を追加し、Day1 のアラート規則とメール通知も試します。

## 始める前に

- [Day1 の手順](02-azure.md)で LAW、Windows イベント収集用 DCR、ログ検索アラート、アクショングループを作成してください。
- 対象 Windows Server を操作する管理者アカウントと、Azure Arc の登録、拡張機能の導入、DCR の関連付けができる Azure アカウントを用意してください。ログ検索には LAW の閲覧権限も必要です。
- 明示 Proxy は、認証不要の HTTP Proxy と HTTPS CONNECT に対応したものを使用します。Proxy 側でも、Azure の通信先を名前解決し、TCP 443 で接続できる必要があります。
- 既存のサーバーや Proxy を使う場合は、Arc、AMA、Proxy、通信許可の設定を確認してから進めてください。

この手順は Azure Portal を中心に進めます。Arc の登録には Portal が生成するスクリプト、AMA の導入には DCR のリソース追加を使用します。画面の項目名は今後変更になる可能性があります。

### 入力例

Day1 のリソースとオンプレミス VM を使用する入力例です。お客様の環境では、作成済みのリソース名、対象 VM の名前や Proxy の値に置き換えてください。

| 対象 | 例 |
|---|---|
| サブスクリプション / RG | Day1 と同じサブスクリプション / `rg-arclab-lab` |
| リージョン | Japan East。コマンドでは `japaneast` |
| オンプレミス VM | `onprem-vm01` / `192.168.100.10` |
| Windows DNS | `10.30.1.5` |
| Firewall / Firewall Policy | `arclab-fw` / `arclab-fw-policy` |
| Proxy の IP / 待受ポート | `10.20.0.4` / `3128` |
| Proxy URL | `http://10.20.0.4:3128` |
| LAW / DCR | `law-arclab` / `dcr-arclab-windows` |
| アラート規則 / アクショングループ | `alert-arclab-event9001` / `ag-arclab-mail` |

Arc の登録とイベント生成は、対象のオンプレミス VM で行います。

Step0 の検証ラボを使う場合は、Hyper-V ホスト `arclab-hyperv` に Bastion で接続し、Hyper-V Manager / VMConnect からオンプレミス環境を模した VM `ArcOnpremL2` を操作します。以下の例の `onprem-vm01` は、このラボでは `ArcOnpremL2` に置き換えてください。

## 3-1. 明示 Proxy と通信許可を設定する

### Azure Firewall の Explicit Proxy

Step0 のラボでは Azure Firewall Standard を使用します。既存 Proxy を使う場合も、次の通信先を許可し、対象サーバーから Proxy へ接続できるように設定してください。

1. Portal で対象 Firewall を開き、「概要」でプライベート IP を確認します。関連付いた Firewall Policy のリンクを開きます。
2. Policy の「Explicit Proxy / 明示的なプロキシ」を開き、有効にします。HTTP 待受ポートを `3128` にして保存します。HTTP と HTTPS はこの待受ポートを使用します。
3. 「規則コレクション」から Application 規則コレクションを追加します。

   | 項目 | 入力例 |
   |---|---|
   | 名前 | `arc-ama-outbound` |
   | 規則コレクションの種類 | Application |
   | 規則コレクション グループ | `DefaultApplicationRuleCollectionGroup` |
   | 優先度 | `200`。既存コレクションと重複しない値 |
   | アクション | Allow |

4. コレクション内に、次の規則を追加します。プロトコルはすべて `HTTPS:443`、宛先の種類は FQDN です。

   | 規則名 | 対象 FQDN |
   |---|---|
   | `arc-auth` | `login.microsoftonline.com`、`*.login.microsoft.com`、`pas.windows.net` |
   | `arc-arm` | `management.azure.com` |
   | `arc-services` | `*.his.arc.azure.com`、`*.guestconfiguration.azure.com` |
   | `arc-notifications` | `guestnotificationservice.azure.com`、`*.guestnotificationservice.azure.com`、`*.servicebus.windows.net` |
   | `agent-download` | `aka.ms`、`download.microsoft.com`、`gbl.his.arc.azure.com`、`*.blob.core.windows.net` |
   | `ama-control` | `global.handler.control.monitor.azure.com`、`japaneast.handler.control.monitor.azure.com` |
   | `ama-ingestion` | `<Workspace ID>.ods.opinsights.azure.com` |
   | `ama-service` | `global.prod.microsoftmetrics.com` |

   ソースの種類は「IP アドレス」にし、Proxy から見た対象のオンプレミス VM の送信元 IP を指定します。NAT を経由する場合は変換後の IP を使用してください。Step0 のラボでは Hyper-V ホストの WinNAT を通るため、ソースはホストの IP `10.30.1.4` です。

   `japaneast` は Arc の登録リージョン、`<Workspace ID>` は LAW の「概要」に表示される GUID に置き換えます。LAW の名前とは異なる値です。

   `agent-download` は初回導入と拡張機能の配布に使用する許可です。配布先はバージョンやリダイレクトによって変わる場合があります。取得に失敗した場合は、ダウンロードのエラーと Proxy で拒否された取得先を確認し、その宛先を追加してください。

5. Policy の「DNS」で、Azure の公開 FQDN を名前解決できる DNS サーバーを指定して保存します。Step0 のラボでは Resolver Inbound IP `10.20.2.4` を指定し、オンプレミス VM の DNS には Windows DNS `10.30.1.5` を使用します。
6. オンプレミス VM から Proxy の待受ポートまでの通信を確認します。Step0 のラボでは Hyper-V ホストの NIC NSG で、宛先 `10.20.0.4`、TCP `3128` を許可済みです。hub と onpre の VNet ピアリングを通じて接続します。

Arc と AMA の通信は、Azure の証明書をそのまま使用するように設定してください。HTTPS 検査や TLS 復号を行う Proxy では、この通信を検査対象から除外します。

### オンプレミス VM から Proxy 経由の HTTPS 接続を試す

対象サーバーの Windows PowerShell で次を実行します。

```powershell
$proxyIp = '10.20.0.4'
$proxyPort = 3128
$proxyUrl = "http://${proxyIp}:$proxyPort"
Test-NetConnection -ComputerName $proxyIp -Port $proxyPort
$response = Invoke-WebRequest -UseBasicParsing `
    -Uri 'https://management.azure.com/metadata/endpoints?api-version=2020-06-01' `
    -Proxy $proxyUrl -ErrorAction Stop
$response.StatusCode
```

確認してみよう。Proxy ポートへの接続は `TcpTestSucceeded = True`、HTTPS 要求は `200` になりましたか。ここでは Proxy 経由の HTTPS 接続を試しています。Arc の認証は次の節で行います。

## 3-2. Azure Portal のスクリプトで Arc に登録する

### オンボードスクリプトを作成する

1. 管理端末の Portal で「Azure Arc → インフラストラクチャ → マシン → オンボード / 作成 → 既存マシンのオンボード」を開きます。
2. 基本設定を入力します。

   | 項目 | 入力例 |
   |---|---|
   | サブスクリプション / RG | Day1 と同じ作成先 / `rg-arclab-lab` |
   | リージョン | Japan East |
   | オペレーティング システム | Windows |
   | 接続方法 | Public endpoint |
   | Proxy サーバー URL | `http://10.20.0.4:3128` |
   | 認証 | マシンを手動で認証 |

3. タグを設定する場合は、演習に使用する値を入力して次へ進みます。
4. 「スクリプトのダウンロードと実行」で設定を確認し、`OnboardingScript.ps1` をダウンロードします。
5. 対象のオンプレミス VM の作業フォルダーへ配置します。例は `C:\ArcOnboarding` です。

### ダウンロードと Agent の Proxy 設定を確認する

`OnboardingScript.ps1` をエディターで開きます。行番号は生成時期によって変わるため、ダウンロード、インストール、`connect` の処理を目印にしてください。

1. ダウンロードとインストールに Proxy が指定されていることを確認します。次は生成スクリプトの処理例です。ダウンロード先と変数名は、生成されたスクリプトの値を使用してください。

   ```powershell
   Invoke-WebRequest -UseBasicParsing `
       -Uri 'https://gbl.his.arc.azure.com/azcmagent-windows' `
       -TimeoutSec 30 -OutFile $installScriptPath `
       -Proxy 'http://10.20.0.4:3128'
   & $installScriptPath -proxy 'http://10.20.0.4:3128'
   ```

2. Agent のインストール処理が終わり、`azcmagent.exe connect` を実行する直前に、次を追加します。Proxy は Agent 固有の設定で指定します。

   ```powershell
   $arc = "$env:ProgramW6432\AzureConnectedMachineAgent\azcmagent.exe"
   & $arc config set proxy.url 'http://10.20.0.4:3128'
   if ($LASTEXITCODE -ne 0) { throw 'Arc proxy setup failed.' }
   & $arc config clear proxy.bypass
   if ($LASTEXITCODE -ne 0) { throw 'Arc proxy bypass setup failed.' }
   ```

   この設定で、Arc のクラウド向け通信を Proxy の対象にします。生成スクリプトに `HTTPS_PROXY` を `Machine` に保存する行がある場合は、その行の代わりに上記の Agent 固有の設定を使用してください。既存の OS 全体の Proxy 設定は維持します。

3. `azcmagent.exe connect` の行に `--use-device-code` を追加します。作成先や Proxy などの、生成済みの引数は保持します。この認証方法では、管理端末のブラウザーで Azure にサインインできます。

### オンプレミス VM で実行して接続を確認する

1. オンプレミス VM の64ビット Windows PowerShell を管理者として開き、配置したスクリプトを実行します。

   ```powershell
   Set-Location 'C:\ArcOnboarding'
   .\OnboardingScript.ps1
   ```

2. 管理端末のブラウザーで [https://login.microsoft.com/device](https://login.microsoft.com/device) を開き、スクリプトに表示されたデバイスコードを入力して、登録先の Azure アカウントで認証します。スクリプトに別の認証 URL が表示された場合は、その URL を使用してください。組織の条件付きアクセスでデバイスコード認証が制限されている場合は、管理者と使用できる認証方法を確認してください。
3. Portal の「Azure Arc → マシン」で、登録した `onprem-vm01` の状態が `Connected` になっていることを確認します。
4. 対象 Arc リソースを開き、サブスクリプション、RG、リージョンを確認します。「概要 → JSON ビュー」の `id` をコピーしてください。後のログ検索では、この Arc リソース ID を使用します。
5. オンプレミス VM の Windows PowerShell で次を実行します。登録したリージョンに合わせて `japaneast` を置き換えてください。

   ```powershell
   $arc = "$env:ProgramFiles\AzureConnectedMachineAgent\azcmagent.exe"
   & $arc show
   if ($LASTEXITCODE -ne 0) { throw 'Arc status check failed.' }
   & $arc check --location japaneast
   if ($LASTEXITCODE -ne 0) { throw 'Arc network check failed.' }
   ```

確認してみよう。Portal の状態は `Connected`、`azcmagent show` の有効な Proxy は指定した URL になっていますか。`check` の結果でも、接続先と Proxy の利用状況を見てみましょう。

## 3-3. Azure Portal で AMA を導入し、Day1 の DCR に追加する

Day1 の DCR に Arc サーバーを追加すると、Portal が AMA を導入し、DCR を関連付けます。操作場所は管理端末の Azure Portal です。

### DCR に Arc サーバーを追加する

1. Portal の「Azure Arc → マシン」で、対象のオンプレミス VM `onprem-vm01` が `Connected` になっていることを確認します。
2. 「モニター → 設定 → データ収集ルール」から、Day1 の `dcr-arclab-windows` を開きます。
3. 「リソース → 追加」で登録した Arc サーバー `onprem-vm01` を選択し、「適用」を押します。Portal は AMA がなければ導入し、DCR を関連付けます。既存 AMA がある場合は、その設定も確認してください。
4. リソース一覧で、Day1 の Azure VM と今回の Arc サーバーが関連付いていることを確認します。
6. Arc サーバーの「拡張機能」で、`AzureMonitorWindowsAgent` のプロビジョニング状態が「成功」になるまで待ちます。失敗した場合は、拡張機能のエラー詳細、Arc の接続状態と Proxy の通信許可を確認してください。

確認してみよう。AMA の導入状態は「成功」、Day1 の DCR には2台の収集元が表示されていますか。

## 3-4. オンプレミス VM の Heartbeat を確認する

Day1 の LAW の「ログ」を開き、KQL モードで次を実行します。`targetResourceId` は、今回登録した Arc サーバーの「概要 → JSON ビュー」の `id` に置き換えてください。

```kusto
let targetResourceId = "<対象 Arc サーバーの JSON ビューの id>";
Heartbeat
| where TimeGenerated > ago(30m)
| where _ResourceId =~ targetResourceId
| where Category == "Azure Monitor Agent"
| project TimeGenerated, IngestedAt=ingestion_time(), Computer, _ResourceId
| order by TimeGenerated desc
| take 100
```

確認してみよう。`Computer` はオンプレミス VM のコンピューター名、`_ResourceId` は `Microsoft.HybridCompute/machines` を含む今回の Arc リソース ID になっていますか。

初回の反映には数分から15分程度かかる場合があります。0件の場合は少し待って再実行し、Arc の状態、AMA、DCR の関連付け、Proxy の許可と LAW の日次上限を確認してください。検索に失敗した場合は、表示されたエラーと LAW の閲覧権限を確認します。

## 3-5. Application イベントを生成して収集を確認する

対象のオンプレミス VM の Windows PowerShell 5.1 を管理者として開き、[Day1 のイベント生成コード](02-azure.md#2-5-サンプルの-application-イベントを生成して収集を確認する)を実行します。同じ Source `ArcAmaHandson`、EventID `9001` でイベントを1件生成し、新しい `Marker` と `GeneratedUtc` が出力されます。

オンプレミス VM に今回の [Write-HandsonEvent.ps1](../scripts/Write-HandsonEvent.ps1) を配置した場合は、そのスクリプトも使用できます。次の例は `C:\Handson` に配置した場合です。Step0 のラボでは、このフォルダーに配置されます。

```powershell
Set-Location 'C:\Handson'
.\Write-HandsonEvent.ps1
```

LAW の「ログ」で、今回の Arc リソース ID、`Marker`、`GeneratedUtc` に置き換えて実行します。

```kusto
let targetResourceId = "<対象 Arc サーバーの JSON ビューの id>";
let marker = "<今回出力された Marker>";
let generatedUtc = todatetime("<今回出力された GeneratedUtc>");
Event
| where TimeGenerated between ((generatedUtc - 1m) .. (generatedUtc + 29m))
| where _ResourceId =~ targetResourceId
| where EventLog == "Application" and Source == "ArcAmaHandson" and EventID == 9001
| where RenderedDescription contains marker
| project TimeGenerated, IngestedAt=ingestion_time(), Computer, EventID,
          Source, RenderedDescription, _ResourceId
| order by TimeGenerated desc
| take 100
```

確認してみよう。今回の `Marker` とオンプレミス VM の `Computer` が表示されていますか。生成時刻と取り込み時刻も見比べてみましょう。

0件の場合は少し待って再実行し、オンプレミス VM のイベントビューアーで Source と EventID を確認します。DCR の XPath と Proxy の通信許可も確認してください。

## 3-6. Day1 のアラートとメール通知を試す

1. LAW の「アラート規則」で、Day1 の `alert-arclab-event9001` を開きます。
2. スコープが今回の LAW、条件が Source `ArcAmaHandson` / EventID `9001`、ディメンションが `Computer` になっていることを確認します。「将来の値をすべて含める」が有効なら、今回追加したオンプレミス VM も対象になります。
3. アクショングループ `ag-arclab-mail` の関連付けと、規則が有効になっていることを確認します。
4. オンプレミス VM で生成したイベントが LAW に到着した後、Portal の「モニター → アラート」を開きます。対象サブスクリプション、RG、時間範囲と「情報」の重大度で表示を絞り込みます。
5. `alert-arclab-event9001` のアラートを開き、ディメンションの `Computer` がオンプレミス VM のコンピューター名になっていることを確認します。
6. Day1 でご自身のメールアドレスを設定し、宛先確認を済ませたお客様は、受信メールで規則名とオンプレミス VM のコンピューター名も確認してください。メール受信を試すかどうかは、お客様の判断で決めてください。

規則は直近15分のイベントを5分間隔で評価します。ログの取り込み、規則の評価、通知まで少し待って確認してください。取り込みが遅れて検索範囲を過ぎた場合は、オンプレミス VM で新しいイベントを生成して試せます。

Day1 の Azure VM とオンプレミス VM は、`Computer` が異なるため個別に評価されます。同じオンプレミス VM でもう一度通知を試したい場合は、その VM のアラートが「解決済み（Resolved）」になってからイベントを再度生成してください。自動解決までの時間は [Day1 の通知確認](02-azure.md#2-8-イベントを再度生成してアラートを確認する)を参照してください。

確認してみよう。同じ LAW、DCR、アラート規則を使って、オンプレミス VM のイベントも検知できましたか。

## 収集元と通信設定を見比べる

LAW の「ログ」で次を実行すると、共通の演習イベントを送ったコンピューターとリソース ID を一覧にできます。

```kusto
Event
| where TimeGenerated > ago(30m)
| where EventLog == "Application" and Source == "ArcAmaHandson" and EventID == 9001
| summarize EventCount=count(), LastEventUtc=max(TimeGenerated) by Computer, _ResourceId
| order by LastEventUtc desc
| take 100
```

Day1 のイベントが30分より前の場合は、Azure VM でも新しいイベントを生成してから見比べてください。Azure VM は `Microsoft.Compute/virtualMachines`、オンプレミス VM は `Microsoft.HybridCompute/machines` のリソース ID で区別できます。


## 公式資料

- [Azure Firewall の Explicit Proxy](https://learn.microsoft.com/azure/firewall/explicit-proxy)
- [Arc Agent のネットワーク要件](https://learn.microsoft.com/azure/azure-arc/servers/network-requirements)
- [Portal による Arc 登録](https://learn.microsoft.com/azure/azure-arc/servers/onboard-portal)
- [Arc Agent の Proxy 設定](https://learn.microsoft.com/azure/azure-arc/servers/manage-agent-proxy-settings)
- [AMA のネットワークと Proxy 設定](https://learn.microsoft.com/azure/azure-monitor/agents/azure-monitor-agent-network-configuration)
- [DCR のリソース追加](https://learn.microsoft.com/azure/azure-monitor/vm/data-collection)
