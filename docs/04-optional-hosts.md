# 4.3 hosts を用いる場合

[README](../README.md) | [4. オプション](04-optional.md) | [Step1：Azure VM のログ収集](02-azure.md)

Azure VM が参照する DNS サーバーを変更できず、AMA の公開接続先を名前解決できない場合に、対象 VM の hosts に FQDN と IPv4 アドレスを一時的に追加する演習です。

hosts の設定で名前解決を補い、Step1 の AMA 導入とログ収集を試します。NSG の送信許可とインターネットへの送信経路は、Step1 の設定を使用します。

## 始める前に

- Step1 の LAW を作成し、対象 Azure VM のリージョンと LAW の Workspace ID を確認してください。
- 対象 VM の管理者権限と、対象 FQDN を名前解決できる管理端末を用意してください。
- 公開エンドポイントの IP アドレスは変更されます。hosts の設定は自動で更新・失効しないため、一時的な演習用として使用してください。
- hosts にはワイルドカードやポート番号を指定できません。各 FQDN に実際の IP アドレスを記載します。

## 1. 接続先の IPv4 アドレスを取得する

名前解決できる管理端末の PowerShell で実行します。`japaneast` は対象 Azure VM のリージョン、`<Workspace ID>` は LAW の「概要」に表示される GUID に置き換えてください。

```powershell
$region = 'japaneast'
$workspaceId = '<Workspace ID>'
$names = @(
    'management.azure.com'
    'global.handler.control.monitor.azure.com'
    "$region.handler.control.monitor.azure.com"
    "$workspaceId.ods.opinsights.azure.com"
    'global.prod.microsoftmetrics.com'
)
foreach ($name in $names) {
    Resolve-DnsName -Name $name -Type A -DnsOnly -ErrorAction Stop |
        Where-Object Type -eq 'A' |
        Select-Object @{Name='Fqdn'; Expression={ $name }}, IPAddress, TTL
}
```

最終的な A レコードの `IPAddress` を使用します。`NameHost` に表示される CNAME ではなく、上記の元の FQDN を hosts の右側に記載してください。複数の IP が返る場合、この演習では使用する1個を選び、TCP 443 の接続を確認します。DNS による宛先の切り替えは hosts では再現できません。

`management.azure.com` は ARM の操作向けに含めています。通常の Windows イベント収集だけで、常にこの FQDN への接続が必要になるという意味ではありません。

## 2. hosts の記入例

次の `<...>` は、前節で取得した IPv4 アドレスと Workspace ID に置き換えます。このままでは使用できません。

```text
<ARMのIPv4> management.azure.com
<制御サービスのIPv4> global.handler.control.monitor.azure.com
<リージョン制御サービスのIPv4> japaneast.handler.control.monitor.azure.com
<LAW取り込み先のIPv4> <Workspace ID>.ods.opinsights.azure.com
<メトリックサービスのIPv4> global.prod.microsoftmetrics.com
```

### rg-arcama-lab の取得例

2026年11月26日14:38 JST に管理端末で取得した値です。対象 Azure VM は `arcama-workload`、リージョンは `japaneast`、LAW は `law-arcama` です。

```text
4.150.240.10    management.azure.com
40.64.132.130   global.handler.control.monitor.azure.com
20.43.64.69     japaneast.handler.control.monitor.azure.com
40.79.194.118   680b2bdc-f503-46f5-a09f-06470e5695cc.ods.opinsights.azure.com
40.77.24.27     global.prod.microsoftmetrics.com
```

取得した DNS の TTL は5〜68秒でした。上記は取得時点の値であり、現在も同じ IP で接続できることを保証しません。適用前に前節のコマンドで再取得してください。

同じ内容の [hosts 追記用サンプル](../samples/hosts-rg-arcama-lab.txt) も用意しています。他の LAW を使用する場合は Workspace ID も置き換えます。

## 3. 対象 Azure VM の hosts に追記する

操作場所は対象 Azure VM です。Bastion などで接続し、Windows PowerShell を管理者として開きます。

1. 既存 hosts のバックアップを作成し、メモ帳で開きます。

   ```powershell
   $hostsPath = Join-Path $env:SystemRoot 'System32\drivers\etc\hosts'
   $backupPath = "$hostsPath.$(Get-Date -Format 'yyyyMMdd-HHmmss').bak"
   Copy-Item -LiteralPath $hostsPath -Destination $backupPath -ErrorAction Stop
   notepad.exe $hostsPath
   ```

2. 元の内容は残し、取得した5件を末尾へ追記して保存します。同じ FQDN の既存行がある場合は、用途を確認してから編集し、重複した別 IP の行を作らないようにしてください。サンプルファイルで hosts 全体を置き換えないでください。
3. DNS キャッシュを消去します。

   ```powershell
   ipconfig /flushdns
   ```

## 4. 名前解決と接続を確認する

対象 VM の PowerShell で、前節に設定した FQDN を確認します。次はグローバル制御サービスの例です。

```powershell
$fqdn = 'global.handler.control.monitor.azure.com'
[Net.Dns]::GetHostAddresses($fqdn) | Select-Object IPAddressToString
Test-NetConnection -ComputerName $fqdn -Port 443
```

他の4件も `$fqdn` を置き換えて実行します。`IPAddressToString` が hosts の値になっているか、`TcpTestSucceeded` が `True` かを確認してください。`nslookup` は DNS サーバーへ問い合わせるため、hosts の確認には使用しません。

確認してみよう。対象 VM で設定した IP アドレスが表示され、TCP 443 で接続できましたか。その後、Step1 の AMA 導入、Heartbeat、イベント収集の手順を進めてください。

TCP 接続が失敗した場合は、取得した IP と NSG の許可、送信経路を確認します。拡張機能のダウンロードなど、別の FQDN で名前解決に失敗する場合は、エラーに表示された接続先も管理端末で調べてください。

## 5. 一時的な追記を戻す

演習が終わったら、管理者として hosts を開き、今回追加した行だけを削除して保存します。`ipconfig /flushdns` を実行してください。他の操作で追加された行を残すため、バックアップで全体を上書きする操作は避けてください。

DNS の問題が続いている間は、追記を戻すと名前解決とログ収集に影響します。戻すタイミングは演習の利用状況に合わせて決めてください。
