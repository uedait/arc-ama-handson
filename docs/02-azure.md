# Day1／Step1：Azure VM のログ収集とメール通知

[README](../README.md) | [Step0：検証環境の構築](01-lab.md)

Azure VM に Azure Monitor Agent（AMA）を導入し、Windows イベントを Log Analytics ワークスペース（LAW）へ収集します。その後、サンプルイベントを検知するログ検索アラートとメール通知を設定します。


## 始める前に

- Windows Server の Azure VM と、VM を操作するための管理接続を用意してください。Step0 のラボでは workload VM に Bastion で接続します。
- Azure Portal で LAW、DCR、アラート規則、アクショングループを作成でき、対象 VM の ID、拡張機能、NSG を設定できるアカウントを使用してください。ログの検索にも LAW の閲覧権限が必要です。
- 既存 VM を使用する場合は、既存の監視、マネージド ID、DCR、送信規則を確認してから進めてください。
- LAW の取り込みと保持、ログ検索アラートには費用が発生します。コストを確認し、演習に使用する名前、リージョン、利用期間を決めてください。

この手順は Azure Portal を中心に進めます。画面の項目名は今後変更になる可能性があります。

### 入力例

Step0 の既定値を使用した場合の例です。お客様の環境では、対象の名前やリージョンに置き換えてください。

| 対象 | 例 |
|---|---|
| リソースグループ（RG） | `rg-arclab-lab` |
| リージョン | Japan East |
| Azure VM / Private IP | `arclab-workload` / `10.10.1.4` |
| LAW | `law-arclab` |
| DCR | `dcr-arclab-windows` |
| アラート規則 | `alert-arclab-event9001` |
| アクショングループ | `ag-arclab-mail` |
| メールアドレス | `handson-alert@example.com` |

メールアドレスは架空のものですので、メール受信も試したい場合は、受信できるご自身のアドレスに置き換えてください。

## 2-1. LAW を作成する

操作場所は Azure Portal です。

1. 上部の検索で「Log Analytics ワークスペース」を開き、「作成」を選びます。
2. 次を入力します。

   | 項目 | 入力例 |
   |---|---|
   | サブスクリプション | 演習に使用するサブスクリプション |
   | リソースグループ | `rg-arclab-lab` |
   | 名前 | `law-arclab` |
   | リージョン | Japan East |

3. 「確認および作成」で入力値を確認し、作成します。
4. 作成した LAW の「概要」で、名前、RG、リージョンと Workspace ID を確認します。
5. 「ネットワークの分離」の Public access で、取り込み（Ingestion access）とクエリ（Query access）を、すべてのネットワークから有効にします。
6. 「使用量と推定コスト」で取り込みプラン、保持期間、日次上限を確認します。保持期間の例は30日です。日次上限を設定する場合は、到達するとログ収集が止まることに注意してください。

確認してみよう。作成した LAW の名前と Workspace ID を確認できましたか。Day2 でもこの LAW を使用します。

## 2-2. Azure VM の送信通信を許可する

AMA は Azure Monitor の公開エンドポイントへ接続します。対象 VM の送信制限に、その通信を許可する規則を追加します。

1. Portal で対象 VM を開き、「ネットワーク設定」から NIC と、その NIC に関連付いた NSG を確認します。サブネットにも NSG がある場合は、そちらの送信規則も確認してください。
2. NSG の「送信セキュリティ規則」を開き、Internet 向け通信を制限している Deny 規則の優先度を確認します。Step0 のラボは `LabInternetOutbound` の優先度 `400` です。
3. 「追加」から次の2規則を作成します。優先度は、送信方向で未使用の番号かつ Deny 規則より小さい番号を使用してください。番号が小さい規則から評価されます。

   | 項目 | Azure Monitor 用 | Azure Resource Manager 用 |
   |---|---|---|
   | ソース | IP Addresses | IP Addresses |
   | ソース IP | 対象 VM の Private IP。ラボは `10.10.1.4` | 同左 |
   | ソースポート範囲 | `*` | `*` |
   | 宛先 | Service Tag | Service Tag |
   | 宛先サービス タグ | `AzureMonitor` | `AzureResourceManager` |
   | 宛先ポート範囲 | `443` | `443` |
   | プロトコル | TCP | TCP |
   | アクション | Allow | Allow |
   | 優先度の例 | `200` | `201` |
   | 名前の例 | `AllowAmaAzureMonitor` | `AllowAmaArm` |

