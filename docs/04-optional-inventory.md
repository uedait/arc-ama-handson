# 4.4 インベントリ管理と変更追跡

[4. オプション](04-optional.md) | [Step1：Azure VM のログ収集](02-azure.md) | [Step2：オンプレミス VM のログ収集](03-onpremises.md)

Azure VM と Arc 登録済みのオンプレミス VM で、Azure Monitor Agent（AMA）版の Change Tracking & Inventory を使います。インベントリで「どの VM に、どのソフトウェアが、どのバージョンで入っているか」を確認します。VM にログインせずに、更新対象の特定や作業結果の確認ができることを確かめます。

## 始める前に

- Step1・Step2 を完了し、同じ Log Analytics ワークスペース（LAW）で両 VM の Heartbeat を確認します。
- 既定の例では、Azure VM は `arclab-workload`、Arc マシンは `ArcOnpremL2`、LAW は `law-arclab` です。リソース名は実際の環境に読み替えてください。
- Azure で DCR の作成と関連付け、拡張機能の導入、LAW の検索ができる権限と、両 VM でソフトウェアを導入・削除できる権限が必要です。

## 1. Azure Portal で有効化の準備をする (Optional)
Portal の「有効化」では、変更追跡用 DCR が `ct-dcr-...` などの名前で自動作成される場合があります。任意の命名規則の DCR を作成したい場合には、公式のテンプレートからのカスタムデプロイを行う必要があります。命名をコントロールする必要がある場合には、専用 DCR を手動でAzure Portal で用意します。

1.公式テンプレートを読み込みます。 Microsoft Learn の CtDcrCreation.json の JSON をコピーします。Portal で「カスタム テンプレートのデプロイ」を検索し、「エディターで独自のテンプレートを作成」を選びます。エディター内の初期内容を JSON で置き換え、「保存」を選びます。
2.デプロイ先と送信先を指定します。 「基本」で law-arclab と同じサブスクリプションとリソースグループ rg-arclab-lab を選びます。Data Collection Rule Name に未使用の名前（例 dcr-arclab-change-tracking）、Workspace Resource Id に手順1の値を入力します。DCR はテンプレートによってワークスペースと同じリージョンに作成されます。
3.作成結果を確認します。 「確認と作成」→「作成」を選びます。デプロイ完了後、「Azure Monitor」→「データ収集ルール」で、指定した名前の DCR が表示されることを確認します。


## 2. Azure VM とオンプレミス VM に適用する

1. Azure Portal の「仮想マシン」で対象 Azure VM を開き、リソースメニューから「変更追跡（Change tracking）」を開きます。Arc マインの場合は、「Azure Arc → マシン」で対象のオンプレミス VM を開き、「操作 → 変更追跡」を選びます。
3. デプロイ完了後、Azure VM の「拡張機能とアプリケーション」と Arc マシンの「拡張機能」を開きます。種類 `ChangeTracking-Windows`、発行元 `Microsoft.Azure.ChangeTrackingAndInventory` の拡張機能が正常に導入されたことを確認します。
4. 「モニター → データ収集ルール」で、各 VM に関連付いている変更追跡用 DCR の「リソース」と宛先を確認します。


## 3. 適用結果を確認する

1. Azure Portal で各 VM の「変更追跡 → インベントリ」を開きます。両 VM のソフトウェア一覧が取得できます。
3. 一例として、 7-Zip をマシンにインストールします。オンプレミス VM のインベントリに製品名とバージョンが表示され、「変更追跡」に追加が記録されたことを確認します。この時点では Azure VM のインベントリに 7-Zip はありません。
3. Azure VM にも 7-Zip をインストールし、両方のインベントリに表示されることと、それぞれの追加が別の VM・時刻で記録されることを確認します。

[Windows ソフトウェアの収集間隔](https://learn.microsoft.com/ja-jp/azure/azure-change-tracking-inventory/change-tracking-inventory-support-matrix)は30分です。反映までは時間を要します。

### LAW で2台の結果を比較する

LAW の「ログ」で、2つのリソース ID を置き換えて実行します。初回のソフトウェア情報が両 VM から届いたことを確認します。

```kusto
ConfigurationData
| where TimeGenerated >= ago(24h)
| where _ResourceId in~ ("AZURE_VM_RESOURCE_ID", "ARC_VM_RESOURCE_ID")
| where ConfigDataType =~ "Software"
| summarize LastReport = max(TimeGenerated), Records = count() by Computer, _ResourceId
```

次のクエリでは、7-Zip の追加・削除を時刻と VM で見比べられます。`TimeGenerated` はログの記録時刻で、インストーラーを実行した時刻そのものではありません。

```kusto
ConfigurationChange
| where TimeGenerated >= ago(24h)
| where _ResourceId in~ ("AZURE_VM_RESOURCE_ID", "ARC_VM_RESOURCE_ID")
| where ConfigChangeType =~ "Software" and SoftwareName startswith "7-Zip"
| project TimeGenerated, Computer, _ResourceId, SoftwareName, ChangeCategory
| order by TimeGenerated asc
```

両 VM のインベントリが取得でき、意図した VM で追加・削除を区別できれば演習は完了です。
