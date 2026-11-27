# 4. オプション

[README](../README.md)

必要な演習だけを選んで実施できます。演習ごとに、下表のタイミングで実施してください。

| 演習 | 内容 | 実施できるタイミング |
|---|---|---|
| [4.1 コストアラートの設定](04-optional-budget.md) | 監視リソースグループの月次予算を作成し、実績費用と予測費用に応じてメールで通知する | 監視用リソースグループの作成後 |
| [4.2 ワークブックの作成](04-optional-workbook.md) | 完成版 JSON を貼り付け、「概要」で2台の監視状況とアラートを確認する。「構成」では Azure VM と Arc マシンの OS、IP、状態を見比べる | Step1・Step2 の完了後。CPU とメモリを表示するには性能カウンターの収集が必要 |
| [4.3 hosts を用いる場合](04-optional-hosts.md) | DNS サーバーを変更できない場合に、対象 Azure VM の hosts へ公開 FQDN と IPv4 アドレスを一時的に追加する | Step1 の LAW 作成後 |
| [4.4 インベントリ管理と変更追跡](04-optional-inventory.md) | Azure VM と Arc マシンのソフトウェア一覧を確認し、7-Zip の追加・削除を追跡する | Step1・Step2 の完了後 |
| [4.5 Windows イベントの Critical を検知するアラート](04-optional-critical-event-alert.md) | Critical を1件以上検知するログ検索アラートを追加し、既存のアクショングループで通知する | Critical のログ収集と通知の設定後 |