4. サブネットにも NSG がある場合は、両方の NSG で対象 VM の通信が許可されるように設定します。
5. VM の「ネットワーク設定」から NIC の「有効なセキュリティ規則」を開き、今回の送信許可を確認します。

NSG の許可に加えて、VM がインターネットへ送信できる経路も必要です。Step0 のラボでは workload VM の Public IP を使用します。お客様の既存 VM では、既存の送信方式とルートを確認してください。

AMA の拡張機能を配布する Azure VM Agent は、`168.63.129.16` の TCP `80` と `32526` を使用します。Step0 のラボはこの通信を許可済みです。既存 VM ではゲスト Firewall も含めて確認してください。サポート対象の VM Agent は、この経路で拡張機能パッケージの取得と状態報告を行えます。

確認してみよう。対象 VM の TCP 443 の送信許可は、Internet 向け Deny より先に評価される設定になっていますか。

## 2-3. マネージド ID、DCR、AMA を設定する

DCR は、AMA が収集するデータと、その保存先を定義します。この演習では、Application ログの Source `ArcAmaHandson`、EventID `9001` を収集します。

### Azure VM のマネージド ID

1. Portal で対象 VM の「ID」を開きます。
2. Step0 の workload VM では、「システム割り当て」の状態を「オン」にして保存します。
3. 既存 VM にユーザー割り当て ID がある場合は、利用中の構成に合わせて、DCR のリソース追加時に AMA が使用する ID を選択してください。

AMA は VM のマネージド ID を使用して認証します。

### DCR の作成と AMA の導入

1. Portal の検索で「監視」を開き、「設定 → データ収集ルール → 作成」を選びます。
2. 基本設定を入力します。

   | 項目 | 入力例 |
   |---|---|
   | 名前 | `dcr-arclab-windows` |
   | サブスクリプション / RG | 演習に使用するサブスクリプション / `rg-arclab-lab` |
   | リージョン | LAW と同じ Japan East |
   | テレメトリの種類 | `Agent-based - Windows or Linux` |

   従来の作成画面では「プラットフォームの種類」で Windows を選択します。

3. 「リソース → リソースの追加」で対象 Azure VM を選択します。ラボでは `arclab-workload` です。マネージド ID の選択が表示された場合は、先ほど確認した ID を指定します。
4. 「収集と配信」で「新しいデータフローを追加」または「データソースの追加」を選びます。
5. データソースの種類に「Windows イベント ログ」、選択モードに「カスタム」を指定し、次の XPath を入力します。

   ```text
   Application!*[System[(EventID=9001) and Provider[@Name='ArcAmaHandson']]]
   ```

6. 宛先に「Log Analytics ワークスペース」を選び、`law-arclab` を追加します。データソースと宛先を保存し、収集と配信の一覧に反映されたことを確認します。
7. 「確認および作成」で対象 VM と LAW を確認し、DCR を作成します。Portal は対象 VM に AMA がなければ拡張機能を導入し、DCR を関連付けます。
8. VM の「拡張機能とアプリケーション」で、`AzureMonitorWindowsAgent` のプロビジョニング状態が「成功」になるまで待ちます。失敗した場合は、その拡張機能のエラー詳細と、前節の送信許可・VM Agent の通信を確認してください。
9. DCR の「リソース」で、対象 VM が関連付いていることを確認します。

この Azure VM の AMA は、公開エンドポイントへ直接接続する設定を使用します。新規ラボの AMA は既定の直接接続で進めます。既存 AMA に Proxy の設定がある場合は、変更範囲を管理者と確認し、章末の公式ネットワーク設定手順に沿って直接接続へ設定してください。

