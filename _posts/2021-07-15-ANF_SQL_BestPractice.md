---
title: Microsoft SQL on Azure NetApp Files の性能考慮事項
author: Yi Hu
date: 2021-07-14 00:00:00 +0900
categories: [Azure NetApp Files]
tags: [Azure NetApp Files, SQL]
---
# 1. Microsoft SQL ワークロードで ANF を利用するメリット
Microsoft SQLのライセンス料金は、インフラコストの大半を占めていて、ANF を利用することより、VM コア数を抑えることができ、それによってインフラコストを削減することができます。  
具体的には、ANF は NAS サービスであり、ANF へのアクセスは VM の NIC を介して行われます。そのため、ANF の性能は VM レベルのスループット、IOPS 上限値に制限されていく、VM のネットワーク帯域制限のみが適用されます。  
なお、VM への Inbound 通信は VM のネットワーク帯域に制限されていませんので、ANF への Read 操作は VM レベルの制限がありません。  
Write 操作のスループットは、VM レベルのネットワーク帯域に制限されていますが、その制限は通常、VM レベルのスループットの制限の数倍となり、かつ IOPS に対しての制限がありません。  

上記を踏まえて、ANF を利用したコアの少ない VM でも管理ディスクを使った大きいサイズ VM の同等な SQL 性能を実現することが可能です。 

利用コストの計算例は、公開ドキュメントに公開されていますので、ご参考ください。  

<div style="text-align: left"><img src="/assets/blog/2021-07-15-ANF_SQL_BestPractice1/1.png" ></div>
<br>

[**SQL Server のデプロイに Azure NetApp Files を使用する利点**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/solutions-benefits-azure-netapp-files-sql-server)


# 2. Microsoft SQL on Azure NetApp Files の性能考慮事項
Microsoft SQL ワークロードでANFを利用する場合、より高い性能を実現するには、いくつかの考慮事項があり、詳細は以下にご紹介します。
## 2.1 VMのサイズの選定  
ANF への Write 性能は、VM のネットワーク帯域に依存しますので、以下のドキュメントに公開されている VM のネットワーク帯域を確認していただき、データベースの性能要件に合わせて、VM サイズを選定する必要があります。  
<br />
[**Azure の仮想マシンのサイズ**](https://docs.microsoft.com/ja-jp/azure/virtual-machines/sizes)

## 2.2. 高速ネットワークの設定  
ANF は NAS のため、ストレージへのアクセス全部 VM の NIC を介して行われます。高速ネットワーク機能を有効することでより高スループット、低遅延を実現できますので、この機能を対応する VM サイズを優先的に選定し、VM 構築時に当該機能を有効にすることが推奨されます。  
<br />
また、稀ではありますが、ポータル上で高速ネットワークを有効しても、OS 内部で高速ネットワークのドライバうまくロードされていないことがあります。  
ドライバばロードされているかどうかは OS 内部から確認できます。 
<br />
確認方法はこのドキュメントにて紹介されています。  
<br>
[**Windows VM にイーサネット コントローラーがインストールされていることを確認する**](https://docs.microsoft.com/ja-jp/azure/virtual-network/create-vm-accelerated-networking-powershell#confirm-the-ethernet-controller-is-installed-in-the-windows-vm)
<div style="text-align: left"><img src="/assets/blog/2021-07-15-ANF_SQL_BestPractice1/2.png" ></div>
<br>

## 2.3. SMB マルチチャネルの設定  
SMB マルチチャネル数は、ANF と VM 間に張られている SMB セッションの数になります。
CPU の性能余裕がありましたら、より高い SMB マルチチャネル数を設定することで、高いスループットを実現できます。SMB マルチチャネル数は最大16まで設定できます。  
設定は方法は以下のドキュメントにて紹介されています。  
<br />
[**SMB マルチチャネルのパフォーマンスはどのようなものですか?**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/azure-netapp-files-smb-performance#whats-the-performance-like-for-smb-multichannel)
<div style="text-align: left"><img src="/assets/blog/2021-07-15-ANF_SQL_BestPractice1/3.png" ></div>
<br>

## 2.4. Large MTUの設定
最大転送単位 (MTU) を既定で有効にすることにより、 SQL Server データ ウェアハウス、データベースのバックアップと復元など、大規模なシーケンシャル転送のパフォーマンスが大幅に向上しますので、Large MTU の有効化設定を推奨します。  
<br>
[**SMB ファイル共有ストレージを使用して SQL Server をインストールする**](https://docs.microsoft.com/ja-jp/sql/database-engine/install-windows/install-sql-server-with-smb-fileshare-as-a-storage-option?view=sql-server-ver15)

## 2.5. SQL Serverの並列度設定
MAXDOP は SQL が推奨されていない数値に設定されましたら、多くのスレッドで特定のスレッドの処理を待つような状況が発生する可能性があり、それによって、SQL のパフォーマンスが低下する事象が発生します。
公開ドキュメントにて紹介されている推奨事項に従い、MAXDOP を設定してください。  
<br>
[**max degree of parallelism サーバー構成オプションの構成**](https://docs.microsoft.com/ja-jp/sql/database-engine/configure-windows/configure-the-max-degree-of-parallelism-server-configuration-option?view=sql-server-ver15#recommendations)

## 2.6. ログファイルとデータファイルの配置
ログファイル、データベースファイルに対しての IO 特性があります。
一般的には：  
<br />
    ログファイル　⇒　Read Sequential　がメイン  
    データベース　⇒  Read、Write 両方がある  
<br />
そのため、より高い性能を実現するには、ログファイル、データベースファイルを別々のボリューム上に配置することが推奨されています。

## 2.7. SMB CA  
SMB CA 機能を有効することにより、ANFメンテナンスが実施されてもストレージへのセッションが維持され、接続瞬断によるサービス影響を避けることができます。  
<br />
SMB CA による性能向上がありませんが、可用性の観点で、SQL利用シナリオにおいては、この機能のご利用が強く推奨されています。  
[**SMB CA 申請フォーム**](https://forms.office.com/Pages/ResponsePage.aspx?id=v4j5cvGGr0GRqy180BHbR2Qj2eZL0mZPv1iKUrDGvc9UQUFTUjExUDA5VU5KMUY1RllSVjNEOUVTWCQlQCN0PWcu)） 
[**既存の SMB ボリュームを変換して継続的可用性を使用する**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/convert-smb-continuous-availability?WT.mc_id=Portal-Microsoft_Azure_Health)


**※リンク先などを含む本情報の内容は、作成日時点でのものであり、予告なく変更される場合がございますので、ご了承ください。**

[^ga-filters]: [Google Analytics Core Reporting API: Filters](https://developers.google.com/analytics/devguides/reporting/core/v3/reference#filters)