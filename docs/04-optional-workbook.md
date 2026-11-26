# 4.2 ワークブックの作成

[4. オプション](04-optional.md) | [Step1：Azure VM のログ収集](02-azure.md) | [Step2：オンプレミス VM のログ収集](03-onpremises.md)

メトリックやログは、アラート通知に加えて、日常の点検や障害の切り分けにも使えます。異常が起きた時刻の前後で CPU や空きメモリがどう変わったか、どちらの VM でイベントが増えたかを一つの画面で見比べます。

Azure Monitor ワークブックに、Azure VM 1台と Arc に登録したオンプレミス VM 1台の情報を表示します。「概要」で対応が必要な VM を見つけ、タブを切り替えて原因の切り分けに必要な情報を確認します。

## 完成版 JSON を貼り付けて使う

1. [完成版ワークブック JSON](04-optional-workbook.json) を開きます。GitHub では `Raw` で JSON 本文を表示できます。JSON をテキストエディターにコピーし、次の文字列をすべて置換します。この例では対象のリソースを同じサブスクリプションに配置します。

   | 置換する文字列 | 値 |
   |---|---|
   | `LAB_SUBSCRIPTION_ID` | サブスクリプションの GUID のみ |
   | `LAW_RESOURCE_ID` | LAW のリソース ID。`/subscriptions/` から始まる全文 |
   | `AZURE_VM_RESOURCE_ID` | Azure VM のリソース ID |
   | `ARC_VM_RESOURCE_ID` | Arc マシンのリソース ID |

   リソース ID は Portal の「プロパティ」または「JSON ビュー」で取得できます。Step1 のアラート規則名を変更した場合は、JSON 内の `alert-arclab-event9001` も実際の名前に置き換えます。置換後の JSON を公開リポジトリに追加する必要はありません。

