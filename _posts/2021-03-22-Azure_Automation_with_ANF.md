---
title: Azure Automation による Azure NetApp Files ボリューム伸縮自動化
author: Yi Hu
date: 2021-03-22 00:00:00 +0900
categories: [Azure NetApp Files, Azure Automation]
tags: [Azure NetApp Files, Azure Automation]
---

## 1.概要
Azure NetApp Files は NetApp社業界トップレベルのストレージテクノロジーを活用したベアメタルサービスであり、高性能、高可用性が求められているワークロードに最適なストレージサービスとなります。

Azure NetApp Files の最も特徴的な特性は、ダウンタイムなしでボリュームサイズ伸縮し、ボリュームのスループットを調整できることであり、ワークロードの負荷に合わせて、ボリュームサイズを調整する運用により、ストレージの性能、コストの最適化を同時に図ることができます。

WVDのシナリオを例としてあげるのならば、以下のような運用が考えられます。

1. 朝のサインインストームによる性能劣化を防ぐため、8時にボリューム、容量プールサイズを増やす
2. サインインピークが過ぎた後、10時にボリューム、容量プールサイズを減らす
3. 午後のサインアウトストームによる性能劣化を防ぐため、17時にボリューム、容量プールサイズを増やす
4. サインアウトピークが過ぎた後、19時にボリューム、容量プールサイズを減らす

現時点、Azure NetApp Files ではボリューム、容量プールの自動サイズ変更機能が提供していません。一方で、サイズ変更用の PowerShell のコマンド提供されていますので、Azure Automation を利用すれば、スケジューリングを設定し簡単にサイズの自動化を実現できます。
実装方法は以下にご紹介します。

<br>

## 2.Automation アカウント 作成
1. Azure Portal で Automation アカウント を検索し、「Automation アカウント」を選択
![1.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/1.png" }})


2. 「+作成」をクリックし、右側に表示されている設定画面で、アカウントの情報を記入し、アカウント作成します
![2.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/2.png" }})

<br>

## 3.Az.Accounts、 Az.NetAppFiles モジュールを追加
Automation が Azure NetApp Files のコマンドを実行させるため、必要な PowerShell モジュールをAz.Accounts、 Az.NetAppFilesインストールする必要がります

1. Automation アカウントを開き、「モジュール」⇒「ギャラリーを参照」をクリックし、以下の順番でモジュールを追加
- Az.Accounts 
- Az.NetAppFiles
![3.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/3.png" }})
![4.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/4.png" }})
![5.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/5.png" }})

<br>

## 4.Runbook 作成
1. 「Runbook」⇒「＋Runbookの作成」をクリックし、Runbookの種類を「PowerShell」に指定した上、Runbook を作成します
![6.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/6.png" }})

2. 作成したRunbook画面で「編集」をクリックし、編集画面を表示させます。
![7.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/7.png" }})

3. Runbookに定期的に実行したいPowerShellスクリプトを追加し、「保存」をクリックします。

ボリュームサイズを増やすRunbook例：
```powershell
#サブスクリプションにログイン
$connection = Get-AutomationConnection -Name AzureRunAsConnection

while(!($connectionResult) -and ($logonAttempt -le 10))
{
    $LogonAttempt++
    # Logging in to Azure...
    $connectionResult = Connect-AzAccount `
                            -ServicePrincipal `
                            -Tenant $connection.TenantID `
                            -ApplicationId $connection.ApplicationID `
                            -CertificateThumbprint $connection.CertificateThumbprint

    Start-Sleep -Seconds 30
}

#容量プールのサイズを指定する
Update-AzNetAppFilesPool -ResourceGroupName "リソースグループ名" -l "リージョン" -AccountName "ANFアカウント名" -PoolName "容量プール名" -PoolSize "ボリュームサイズ(バイト)" -QosType "Auto"

#ボリュームのサイズを指定する
Update-AzNetAppFilesVolume -ResourceGroupName "リソースグループ名" -l "リージョン" -AccountName "ANFアカウント名" -PoolName "容量プール名" -Name "ボリューム名" -UsageThreshold "ボリュームサイズ(バイト)"
```

ボリュームサイズを減らすRunbook例：
```powershell
#サブスクリプションにログイン
$connection = Get-AutomationConnection -Name AzureRunAsConnection

while(!($connectionResult) -and ($logonAttempt -le 10))
{
    $LogonAttempt++
    # Logging in to Azure...
    $connectionResult = Connect-AzAccount `
                            -ServicePrincipal `
                            -Tenant $connection.TenantID `
                            -ApplicationId $connection.ApplicationID `
                            -CertificateThumbprint $connection.CertificateThumbprint

    Start-Sleep -Seconds 30
}

#ボリュームのサイズを指定する
Update-AzNetAppFilesVolume -ResourceGroupName "リソースグループ名" -l "リージョン" -AccountName "ANFアカウント名" -PoolName "容量プール名" -Name "ボリューム名" -UsageThreshold "ボリュームサイズ(バイト)"

#容量プールのサイズを指定する
Update-AzNetAppFilesPool -ResourceGroupName "リソースグループ名" -l "リージョン" -AccountName "ANFアカウント名" -PoolName "容量プール名" -PoolSize "ボリュームサイズ(バイト)" -QosType "Auto"
```

**※2021年3月時点、PowerShellで指定するボリューム、容量プールサイズの単位は byte になりますので、設定値を間違えないようにご注意ください。**

https://docs.microsoft.com/en-us/powershell/module/az.netappfiles/update-aznetappfilespool?view=azps-5.6.0
https://docs.microsoft.com/en-us/powershell/module/az.netappfiles/update-aznetappfilesvolume?view=azps-5.6.0

ボリューム、容量プールのサイズは以下のコマンドで確認できます。

Get-AzNetAppFilesPool
Get-AzNetAppFilesVolume

```console
PS C:\> Get-AzNetAppFilesPool -ResourceGroupName "hyi.rg" -AccountName "anfjpe" -PoolName "pool1"
ResourceGroupName       : hyi.rg
Location                : japaneast
Id                      : /subscriptions/xxx/resourceGroups/xxx/providers/Microsoft.NetApp/netAppAccounts/anfjpe/capacityPools/pool1
Name                    : anfjpe/pool1
Type                    : Microsoft.NetApp/netAppAccounts/capacityPools
Tags                    : 
PoolId                  : xxx
Size                    : 4398046511104
ServiceLevel            : Standard
ProvisioningState       : Succeeded
TotalThroughputMibps    : 65.536
UtilizedThroughputMibps : 0
QosType                 : Auto
```

4.「テスト ウィンドウ」でRunbookをテストし、問題がなければ「公開」をクリックします。
![8.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/8.png" }})

<br>

## 5.Runbook のスケジュール設定
1. Runbookのポータルで「スケジュール」⇒「＋スケジュールの追加」をクリックし、スケジュールを指定し、「OK」をクリックします。
※例のようにパラメータはRunbook内で記載している場合は、「パラメータと実行設定」を設定しなくても問題がありません。
![9.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/9.png" }})

<br>

## 6.Runbook の実行状況を確認
1. Runbookのポータルで「ジョブ」をクリックし、右側に表示されているジョブ詳細画面でRunbookの自動実行状況を確認できます。
![10.png]({{ "/assets/blog/2021-03-22-Azure_Automation_with_ANF/10.png" }})


**※リンク先などを含む本情報の内容は、作成日時点でのものであり、予告なく変更される場合がございますので、ご了承ください。**

[^ga-filters]: [Google Analytics Core Reporting API: Filters](https://developers.google.com/analytics/devguides/reporting/core/v3/reference#filters)