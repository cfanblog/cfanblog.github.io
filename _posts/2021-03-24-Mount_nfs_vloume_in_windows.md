---
title: Windows Severで ANF NFS ボリュームを利用する方法
author: Yi Hu
date: 2021-03-24 00:00:00 +0900
categories: [Azure NetApp Files]
tags: [Azure NetApp Files, Troubleshooting]
---

# 1. 概要
クライアントアプリケーション、ワークロードの要件によって Windows Sever から ANF NFS ボリュームをアクセスする場合があるかと思います。接続方法はご紹介します。

# 2. 準備作業
## 2.1 ボリュームの操作権限を変更 
NFSボリュームを Linux VM にマウントし、chmod 777 または chmod 775 でボリューム操作権限を変更します。
変更しておかないと、マウントすることができますが、Windows マシンからボリュームにアクセスできません。

```console
[root@redhat82 anfadmin]# chmod 777 nfsvol1
[root@redhat82 anfadmin]# ll
total 4
drwxrwxrwx. 2 root root 4096 Mar 24 09:41 nfsvol1
```

## 2.2 NFSクライアントをインストール
Server Manager から NFS クライアントを追加します。

<div style="text-align: left"><img src="/assets/blog/2021-03-24-Mount_nfs_vloume_in_windows/1.png" ></div>
<br>


# 3. NFS ボリュームマウント
CMDを起動し、mountコマンドでボリュームをマウントします。

```console
C:\Users\huyi.dc2>mount 170.3.0.5:/nfsvol1 m:
m: is now successfully connected to 170.3.0.5:/nfsvol1

The command completed successfully.
```

File Explorer でマウント状態を確認します。

<div style="text-align: left"><img src="/assets/blog/2021-03-24-Mount_nfs_vloume_in_windows/2.png" ></div>
<br>

Windows から ボリューム内のデータを確認します。

<div style="text-align: left"><img src="/assets/blog/2021-03-24-Mount_nfs_vloume_in_windows/3.png" ></div>
<br>


Linux から ボリューム内のデータを確認します。

```console
[root@redhat82 nfsvol1]# ll
total 0
-rwxr-xr-x. 1 4294967294 4294967294 0 Mar 24 10:02 testfile.txt
[root@redhat82 nfsvol1]#
```

**※リンク先などを含む本情報の内容は、作成日時点でのものであり、予告なく変更される場合がございますので、ご了承ください。**

[^ga-filters]: [Google Analytics Core Reporting API: Filters](https://developers.google.com/analytics/devguides/reporting/core/v3/reference#filters)