2. Azure Portal で「モニター → ブック → 新規」を開きます。編集モードでツールバーの `</>`（詳細エディター）を開き、「形式」が「ブック テンプレート」になっていることを確認します。
3. コード欄をクリックして `Ctrl+A` で既存の JSON をすべて選び、置換済みの JSON を貼り付けて「適用」を選びます。`{` から `}` までの本文を貼り付けます。Markdown のコードフェンスは含めません。
4. 「概要」が表示されたら、[保存して確認する](#保存して確認する)に進みます。貼り付けで性能収集は始まりません。DCR に性能カウンターがなければ[性能カウンターを収集する](#性能カウンターを収集する)を実施します。

各パネルの設定を学ぶ場合は、[手作業でワークブックを作成する](#手作業でワークブックを作成する)を使ってください。完成版を貼り付けた場合、パネルを追加し直す必要はありません。

## タブの使い方

| タブ | 確認できること |
|---|---|
| 概要 | VM ごとの確認状況、Heartbeat の経過分、CPU と空きメモリの直近5分平均、過去24時間のイベント件数と関連アラートの集計 |
| 性能 | 最新の Heartbeat と、過去24時間の CPU・空きメモリの5分平均 |
| イベント・アラート | 過去24時間のイベント件数の上位10件と、関連アラートの発生時刻・検知状態 |
| 構成 | 2台の OS バージョンと IP、Azure VM の電源状態、Arc マシンの接続状態 |

「概要」の「要確認」は、Heartbeat が10分以上前、CPU が80%以上、または空きメモリが512 MiB未満の場合に表示します。これらは演習用の目安で、アラート規則の閾値ではありません。性能や Heartbeat が空欄の場合はデータ未受信です。関連アラートの「発生中」は、過去24時間に発生したアラートのうち、検知状態が `Fired` の件数です。

## 始める前に

- Step1 と Step2 を完了し、同じ Log Analytics ワークスペース（LAW）で両 VM の Heartbeat とサンプルイベントを確認してください。既定の VM 名は `arclab-workload` と `ArcOnpremL2` です。
- Portal でデータ収集ルール（DCR）を作成して2台に関連付け、ワークブックを保存できるアカウントを使用します。ワークブックの閲覧者にも、LAW のログ検索、Azure VM のインスタンス ビューと NIC、Arc マシン、アラートの読み取り権限が必要です。
- Azure VM、Arc マシン、LAW の各リソースの「プロパティ」または「JSON ビュー」で、Azure リソース ID を控えます。以下の `AZURE_VM_RESOURCE_ID`、`ARC_VM_RESOURCE_ID`、`LAW_RESOURCE_ID` を実際の値に置き換えてください。LAW のリソース ID は、ワークスペース ID（GUID）とは異なります。
- 性能カウンターの追加により LAW の取り込み量が増えます。利用期間とコストを確認し、既存の DCR が同じカウンターを同じ LAW に送信している場合は重複して設定しないでください。

## 性能カウンターを収集する

Step1 の DCR は Windows イベントを収集します。今回の CPU とメモリは、2台のゲスト OS で同じ Windows パフォーマンスカウンターを測定し、同じ LAW の `Perf` テーブルに送ります。

1. Portal の「モニター → データ収集ルール → 作成」を開きます。名前は `dcr-arclab-workbook-perf`、リージョンは LAW と同じ場所を選びます。テレメトリはエージェントベース、プラットフォームを指定する画面では Windows を選択します。
2. 「リソース」で、Step1 の Azure VM と Step2 の Arc マシンだけを追加します。既存の AMA と ID、オンプレミス側の Proxy 設定は引き続き使用します。
3. 「収集と配信」で「パフォーマンス カウンター → カスタム」を選び、次の2つを追加します。サンプリング間隔は両方とも `60` 秒に設定します。

   | 指標 | カウンター |
   |---|---|
   | ゲスト OS 全体の CPU 使用率 (%) | `\Processor Information(_Total)\% Processor Time` |
   | 利用可能なメモリ (バイト) | `\Memory\Available Bytes` |

4. 宛先の種類は「Azure Monitor ログ」（画面によっては「Log Analytics ワークスペース」）を選び、Step1 の LAW を指定します。DCR を作成し、「リソース」に2台が表示されることを確認します。既存のイベント収集用 DCR とアラート規則は変更しません。

## 手作業でワークブックを作成する

Portal の「モニター → ワークブック → 新規」を開きます。「追加 → クエリの追加」で以下の項目を順に作ります。クエリのタイトルを付け、実行結果を確認してから各項目の編集を完了します。

完成版と同じレイアウトにする場合は、「追加 → リンクまたはタブの追加」で4つのタブを作ります。各タブのアクションを「パラメーター値の設定」にし、共通のパラメーター `Section` に `summary`、`performance`、`signals`、`inventory` のいずれかを設定します。各パネルの条件付き表示を対応する値に設定します。概要の2つのクエリは、完成版 JSON の `vm-summary` と `alert-summary` の `query` を使用します。

### 2台の最新の Heartbeat

データソースを「ログ」、リソースを今回の LAW、時間範囲を「過去24時間」、可視化を「グリッド」に設定します。Azure リソース ID で絞り込むので、同じ LAW に別の VM がログを送っていても対象に入りません。

```kusto
Heartbeat
| where _ResourceId in~ ("AZURE_VM_RESOURCE_ID", "ARC_VM_RESOURCE_ID")
| summarize arg_max(TimeGenerated, Computer) by ResourceId = tolower(_ResourceId)
| project Computer, ['環境'] = iff(ResourceId =~ "ARC_VM_RESOURCE_ID", "オンプレミス (Arc)", "Azure"),
          ['最終ハートビート'] = TimeGenerated, ['経過分'] = datetime_diff('minute', now(), TimeGenerated)
| order by ['環境'] asc
```

最終ハートビートの時刻が古いときや行が表示されないときは、まずログ収集経路を確認します。Heartbeat が見つからないだけでは VM が停止したとは判断できません。

### CPU 使用率と利用可能なメモリ

ログのクエリ項目を2つ作ります。どちらも同じ LAW と「過去24時間」を選び、可視化を「折れ線グラフ」に設定します。X 軸は `TimeGenerated`、系列は `VM`、Y 軸はクエリの数値列です。

CPU 使用率のクエリです。2台を同じグラフで比較します。

```kusto
Perf
| where _ResourceId in~ ("AZURE_VM_RESOURCE_ID", "ARC_VM_RESOURCE_ID")
| where ObjectName =~ "Processor Information" and CounterName =~ "% Processor Time" and InstanceName =~ "_Total"
| extend VM = strcat(iff(_ResourceId =~ "ARC_VM_RESOURCE_ID", "Arc: ", "Azure: "), Computer)
| summarize CpuPercent = avg(CounterValue) by bin(TimeGenerated, 5m), VM
| order by TimeGenerated asc
```

利用可能なメモリのクエリです。バイトを MiB に換算して表示します。値が下がった時刻をイベントやアラートの時刻と見比べます。搭載メモリの異なる VM 同士では、空き容量の大小だけで逼迫を判断せず、各 VM の推移を見ます。

```kusto
Perf
| where _ResourceId in~ ("AZURE_VM_RESOURCE_ID", "ARC_VM_RESOURCE_ID")
| where ObjectName =~ "Memory" and CounterName =~ "Available Bytes"
| extend VM = strcat(iff(_ResourceId =~ "ARC_VM_RESOURCE_ID", "Arc: ", "Azure: "), Computer)
| summarize AvailableMemoryMiB = avg(CounterValue) / 1048576.0 by bin(TimeGenerated, 5m), VM
| order by TimeGenerated asc
```

どちらも5分ごとの平均です。データがまだない時間帯を `0` として表示しないでください。

### Windows イベント件数の上位10件

データソースを「ログ」、リソースを同じ LAW、時間範囲を「過去24時間」、可視化を「グリッド」にします。VM・ログ・ソース・イベント ID の組み合わせを件数順に表示します。

```kusto
Event
| where _ResourceId in~ ("AZURE_VM_RESOURCE_ID", "ARC_VM_RESOURCE_ID")
| extend VM = strcat(iff(_ResourceId =~ "ARC_VM_RESOURCE_ID", "Arc: ", "Azure: "), Computer)
| summarize ['件数'] = count(), ['最終発生時刻'] = max(TimeGenerated) by VM, EventLog, Source, EventID
| top 10 by ['件数'] desc
```

既定構成では `Application` の `ArcAmaHandson` / `9001` だけが収集対象なので、両 VM がイベントを送っていれば2行が表示されます。イベントログ全体のランキングではありません。この演習のためにイベント収集範囲を広げる必要はありません。

### 関連するログ検索アラート

データソースを「Azure Resource Graph」、対象を演習のサブスクリプション、可視化を「グリッド」に設定します。Step1 の規則名を変更した場合はクエリでも置き換えます。アラート規則のスコープは LAW なので、VM のリソース ID だけで絞り込むと LAW を対象とするアラートを取りこぼします。

```kusto
alertsmanagementresources
| where type =~ "microsoft.alertsmanagement/alerts"
| extend StartedAt = todatetime(properties.essentials.startDateTime),
         Target = tostring(properties.essentials.targetResource),
         Rule = tostring(properties.essentials.alertRule)
| where StartedAt >= ago(24h)
| where Target in~ ("LAW_RESOURCE_ID", "AZURE_VM_RESOURCE_ID", "ARC_VM_RESOURCE_ID")
| where name =~ "alert-arclab-event9001" or Rule endswith "/alert-arclab-event9001"
| project StartedAt, name, Severity = tostring(properties.essentials.severity),
          Condition = tostring(properties.essentials.monitorCondition),
          Dimensions = properties.context.context.condition.allOf[0].dimensions,
          Target, AlertId = id
| order by StartedAt desc
```

`Condition` は検知状態（`Fired` / `Resolved`）です。`Dimensions` に `Computer` が表示されない場合は、「モニター → アラート」で該当する発生レコードの詳細を開いて確認します。この一覧は直近24時間に発生したアラートであり、現在も発生中のアラートだけを示すものではありません。

### Azure VM と Arc マシンの OS バージョンと状態

Azure VM の OS と電源状態は、Azure Resource Manager の `GET` で `AZURE_VM_RESOURCE_ID/instanceView` から取得します。URL パラメーターは `api-version` = `2024-07-01` です。表示する項目は `osName`、`osVersion`、`statuses` 内の `PowerState/` で始まる状態です。

Arc マシンは、データソースを「Azure Resource Graph」、対象を演習のサブスクリプション、可視化を「グリッド」に設定します。この情報は Azure Arc のマシンリソースから取得します。

```kusto
Resources
| where type =~ "microsoft.hybridcompute/machines" and id =~ "ARC_VM_RESOURCE_ID"
| project ['マシン'] = name, OS = tostring(properties.osName),
          ['OS バージョン'] = tostring(properties.osVersion), ['Arc 接続状態'] = tostring(properties.status)
```

OS バージョンはパッチ適用や更改の対象を調べる手掛かりになります。Azure の電源状態と Arc の接続状態は意味が異なります。ゲストから監視データが届いているかは、どちらも「概要」の Heartbeat で確認してください。

### Azure VM と Arc マシンの IP アドレス

Azure VM のプライベート IP は、Azure Resource Graph から対象 VM に接続された NIC の IP 構成を取得します。完成版 JSON の `azure-ip` のクエリを使用します。

Arc マシンは、Arc が報告したネットワークプロファイルから取得します。

1. クエリを追加し、データソースを「Azure Resource Manager」、HTTP メソッドを `GET`、可視化を「グリッド」に設定します。URL パスには先頭が `/subscriptions/` の Arc マシンのリソース ID を使います。

   ```text
   ARC_VM_RESOURCE_ID
   ```

2. URL パラメーターに `api-version` = `2025-01-13` を追加します。
3. 「結果の設定」で形式を「JSON Path」にし、テーブルのパスを `$.properties.networkProfile.networkInterfaces[*].ipAddresses[*]` と指定します。次の列を追加してクエリを実行します。

   | 列名 | JSON Path |
   |---|---|
   | IP アドレス | `$.address` |
   | IP バージョン | `$.ipAddressVersion` |
   | サブネット | `$.subnet.addressPrefix` |

Arc が報告した IP アドレスを、Proxy や Firewall のログで対象ホストを調べる際の手掛かりにできます。複数の NIC や IPv6 がある場合は複数行が表示されます。用途に合うアドレスを確認し、この情報をリアルタイムの到達性の判定には使わないでください。権限エラーの場合は、Arc マシンのネットワークプロファイルを読む権限を確認します。

## 保存して確認する

1. ワークブックを `wb-arclab-hybrid-monitoring` などの名前で演習用 RG に保存し、開き直します。
2. 初期表示が「概要」で、2台の確認状況と関連アラート件数をタブのすぐ下で確認できることを確認します。「性能」で両 VM の Heartbeat と CPU・メモリの各2系列を確認します。
3. 「イベント・アラート」で `9001` の発生時刻とアラートの `Computer` を照合します。直近24時間にイベントがない場合は、Step1・Step2 の既存手順で対象 VM にテストイベントを1件ずつ生成し、LAW への取り込みと規則の評価を待って更新します。「構成」では両 VM の OS と IP、Azure VM の電源状態、Arc マシンの接続状態を確認します。OS と IP は両 VM 内で `Get-CimInstance Win32_OperatingSystem` と `Get-NetIPAddress` の結果とも照合できます。
4. 演習後に性能収集が不要なら、この演習で追加した性能用 DCR の関連付けを外します。ワークブックを閉じたり削除したりしても性能データの収集は止まりません。Step1・Step2 の DCR、AMA、アラート規則はそのまま残します。

## 公式資料

- [Azure Monitor ワークブックを作成する](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-create-workbook)
- [ワークブックのデータソース](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-data-sources)
- [AMA でパフォーマンスカウンターを収集する](https://learn.microsoft.com/azure/azure-monitor/vm/data-collection-performance)
- [Azure Monitor の Resource Graph クエリ例](https://learn.microsoft.com/azure/azure-monitor/fundamentals/resource-graph-samples)
- [Arc マシンの取得 API](https://learn.microsoft.com/rest/api/hybridcompute/machines/get?view=rest-hybridcompute-2025-01-13)
- [ワークブックで JSONPath を使用する](https://learn.microsoft.com/azure/azure-monitor/visualize/workbooks-jsonpath)
