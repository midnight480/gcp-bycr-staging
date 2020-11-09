# gcp-bycr-staging

## 環境イメージ

### 作成ツール

* [Visual Studio Code](https://azure.microsoft.com/ja-jp/products/visual-studio-code/)
* [Draw.io Integration](https://marketplace.visualstudio.com/items?itemName=hediet.vscode-drawio)

### ファイル

* [初期段階の構想](./images/gcp.drawio) [エクスポート画像](./images/gcp.png)
* [On-Prem用だとGKE利用前提になるので倣って構想](./images/gcp_kube.drawio) [エクスポート画像](./images/gcp_kube.png)
* [GCLB - GCEのほうが運営的にもGKE利用するほど大規模にならないのでは構想](./images/gcp_gclb.drawio) [エクスポート画像](./images/gcp_gclb.png)

## 実装目的

* BeyondCorp RemoteAccessを利用（CloudIAP）
* SSL証明書はGoogleマネージド証明書を利用（中身は、Let's Encryptの代理発行と3ヶ月毎の自動更新をGoogleが実施）
* GCPとOn-Prem間は、CloudVPNで接続
* On-Premのリソースは、HTTPで受ける（GCP上にSSLアクセラレータが必要）

## 想定リソース

### GAEでリバースプロキシ

[GAE](./src/gae.md)

### GCEでリバースプロキシ

[GCE](./src/gce.md)

## Stackdriver logging

```
curl -sSO https://dl.google.com/cloudagents/install-logging-agent.sh
sudo bash install-logging-agent.sh
```

## create custom image

```
$ gcloud compute images create byc-centos7-custom \
>   --source-disk byc-proxy-1 \
>   --source-disk-zone asia-northeast1-b
Created [https://www.googleapis.com/compute/v1/projects/bycr-20201107/global/images/byc-centos7-custom].
NAME                PROJECT        FAMILY  DEPRECATED  STATUS
byc-centos7-custom  bycr-20201107                      READY
```

## ハマったこと1: GoogleDomainで取得済みのドメインが既に別のプロジェクトに紐付いている

* カスタムドメインが既に別のプロジェクトと紐付いているとき  
* そのプロジェクトが既にシャットダウン（削除済み）で、 `gloucd projects list`　でも表示されないとき  
* GCPコンソールから該当するプロジェクトの選択はできないので、gcloudコマンドで強制的に剥奪する  

`gcloud --project <project_name> app domain-mappings delete <your_domain>`

```
gcloud app domain-mappings create '${domain}'
ERROR: (gcloud.app.domain-mappings.create) Apps instance [${ProjectA}] is the subject of a conflict: Domain '${domain}' is already mapped to another application. You must delete the existin
g domain mapping before you can re-map the domain, or you may specify 'DomainOverrideStrategy.OVERRIDE' on the request to force overwrite the existing mapping. Domain '${domain}' is currently
mapped to application '${ProjectB}'.
```

```
gcloud --project '${ProjectB}' app domain-mappings delete '${domain}'
Deleting mapping [${domain}]. This will stop your app from serving
from this domain. (Y/n)?  Y

Waiting for operation [apps/${ProjectB}/operations/17c6dda4-c9e2-4709-bef0-397369068229] to complete...done.
Deleted [${domain}].
```

```
$ gcloud app domain-mappings create '${domain}'
Waiting for operation [apps/${ProjectA}/operations/1874f9f9-d47c-40e1-bf66-e212d4bef9f6] to complete...done.
Created [${domain}].
Please add the following entries to your domain registrar. DNS changes can require up to 24 hours to take effect.
id: ${domain}
resourceRecords:
- rrdata: ..32.
  type: A
...
- rrdata: xx:32::15
  type: AAAA
...
$
```

## ハマったこと2: GCPのVPCにおけるServerlessVPCコネクタが正常に生成されない

VPCのプライベートIPアドレスは正常にリソースにアタッチされているが、赤い！がついて状態から変わらず。。

## ハマったこと3: CentOS7のSELinuxによってLet's Encryptのフォルダの読み込みエラー

```
sudo cp /etc/letsencrypt/live/${domain}/privkey.pem /etc/pki/tls/private/privkey.pem
sudo cp /etc/letsencrypt/live/${domain}/fullchain.pem /etc/pki/tls/certs/fullchain.pem
```

# 参考にしたサイト

## Google公式Document

* [IAP による Compute Engine アプリとリソースの保護](https://cloud.google.com/context-aware-access/docs/securing-compute-engine?hl=ja)
* [単一の VM に Cloud Logging エージェントをインストールする](https://cloud.google.com/logging/docs/agent/installation?hl=ja)
* [デフォルトの Logging エージェントのログ](https://cloud.google.com/logging/docs/agent/default-logs?hl=ja)
* [gcloud compute disks list](https://cloud.google.com/sdk/gcloud/reference/compute/disks/list)
* [カスタム イメージの作成、削除、利用非推奨](https://cloud.google.com/compute/docs/images/create-delete-deprecate-private-images#gcloud)
* [ヘルスチェックと自動修復の設定](https://cloud.google.com/compute/docs/instance-groups/autohealing-instances-in-migs?hl=ja)

```
ソース IP 範囲 130.211.0.0/22, 35.191.0.0/16
許可対象プロトコル / ポート tcp:443
```

## GAE - ReverseProxy

* [GAE Standard環境のNode.jsでリバースプロキシ構築](https://www.kwonline.org/memo2/2019/02/22/reverse-proxy-on-gae-standard-nodejs/)
* [Golangでリバースプロキシ](https://sites.google.com/site/progrhymetechwiki/home/memo/2017/20170425)
* [http-revese-proxy-java](https://github.com/h-r-k-matsumoto/http-revese-proxy-java)

## GCE - ReverseProxy

* [Google Compute Engineのロードバランサを読み解く](https://dev.classmethod.jp/articles/gce-lb2/)
* [Google Cloud Platform nginx 踏み台サーバー 〇分クッキング](https://qiita.com/AkiQ/items/5392658898be66fbf77b)
* [Google Cloud Identity-Aware Proxy(Cloud IAP)でWebアプリに認証を追加する](https://takipone.com/gcp-cloud-iap-ataglance/)
* [nginxでリバースプロキシ設定[GCE]](https://sbc-web.hatenablog.jp/entry/2017/05/02/nginx%E3%81%A7%E3%83%AA%E3%83%90%E3%83%BC%E3%82%B9%E3%83%97%E3%83%AD%E3%82%AD%E3%82%B7%E8%A8%AD%E5%AE%9A%5BGCE%5D)
* [BeyondCorp Remote Accessを支える技術 #1 GCP Cloud IAP connectorを試してみた](https://dev.classmethod.jp/articles/beyondcorp-remote-access-getting-started1/)
* [簡単！Google Cloud Platform で HTTP/2対応サイトを作る](https://blog.apar.jp/web/4908/)
* [Stackdriver Logging + BigQuery で GCE の nginx ログを取り込む](https://runble1.com/stackdriver-logging-bigquery-nginx/#GCE_Stackdriver_logging_agent)

## nginx - ReverseProxy

* [CentOS7 の Nginx で Let's Encrypt を使う](https://qiita.com/ekzemplaro/items/15bceed7c5612fe039d5)
* [nginx で SSL解きリバースプロキシな設定のお作法](https://qiita.com/ywatai@github/items/a179186a458a42b3c7f0)
* [Let's Encrypt](https://qiita.com/mid480/private/beddc662c95bff82d6a7)
* [SELinuxを無効にするには](http://park1.wakwak.com/~ima/centos4_selinux0001.html)