確認してみよう。DCR の収集対象は Application の `ArcAmaHandson` / `9001`、保存先は今回の LAW になっていますか。

## 2-4. Heartbeat を確認する

Heartbeat は AMA が稼働していると定期的に送信されます。

1. 対象 VM の「概要 → JSON ビュー」で、完全なリソース ID の `id` をコピーします。
2. 今回の LAW の「ログ」を開き、KQL モードで次を実行します。`targetResourceId` をコピーした ID に置き換えてください。

   ```kusto
   let targetResourceId = "<対象 Azure VM の JSON ビューの id>";
   Heartbeat
   | where TimeGenerated > ago(30m)
   | where _ResourceId =~ targetResourceId
   | where Category == "Azure Monitor Agent"
   | project TimeGenerated, IngestedAt=ingestion_time(), Computer, _ResourceId
   | order by TimeGenerated desc
   | take 100
   ```

確認してみよう。`Computer` と `_ResourceId` は対象 VM の値になっていますか。初回の反映には数分から15分程度かかる場合があります。

結果が0件の場合は少し待って再実行し、AMA の状態、DCR の関連付け、送信許可、LAW の日次上限を確認してください。検索自体が失敗した場合は、表示されたエラーと LAW の閲覧権限を確認します。

## 2-5. サンプルの Application イベントを生成して収集を確認する

### Azure VM でイベントを生成する

対象 VM に Bastion などで接続し、スタートメニューから「Windows PowerShell」を管理者として開きます。このコードは `New-EventLog` と `Write-EventLog` を使用しますので、 VM 内の Windows PowerShell 5.1 で実行してください。

```powershell
$ErrorActionPreference = 'Stop'
$identity = [Security.Principal.WindowsIdentity]::GetCurrent()
$principal = New-Object Security.Principal.WindowsPrincipal($identity)
if (-not $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)) {
    throw 'Run Windows PowerShell as Administrator.'
}
$source = 'ArcAmaHandson'
if ([Diagnostics.EventLog]::SourceExists($source) -and
    [Diagnostics.EventLog]::LogNameFromSourceName($source, '.') -ne 'Application') {
    throw 'The event source belongs to another log.'
}
if (-not [Diagnostics.EventLog]::SourceExists($source)) {
    New-EventLog -LogName Application -Source $source
}
$marker = [guid]::NewGuid().ToString()
$utc = [datetime]::UtcNow.ToString('o')
Write-EventLog -LogName Application -Source $source -EventId 9001 `
    -EntryType Information -Message "ArcAmaHandson marker=$marker; utc=$utc"
[pscustomobject]@{
    Marker = $marker; GeneratedUtc = $utc; Computer = $env:COMPUTERNAME
    EventId = 9001; Source = $source
}
```

Application ログにイベントを1件生成します。出力された `Marker` と `GeneratedUtc` を、次の検索に使用します。VM のイベントビューアーでも「Windows ログ → Application」から Source と EventID を確認できます。

このコードと同じ Source、EventID、メッセージ形式で、リポジトリの [Write-HandsonEvent.ps1](../scripts/Write-HandsonEvent.ps1) もイベントを生成します。

### LAW で今回のイベントを探す

LAW の「ログ」で次を実行します。VM のリソース ID と、今回出力された `Marker`、`GeneratedUtc` を置き換えてください。

```kusto
let targetResourceId = "<対象 Azure VM の JSON ビューの id>";
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

確認してみよう。今回の `Marker` を含むイベントが見つかりましたか。`TimeGenerated` と `IngestedAt` を見比べると、イベントの生成から取り込みまでの時間も確認できます。

初回は少し待って再実行してください。0件が続く場合は、VM のイベントビューアーと DCR の XPath を見比べます。検索エラーが表示された場合は、その内容を確認してください。

## 2-6. LAW にログ検索アラートを作成する

