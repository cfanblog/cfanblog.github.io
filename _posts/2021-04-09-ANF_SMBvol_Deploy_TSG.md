---
title: Azure NetApp Files SMB ボリューム作成失敗時のチェックリスト
author: Yi Hu
date: 2021-04-09 00:00:00 +0900
categories: [Azure NetApp Files]
tags: [Azure NetApp Files, TroubleshootingGuide]
---
# 1. 概要
Azure NetApp Files はベアメタルストレージサービスであり、委任サブネットなど、独自のネットワーク構成を持っています。サービスの仕様を考慮せず、SMB ボリューム の作成が失敗したケースが多いかと思いますので、本記事で作成失敗時のチェックリストをご紹介します。

# 2. Azure NetApp Files SMB ボリューム作成失敗時のチェックリスト
## **2.1 ネットワーク設定まわりのチェックリスト*** 
1. ANF 委任サブネット は **/28** 以上のアドレス空間が割り当てられていますか  
原因：ANF 委任サブネット は少なくとも /28 のアドレス空間が必要（[**ANF委任サブネット**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/azure-netapp-files-delegate-subnet)）

2. ANF 委任サブネット に **NSG、UDR**を設定していますか  
原因：現状 ANF 委任サブネット はNSG、UDRをサポートしていません（[**ネットワーク制限事項**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/azure-netapp-files-network-topologies#constraints)）  
回避策：アクセス元側のリソースのサブネットに対してNSG、UDRを設定することが可能  

3. AD、ANF の Vnet は **Global Peering** で接続していますか  
原因：現状 Global Peering はサポートしていません  （[**ネットワーク制限事項**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/azure-netapp-files-network-topologies#constraints)）  
回避策：ANF と AD が異なるリージョンにある場合、VPN 接続でこの制限を回避

4. AD 側は必要な**ポート**を開放していますか  
原因：SMB ボリューム 作成時に AD のドメイン参加が行われ、AD 側で通信ポートがブロックされているのであれば、ボリュームの作成が失敗 （[**Active Directory 接続の要件**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/create-active-directory-connections#requirements-for-active-directory-connections)）  
回避策：ポート開放


## **2.2 AD接続まわりのチェックリスト**
1. AD 接続で設定したユーザーは **コンピューター アカウント を作成する権限**を持っていますか  
原因：AD 接続で設定したユーザーで SMB サーバー のドメイン参加を実施しますので、そのユーザーは コンピューター アカウント 作成権限を持っていないと作成が失敗  
回避策：ユーザー権限をチェック

2. AD 接続で設定した **OU** は AD 上に存在していますか  
原因：AD 接続で既存の OU を指定する必要があります 

3. AD 接続で設定した OU は、このような形式になっていますか **OU=second level, OU=first level**  
原因：SMB サーバー コンピューター アカウントが作成される組織単位 (OU) の LDAP パスを AD 接続で指定しますので、OU=second level, OU=first level というフォーマットで設定する必要があります。
（[**Active Directory 接続を作成する**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/create-active-directory-connections#create-an-active-directory-connection)）  

4. AADDS を 利用する場合、OU は **OU=AADDC Computers** に指定していますか  
原因：AADDS を利用する場合は、利用可能な OU は AADDC Computers に限定されています。
（[**Active Directory 接続を作成する**](https://docs.microsoft.com/ja-jp/azure/azure-netapp-files/create-active-directory-connections#create-an-active-directory-connection)）  

5. AD 接続で**プライマリー、セカンダリー DNS** が設定されている場合、ボリューム作成時に DNS サーバー両方とも起動していますか  
原因：SMB ボリューム作成時に、設定されている全ての DNS に通信し、ADの情報を取得しますので、DNS サーバー に接続できないとボリューム作成が失敗  
回避策：AD 接続で設定されている全てのDNSサーバーを起動

# 3. 事象原因の切り分け
NFSボリュームの作成においては、AD ドメイン参加が実施されません。そのため、SMBボリューム作成が失敗しましたら、NFS ボリュームを作成してから SMBボリューム を作成することによって、ネットワーク設定問題なのかそれともAD接続問題なのかの切り分けが出来ると考えられます。

<br>
<br>

**※リンク先などを含む本情報の内容は、作成日時点でのものであり、予告なく変更される場合がございますので、ご了承ください。**

[^ga-filters]: [Google Analytics Core Reporting API: Filters](https://developers.google.com/analytics/devguides/reporting/core/v3/reference#filters)