---
title: Azure NetApp Files Account が作成できない対処法
author: Yi Hu
date: 2021-03-23 00:00:00 +0900
categories: [Azure NetApp Files]
tags: [Azure NetApp Files, Troubleshooting]
---

# 1. 事象
サブスクリプション内で初めて Azure NetApp Files のアカウントをデプロイする時に、「Creation of netAppAccounts has been restricted in this region」というエラーが発生することがあります。

![1.png]({{ "/assets/blog/2021-03-23-Cannot_create_ANF_account/1.png" }})

原因としては、リソースを作成する時に、タイミングによってリソースプロバイダーの登録が実施されないことがあるからです。これは想定されている動作のため、公開ドキュメントの紹介の通り、リソースプロバイダー（RP）の追加作業が初回目のデプロイの際に必要作業として紹介されています。

<https://docs.microsoft.com/ja-jpjp/azure/azure-netapp-files/azure-netapp-files-quickstart-set-up-account-create-volumes?tabs=azure-portal#register-for-azure-netapp-files-and-netapp-resource-provider>


# 2. 対処方法

## ・対処法1
コマンドでリソースプロバイダーを登録

PowerShell
```powershell
Register-AzResourceProvider -ProviderNamespace Microsoft.NetApp
```

Azure CLI
```Azure CLI
az provider register --namespace Microsoft.NetApp --wait
```

## ・対処法2
Azure Portal からリソースプロバイダーを登録

「サブスクリプション」⇒「リソースプロバイダ」順番でリソースプロバイダ設定画面を開き、「Microsoft.NetApp」を選択した上、「登録」をクリックします。「登録」をクリックして状態が「Registering」に変わったら、再度 Azure NetApp Files のアカウントの作成ができるかをご確認ください。

![2.png]({{ "/assets/blog/2021-03-23-Cannot_create_ANF_account/2.png" }})


![3.png]({{ "/assets/blog/2021-03-23-Cannot_create_ANF_account/3.png" }})


![4.png]({{ "/assets/blog/2021-03-23-Cannot_create_ANF_account/4.png" }})


**※リンク先などを含む本情報の内容は、作成日時点でのものであり、予告なく変更される場合がございますので、ご了承ください。**

[^ga-filters]: [Google Analytics Core Reporting API: Filters](https://developers.google.com/analytics/devguides/reporting/core/v3/reference#filters)