1. LAW の「アラート」から「作成 → アラート規則」を選びます。
2. 「スコープ」が今回の LAW `law-arclab` になっていることを確認します。リソースの種類は `Microsoft.OperationalInsights/workspaces` です。
3. 「条件」で「カスタム ログ検索」を選択し、次の KQL を入力します。

   ```kusto
   Event
   | where EventLog == "Application" and Source == "ArcAmaHandson" and EventID == 9001
   | project TimeGenerated, Computer, Source, EventID, RenderedDescription, _ResourceId
   ```

   このクエリは、LAW に届く共通の演習イベントを検索します。前節のログ確認用クエリと用途が異なるため、ここでは上記のクエリを使用してください。検索する期間は次のウィンドウ サイズで設定します。

4. 「実行」で結果を確認し、「アラートの編集を続行」などの操作で条件設定へ戻ります。
5. 測定とアラート ロジックを設定します。

   | 項目 | 設定 |
   |---|---|
   | 測定 | テーブル行（Table rows） |
   | 集計の種類 | Count |
   | 集計の粒度 / ウィンドウ サイズ | 15分 |
   | しきい値の種類 | 静的 |
   | 演算子 | 次の値より大きい（Greater than） |
   | しきい値 | `0` |
   | 評価頻度 | 5分 |
   | アラートを発生させる違反数 | 1回。評価期間は15分 |

   直近15分に対象イベントが1件以上あると、条件を満たします。評価頻度を短くすると推定コストも変わるため、画面に表示される料金を確認してください。

6. 「ディメンションで分割」で次を設定します。

   | 項目 | 設定 |
   |---|---|
   | ディメンション名 | `Computer` |
   | 演算子 | 含む（Include） |
   | ディメンション値 | 現在のすべての値 |
   | 将来の値をすべて含める（Include all future values） | 有効 |

   `Computer` が候補に表示されるには、クエリ結果にこの列が必要です。現在の VM の値を選択したうえで、将来の値も含む設定を有効にしてください。

   この設定で、Day2 に追加するオンプレミス VM も同じ規則の対象になります。各 `Computer` のイベント件数を個別に評価し、通知にも対象のコンピューター名が含まれます。

確認してみよう。スコープは LAW、ディメンションは `Computer`、将来の値を含む設定になっていますか。

## 2-7. メール通知用アクショングループを設定する

アラート規則の作成画面で、そのまま「アクション」へ進みます。

1. 「アクショングループの選択 → アクショングループの作成」を選びます。
2. 基本設定を入力します。

   | 項目 | 入力例 |
   |---|---|
   | サブスクリプション / RG | 演習に使用するサブスクリプション / `rg-arclab-lab` |
   | リージョン | Global |
   | アクショングループ名 | `ag-arclab-mail` |
   | 表示名 | `ArcAmaLab` |

3. 「通知」で、種類「メール / SMS メッセージ / プッシュ / 音声」を選び、次を入力します。

   | 項目 | 入力例 |
   |---|---|
   | 通知名 | `HandsonEmail` |
   | メール | 有効 |
   | メールアドレス | `handson-alert@example.com` |
   | 共通アラート スキーマ | 有効 |

   メール通知はこの「通知」タブで設定します。「確認および作成」へ進み、アクショングループを作成してください。

4. 宛先確認が求められた場合、受信も試すお客様は、ご自身の実際のメールアドレスに届く確認コード（OTP）で宛先を確認します。公式手順では、保存から30分以内の確認が必要です。期限が切れた場合は、アクショングループを開いて確認コードを再送できます。

   架空の宛先ではメール受信や宛先確認はできません。架空の例を使用する場合は、通知先の設定を見ながら、Portal でのアラート確認を進めてください。宛先未確認の状態では、通知の送信が制限される場合があります。

5. アラート規則の作成画面に戻り、作成した `ag-arclab-mail` が選択されていることを確認します。
6. 「詳細」で次を設定します。

   | 項目 | 設定例 |
   |---|---|
   | サブスクリプション / RG | 演習に使用するサブスクリプション / `rg-arclab-lab` |
   | アラート規則名 | `alert-arclab-event9001` |
   | 説明 | Application の演習イベントを検知してメール通知する |
   | 重大度 | 3：情報 |
   | リージョン | LAW と同じ Japan East |
   | ID | なし（None）。規則作成者の LAW 閲覧権限を使用 |
   | 作成時に有効にする | 有効 |
   | アラートを自動的に解決する | 有効 |

7. 「確認および作成」で LAW、条件、`Computer` の設定、アクショングループを確認し、アラート規則を作成します。

確認してみよう。アラート規則にメール通知用アクショングループが関連付いていますか。

メール受信を試す場合は、作成済みアクショングループの「テスト」からサンプルのログ検索アラートを選び、メール通知を試すこともできます。このテストは通知先の確認用です。VM のイベント検知は次の節で試します。

## 2-8. イベントを再度生成してアラートを確認する

1. VM 内の Windows PowerShell で、[2-5 のイベント生成コード](#2-5-サンプルの-application-イベントを生成して収集を確認する)をもう一度実行します。新しい `Marker` と `GeneratedUtc` が出力されます。
2. LAW のログで、今回の値を使ってイベントの到着を確認します。
3. Portal の「モニター → アラート」を開き、対象サブスクリプション、RG、時間範囲で絞り込みます。必要なら重大度のフィルターに「情報」を含めます。
4. `alert-arclab-event9001` のアラートを開き、発生時刻、状態、ディメンションの `Computer` を確認します。
5. ご自身のメールアドレスを設定して宛先確認を済ませたお客様は、受信したメールでアラート規則名と対象のコンピューター名を確認します。アクショングループ追加時の確認メールや、任意のテストメールとは別に、イベントを検知した通知を確認してください。

ログの取り込み後、規則は5分間隔で評価されます。規則の反映やメール送信にも時間がかかるため、イベント到着後に少し待って確認してください。イベントの取り込みが遅れて15分の検索範囲を過ぎた場合は、新しいイベントを生成して試せます。

最初に生成したイベントが直近15分に残っている場合、規則を作成した直後にアラートが発生することもあります。そのアラートでも、共通条件による検知と通知を確認できます。

同じ `Computer` で既にアラートが「発生（Fired）」している間は、イベントを追加しても新しい発生通知は送られません。もう一度通知を試したい場合は、「解決済み（Resolved）」になってからイベントを生成してください。最後のイベントが15分の検索範囲から外れ、その後の5分間隔の評価で3回連続して条件を満たさなくなると自動解決します。解決まで30分以上かかることもあります。

確認してみよう。イベントの生成、LAW への取り込み、アラートの発生を順に確認できましたか。メール受信を試した場合は、その通知も見比べてみましょう。

## Day2 で使う設定

Day2 では今回の LAW、DCR、アラート規則、アクショングループを引き続き使用します。オンプレミス VM を同じ DCR に追加し、同じ Source `ArcAmaHandson`、EventID `9001` のイベントを生成します。

Day2 でも、LAW 全体の共通イベントを `Computer` ごとに検知する今回のアラート規則をそのまま使用します。

## Optional：監視コストの予算アラート

監視用 RG の費用も確認したい場合は、[Optional：監視リソースグループの予算アラート](04-optional-budget.md)を実施してください。月次予算と実績・予測のメール通知を設定します。既定の `rg-arclab-lab` を対象にすると、監視以外のリソース費用も含まれます。

## 公式資料

- [AMA のネットワーク構成](https://learn.microsoft.com/azure/azure-monitor/agents/azure-monitor-agent-network-configuration)
- [Azure VM 拡張機能のネットワーク要件](https://learn.microsoft.com/azure/virtual-machines/extensions/features-windows#network-access)
- [DCR の Portal 作成と AMA 導入](https://learn.microsoft.com/azure/azure-monitor/vm/data-collection)
- [Windows イベントと XPath](https://learn.microsoft.com/azure/azure-monitor/vm/data-collection-windows-events)
- [ログ検索アラートの作成](https://learn.microsoft.com/azure/azure-monitor/alerts/alerts-create-log-alert-rule)
- [アクショングループとメール通知](https://learn.microsoft.com/azure/azure-monitor/alerts/action-groups)
